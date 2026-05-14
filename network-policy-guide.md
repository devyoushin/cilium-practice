# 네트워크 정책 (Network Policy)

Cilium은 두 가지 종류의 네트워크 정책을 지원합니다:
- **K8s NetworkPolicy**: 표준 Kubernetes 정책 (L3/L4만 지원)
- **CiliumNetworkPolicy**: Cilium 전용 확장 정책 (L7, DNS, 클러스터 전체 정책)

---

## 기본 개념

### 기본 동작 (Default Behavior)

정책이 없는 상태에서는 **모든 트래픽이 허용**됩니다.

```
정책 없음    → 모든 통신 허용 (기본값)
정책 1개 적용 → 해당 파드에 대한 명시적 허용만 통과, 나머지 차단
```

> 정책이 특정 파드에 적용되는 순간, 그 파드는 **명시적으로 허용된 트래픽만** 수신합니다.

---

## K8s NetworkPolicy (L3/L4)

### 인그레스(Ingress) 정책 — 수신 제한

```yaml
# examples/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend          # 이 정책이 적용될 파드
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend   # frontend 파드에서만 수신 허용
      ports:
        - protocol: TCP
          port: 8080
```

```bash
kubectl apply -f examples/network-policy.yaml

# 정책 적용 확인
kubectl get networkpolicy -n default
kubectl describe networkpolicy allow-from-frontend -n default
```

### 이그레스(Egress) 정책 — 송신 제한

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database     # database 파드로만 송신 허용
      ports:
        - protocol: TCP
          port: 5432
    - ports:
        - protocol: UDP
          port: 53              # DNS 조회 허용 (필수!)
```

> **DNS 허용 주의**: Egress 정책 적용 시 UDP 53 포트를 허용하지 않으면
> DNS 조회가 차단되어 서비스 이름으로 통신이 불가합니다.

---

## CiliumNetworkPolicy (L7 포함)

### HTTP 경로 기반 정책 (L7)

```yaml
# examples/cilium-network-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-http-get-only
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: api-server
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: GET           # GET 요청만 허용
                path: "/api/v1/.*"    # 특정 경로만 허용
```

### DNS 기반 이그레스 정책

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-external-api
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
    - toFQDNs:                        # 도메인명으로 외부 허용
        - matchName: "api.example.com"
        - matchPattern: "*.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    - toEndpoints:
        - matchLabels:
            "k8s:io.kubernetes.pod.namespace": kube-system
            "k8s:k8s-app": kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"     # 모든 DNS 쿼리 허용
```

### 클러스터 전체 정책 (CiliumClusterwideNetworkPolicy)

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-health-checks
spec:
  endpointSelector: {}      # 모든 파드에 적용
  ingress:
    - fromEntities:
        - health             # Cilium health check 허용
```

---

## Zero Trust 패턴

기본적으로 모든 트래픽을 차단하고, 필요한 것만 허용하는 패턴입니다.

### 1단계: 네임스페이스 격리

```yaml
# 네임스페이스 간 기본 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}           # 네임스페이스의 모든 파드
  policyTypes:
    - Ingress
    - Egress
```

### 2단계: 필요한 통신만 개방

```yaml
# production 네임스페이스 내부 통신 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}   # 같은 네임스페이스의 모든 파드

---
# ingress-nginx에서 수신 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      expose: "true"
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
```

---

## 정책 검증 및 디버깅

```bash
# 현재 적용된 정책 목록
kubectl get cnp,ccnp -A           # CiliumNetworkPolicy
kubectl get networkpolicy -A      # K8s NetworkPolicy

# 특정 파드의 정책 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium endpoint list

# 특정 엔드포인트의 정책 상세 확인 (엔드포인트 ID 필요)
kubectl -n kube-system exec ds/cilium -- \
  cilium endpoint get <ENDPOINT_ID>

# 네트워크 정책 위반 흐름 확인 (Hubble)
hubble observe --verdict DROPPED --follow

# 특정 파드의 트래픽 흐름
hubble observe --pod default/backend --follow
```

### 정책 시뮬레이션 (적용 전 검증)

```bash
# 두 파드 간 통신이 허용되는지 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium policy trace \
  --src-k8s-pod default:frontend \
  --dst-k8s-pod default:backend \
  --dport 8080/TCP
```

---

## 정책 우선순위

```
K8s NetworkPolicy + CiliumNetworkPolicy 동시 적용 시:
  → 두 정책 모두에서 허용된 트래픽만 통과 (AND 조건)

CiliumClusterwideNetworkPolicy + CiliumNetworkPolicy:
  → 두 정책 중 하나라도 허용하면 통과 (OR 조건)
  → 단, Deny 정책은 Allow보다 우선
```

---

## 레이블 규칙 참고

Cilium은 Kubernetes 레이블을 `k8s:` 접두사로 구분합니다.

```yaml
# Cilium 정책에서 네임스페이스를 지정할 때
fromEndpoints:
  - matchLabels:
      k8s:io.kubernetes.pod.namespace: monitoring

# 또는 namespaceSelector 사용
fromRequires:
  - matchLabels:
      "k8s:io.kubernetes.pod.namespace": production
```

---

## 참고 링크

- [Cilium 정책 언어](https://docs.cilium.io/en/stable/security/policy/language/)
- [K8s NetworkPolicy와 차이점](https://docs.cilium.io/en/stable/security/policy/kubernetes/)
- [DNS 기반 정책](https://docs.cilium.io/en/stable/security/dns/)
- [HTTP L7 정책](https://docs.cilium.io/en/stable/security/http/)
