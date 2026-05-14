# Cilium Service Mesh

Cilium은 **사이드카 프록시 없이** eBPF만으로 서비스 메시 기능을 제공합니다.
Istio와 달리 각 파드에 Envoy를 주입하지 않아 오버헤드가 적습니다.

---

## Cilium Service Mesh vs Istio 비교

| 항목 | Cilium Service Mesh | Istio (Sidecar) |
|---|---|---|
| 프록시 방식 | 사이드카 없음 (eBPF + 노드별 Envoy) | 파드마다 Envoy 사이드카 |
| mTLS | eBPF 또는 노드 레벨 Envoy | 사이드카 Envoy |
| L7 정책 | HTTP/gRPC 지원 | HTTP/gRPC/TCP 풍부한 지원 |
| 트래픽 관리 | 기본 수준 (가중치, 미러링) | 고급 (Circuit Breaker, Fault Injection) |
| 관찰성 | Hubble 통합 | Kiali + Jaeger 별도 설치 |
| 리소스 오버헤드 | 낮음 | 파드당 사이드카 추가 |
| 성숙도 | 빠르게 성장 중 | 매우 성숙 |

---

## 활성화

```yaml
# helm/values.yaml
ingressController:
  enabled: true
  loadbalancerMode: shared    # 또는 dedicated

envoy:
  enabled: true               # 노드 레벨 Envoy DaemonSet

l7Proxy: true                 # L7 정책 활성화
```

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --values my-values.yaml \
  --reuse-values
```

---

## mTLS (상호 TLS)

Cilium은 eBPF 레벨에서 파드 간 mTLS를 제공합니다.

### WireGuard 투명 암호화

파드 간 모든 트래픽을 WireGuard로 자동 암호화합니다.

```yaml
# helm/values.yaml
encryption:
  enabled: true
  type: wireguard
```

```bash
# WireGuard 암호화 상태 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium status | grep Encryption

# WireGuard 터널 인터페이스 확인
kubectl -n kube-system exec ds/cilium -- \
  wg show
```

### IPsec 암호화 (대안)

```yaml
# helm/values.yaml
encryption:
  enabled: true
  type: ipsec

# IPsec 키 시크릿 생성 필요
```

```bash
# IPsec 키 생성 및 시크릿 등록
kubectl create -n kube-system secret generic cilium-ipsec-keys \
  --from-literal=keys="3 rfc4106(gcm(aes)) $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null | xxd -p -c 64)) 128"
```

---

## L7 트래픽 관리

Cilium Ingress Controller를 통해 L7 라우팅을 구성합니다.

### Cilium Ingress 기본 설정

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  annotations:
    ingress.cilium.io/loadbalancer-mode: shared
spec:
  ingressClassName: cilium
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

### 트래픽 가중치 분배 (CiliumEnvoyConfig)

```yaml
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig
metadata:
  name: canary-split
  namespace: default
spec:
  services:
    - name: my-service
      namespace: default
      listener: ""
  backendServices:
    - name: my-service-v1
      namespace: default
    - name: my-service-v2
      namespace: default
  resources:
    - "@type": type.googleapis.com/envoy.config.listener.v3.Listener
      name: my-service/80
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                route_config:
                  virtual_hosts:
                    - name: my-service
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            weighted_clusters:
                              clusters:
                                - name: default/my-service-v1
                                  weight: 90
                                - name: default/my-service-v2
                                  weight: 10
```

---

## L7 네트워크 정책

### HTTP 메서드/경로 기반 정책

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-policy
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "/api/v1/users.*"
              - method: POST
                path: "/api/v1/users"
              - method: PUT
                path: "/api/v1/users/[0-9]+"
```

### gRPC 정책

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: grpc-policy
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: grpc-server
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: grpc-client
      toPorts:
        - ports:
            - port: "50051"
              protocol: TCP
          rules:
            http:
              - method: POST
                path: "/mypackage.MyService/.*"
```

---

## 관찰성 — L7 흐름

Hubble에서 L7 트래픽을 실시간으로 확인합니다.

```bash
# HTTP 요청/응답 흐름
hubble observe --protocol http --follow \
  --output json | jq '{
    from: .flow.source.pod_name,
    to: .flow.destination.pod_name,
    method: .flow.l7.http.method,
    url: .flow.l7.http.url,
    status: .flow.l7.http.code
  }'

# gRPC 흐름
hubble observe --protocol grpc --follow
```

---

## Ingress Controller 상태 확인

```bash
# Cilium Ingress 확인
kubectl get ingress -A
kubectl describe ingress my-ingress -n default

# Envoy 파드 확인
kubectl get pods -n kube-system -l app.kubernetes.io/part-of=cilium,app=cilium-envoy

# Envoy 설정 덤프 (고급)
kubectl -n kube-system exec ds/cilium-envoy -- \
  curl -s http://localhost:9901/config_dump | jq '.configs[] | .dynamic_route_configs'
```

---

## 참고 링크

- [Cilium Service Mesh 문서](https://docs.cilium.io/en/stable/network/servicemesh/)
- [Cilium Ingress Controller](https://docs.cilium.io/en/stable/network/servicemesh/ingress/)
- [WireGuard 투명 암호화](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/)
- [L7 정책](https://docs.cilium.io/en/stable/security/http/)
