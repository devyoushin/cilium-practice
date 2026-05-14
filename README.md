# Cilium on EKS — 실습 저장소

EKS 환경에서 Cilium을 처음부터 운영 수준까지 학습하는 실습 저장소입니다.

---

## 환경 정보

| 항목 | 값 |
|---|---|
| 플랫폼 | AWS EKS |
| Cilium Helm Chart | `cilium/cilium` (권장 버전: 1.16.x) |
| 네임스페이스 | `kube-system` |
| CNI 모드 | ENI 모드 (AWS VPC 네이티브) |
| kube-proxy 대체 | BPF kube-proxy replacement 사용 |
| 리전 | `ap-northeast-2` |

---

## 사전 요구사항

```bash
# 필요 도구 확인
kubectl version --client      # >= 1.25
helm version                  # >= 3.10
aws --version                 # AWS CLI v2
cilium version                # Cilium CLI (선택)

# EKS 클러스터 접속 확인
kubectl get nodes
```

---

## 빠른 시작 (Quick Start)

```bash
# 1. aws-node (VPC CNI) DaemonSet 비활성화
kubectl -n kube-system set image daemonset/aws-node \
  aws-node=public.ecr.aws/amazonlinux/amazonlinux:2023

# 2. Helm 저장소 추가
helm repo add cilium https://helm.cilium.io/
helm repo update

# 3. Cilium 설치 (ENI 모드)
helm install cilium cilium/cilium \
  --version 1.16.0 \
  --namespace kube-system \
  --values helm/values.yaml

# 4. 상태 확인
cilium status --wait
# 또는
kubectl get pods -n kube-system -l k8s-app=cilium
```

---

## 학습 경로

```
1. 설치        → install.md
2. 핵심 개념   → architecture-guide.md
3. 심화
   ├── 보안        → network-policy-guide.md
   ├── 가시성      → hubble-guide.md
   ├── 서비스 메시 → service-mesh-guide.md
   ├── 로드밸런싱  → load-balancing-guide.md
   ├── Egress      → egress-gateway-guide.md
   └── BGP         → bgp-guide.md
4. 문제 해결   → troubleshooting-guide.md
5. 실습        → e2e-practice.md
```

---

## 문서 목록

### 설치
| 파일 | 설명 |
|---|---|
| [install.md](./install.md) | EKS에 Cilium 설치 (Helm, ENI 모드, kube-proxy 대체) |

### 핵심 개념
| 파일 | 설명 |
|---|---|
| [architecture-guide.md](./architecture-guide.md) | Cilium 아키텍처 — eBPF, Agent, Operator, Hubble |

### 심화
| 파일 | 설명 |
|---|---|
| [network-policy-guide.md](./network-policy-guide.md) | 네트워크 정책 — K8s NetworkPolicy, CiliumNetworkPolicy, Zero Trust |
| [hubble-guide.md](./hubble-guide.md) | Hubble — 실시간 네트워크 흐름 관찰, UI, Prometheus 메트릭 |
| [service-mesh-guide.md](./service-mesh-guide.md) | Cilium Service Mesh — 사이드카 없는 mTLS, L7 정책, 트래픽 관리 |
| [load-balancing-guide.md](./load-balancing-guide.md) | BPF 로드밸런싱 — kube-proxy 대체, DSR, Maglev 해싱 |
| [egress-gateway-guide.md](./egress-gateway-guide.md) | Egress Gateway — 파드 트래픽을 고정 IP로 외부 라우팅 |
| [bgp-guide.md](./bgp-guide.md) | BGP Control Plane — LoadBalancer IP를 BGP로 광고 |

### 문제 해결 & 실습
| 파일 | 설명 |
|---|---|
| [troubleshooting-guide.md](./troubleshooting-guide.md) | 자주 발생하는 문제와 진단 방법 |
| [e2e-practice.md](./e2e-practice.md) | End-to-End 실습 (설치 → 정책 → Hubble 관찰) |

---

## 저장소 구조

```
cilium-practice/
├── README.md
├── install.md                   # Helm 설치 가이드
├── architecture-guide.md        # Cilium 아키텍처
├── network-policy-guide.md      # 네트워크 정책
├── hubble-guide.md              # 가시성 (Hubble)
├── service-mesh-guide.md        # 서비스 메시
├── load-balancing-guide.md      # BPF 로드밸런싱
├── egress-gateway-guide.md      # Egress Gateway
├── bgp-guide.md                 # BGP Control Plane
├── troubleshooting-guide.md     # 트러블슈팅
├── e2e-practice.md              # End-to-End 실습
├── helm/
│   ├── values.yaml              # 개발/테스트용 Helm values
│   └── values-prod.yaml         # 운영용 Helm values
└── examples/
    ├── network-policy.yaml          # K8s NetworkPolicy 예시
    ├── cilium-network-policy.yaml   # CiliumNetworkPolicy 예시
    └── egress-gateway.yaml          # Egress Gateway 예시
```

---

## 아키텍처 요약

```
┌─────────────────────────────────────────────────────────────┐
│                        EKS Node                             │
│                                                             │
│  Pod A ──eBPF hook──▶ Cilium Agent ──▶ Policy Engine        │
│                            │                                │
│                     eBPF Map (kernel)                       │
│                            │                               │
│  Pod B ◀──eBPF hook────────┘                                │
│                                                             │
│  Hubble (Observer) ──▶ Hubble Relay ──▶ Hubble UI / CLI    │
└─────────────────────────────────────────────────────────────┘

Cilium Operator ──▶ IPAM (ENI 모드) ──▶ AWS EC2 ENI API
```

| 컴포넌트 | 역할 |
|---|---|
| **Cilium Agent** | 각 노드에서 실행, eBPF 프로그램 로드/관리, 정책 적용 |
| **Cilium Operator** | IPAM, CRD 관리, 가비지 컬렉션 |
| **eBPF Maps** | 커널 공간에서 패킷 포워딩/필터링 상태 저장 |
| **Hubble** | eBPF 기반 네트워크 흐름 관찰 (L3/L4/L7) |
| **Hubble Relay** | 클러스터 전체 흐름 집계 |
| **Hubble UI** | 실시간 서비스 맵 시각화 |

---

## 참고 링크

- [Cilium 공식 문서](https://docs.cilium.io/en/stable/)
- [Cilium GitHub](https://github.com/cilium/cilium)
- [Cilium Helm Chart](https://github.com/cilium/cilium/tree/main/install/kubernetes/cilium)
- [EKS에서 Cilium 사용하기](https://docs.cilium.io/en/stable/installation/k8s-install-helm/)
- [eBPF 공식 문서](https://ebpf.io/)
