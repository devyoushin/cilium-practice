# BPF 로드밸런싱

Cilium은 iptables/IPVS를 완전히 대체하는 **eBPF 기반 로드밸런서**를 제공합니다.
kube-proxy가 하던 일을 eBPF Map으로 처리하여 성능이 크게 향상됩니다.

---

## kube-proxy vs Cilium BPF 로드밸런싱

```
kube-proxy 방식 (iptables):
  요청 → iptables 체인 순차 검색 (O(n)) → DNAT → 파드
          (서비스 1000개 = 규칙 수만 개, 선형 탐색)

Cilium BPF 방식:
  요청 → eBPF Map 해시 조회 (O(1)) → 파드
          (서비스 수에 관계없이 동일한 성능)
```

| 항목 | kube-proxy (iptables) | Cilium BPF |
|---|---|---|
| 규칙 탐색 | O(n) 선형 탐색 | O(1) 해시 맵 |
| 서비스 업데이트 | 전체 iptables 재작성 | Map 항목 업데이트만 |
| DSR (Direct Server Return) | 미지원 | 지원 |
| 소스 IP 보존 | 제한적 | 완전 지원 |
| 세션 어피니티 | 기본 지원 | 지원 + Maglev 해싱 |

---

## kube-proxy 대체 활성화

```yaml
# helm/values.yaml
kubeProxyReplacement: true
k8sServiceHost: "<EKS_API_SERVER_ENDPOINT>"
k8sServicePort: "443"
```

```bash
# kube-proxy replacement 상태 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium status | grep KubeProxyReplacement

# 서비스 목록 (BPF 레이어 기준)
kubectl -n kube-system exec ds/cilium -- \
  cilium service list
```

---

## 로드밸런싱 알고리즘

### 1. 랜덤 (기본값)

```yaml
# helm/values.yaml
loadBalancer:
  algorithm: random
```

### 2. Maglev 해싱 (세션 어피니티)

Google Maglev 알고리즘을 사용하여 동일한 클라이언트 요청이 동일한 백엔드로 라우팅됩니다.

```yaml
# helm/values.yaml
loadBalancer:
  algorithm: maglev
```

```
Maglev 동작:
  클라이언트 IP → 해시 → 일관된 백엔드 선택
                           (백엔드가 추가/제거되어도 재해싱 최소화)
```

### 세션 어피니티 (Service 레벨)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800   # 3시간
  ports:
    - port: 80
      targetPort: 8080
```

---

## DSR (Direct Server Return)

클라이언트로의 응답 트래픽이 로드밸런서를 거치지 않고 파드에서 클라이언트로 직접 전송됩니다.

```
일반 방식:
  클라이언트 → LB → 파드 → LB → 클라이언트
                              (응답도 LB 통과, 병목 가능)

DSR 방식:
  클라이언트 → LB → 파드 → 클라이언트
                   (응답은 파드에서 직접, LB 우회)
```

```yaml
# helm/values.yaml
loadBalancer:
  mode: dsr           # DSR 활성화
  algorithm: maglev   # DSR과 함께 Maglev 권장
```

> **EKS 제약**: DSR은 노드 간 라우팅이 필요합니다.
> ENI 모드에서는 AWS 라우팅 설정이 추가로 필요할 수 있습니다.

---

## NodePort 및 ExternalTrafficPolicy

### ExternalTrafficPolicy: Local

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  externalTrafficPolicy: Local    # 클라이언트 소스 IP 보존
  selector:
    app: my-app
  ports:
    - port: 80
      nodePort: 30080
```

```
ExternalTrafficPolicy: Cluster (기본):
  외부 → NodePort → SNAT → 다른 노드의 파드 가능
                    (소스 IP 손실, 불필요한 홉 발생)

ExternalTrafficPolicy: Local:
  외부 → NodePort → 같은 노드의 파드만 라우팅
                    (소스 IP 보존, 노드에 파드 없으면 DROP)
```

---

## BPF 로드밸런서 디버깅

```bash
# 서비스별 백엔드 목록 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium service list

# 특정 서비스 상세 (서비스 ID 필요)
kubectl -n kube-system exec ds/cilium -- \
  cilium service get <SERVICE_ID>

# BPF Map에서 서비스 엔트리 직접 조회
kubectl -n kube-system exec ds/cilium -- \
  cilium bpf lb list

# 서비스별 트래픽 통계
kubectl -n kube-system exec ds/cilium -- \
  cilium bpf lb list --revnat

# 연결 추적 (CTMap) 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium bpf ct list global | head -20
```

---

## LoadBalancer 타입 서비스 (AWS NLB 연동)

Cilium과 AWS Load Balancer Controller를 함께 사용하여 NLB를 프로비저닝합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

---

## 성능 최적화 — XDP 가속

XDP(eXpress Data Path)를 사용하면 NIC 드라이버 레벨에서 패킷을 처리하여
커널 네트워크 스택 진입 전에 처리합니다.

```yaml
# helm/values.yaml
loadBalancer:
  acceleration: native    # XDP 네이티브 모드 (NIC 드라이버 지원 필요)
  # 또는
  acceleration: best-effort  # 가능하면 XDP, 아니면 generic
```

```bash
# XDP 가속 상태 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium status | grep "XDP Acceleration"
```

---

## 참고 링크

- [Cilium kube-proxy 대체](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [Maglev 해싱](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#maglev-consistent-hashing)
- [DSR 문서](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#direct-server-return-dsr)
- [XDP 가속](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/#loadbalancer-xdp-acceleration)
