# cilium-practice

A hands-on knowledge base for running Cilium on EKS — built from real operational experience.

- **Environment**: EKS / Cilium 1.16.x (ENI mode)
- **Namespaces**: CNI `kube-system` · app `default`

---

## Table of Contents

- [Learning Path](#learning-path)
- [Documents](#documents)
  - [Installation](#-installation-2-docs)
  - [Architecture](#-architecture-1-doc)
  - [Security](#-security-3-docs)
  - [Observability](#-observability-3-docs)
  - [Networking](#-networking-4-docs)
  - [Service Mesh](#-service-mesh-1-doc)
  - [Operations](#-operations-3-docs)
- [Manifest Structure](#manifest-structure)
- [Key Concept Summary](#key-concept-summary)

---

## Learning Path

```
1. Installation      → docs/install/
   ├── Install Cilium on EKS (ENI mode, kube-proxy replacement)
   └── Upgrade Cilium (rolling upgrade, rollback)

2. Core Concepts     → docs/architecture/
   └── eBPF, Cilium Agent, Operator, Hubble architecture

3. Advanced Topics
   ├── Security      → docs/security/       (NetworkPolicy, WireGuard, FQDN Egress)
   ├── Observability → docs/observability/   (Hubble, Prometheus metrics, Grafana)
   ├── Networking    → docs/networking/      (BPF LB, Egress Gateway, BGP, ENI mode)
   └── Service Mesh  → docs/service-mesh/   (Sidecar-less mTLS, L7 policy)

4. Operations        → docs/operations/
   ├── Troubleshooting common issues
   ├── eBPF deep debugging (cilium monitor, BPF maps)
   └── End-to-End hands-on lab
```

---

## Documents

### 📦 Installation (2 docs)

> Cilium 설치, 업그레이드, Helm 설정

| File | Description |
|------|-------------|
| [install.md](./docs/install/install.md) | Install Cilium on EKS via Helm (ENI mode, kube-proxy replacement) |
| [cilium-upgrade.md](./docs/install/cilium-upgrade.md) | Upgrade Cilium — Helm rolling upgrade, rollback, compatibility checks |

---

### 🏗️ Architecture (1 doc)

> eBPF 기반 아키텍처, 컴포넌트 구조, 패킷 흐름

| File | Description |
|------|-------------|
| [architecture-guide.md](./docs/architecture/architecture-guide.md) | Cilium architecture — eBPF, Agent, Operator, Hubble, ENI mode |

---

### 🔒 Security (3 docs)

> 네트워크 정책, 암호화, FQDN 기반 Egress 제어

| File | Description |
|------|-------------|
| [network-policy-guide.md](./docs/security/network-policy-guide.md) | Network policies — K8s NetworkPolicy, CiliumNetworkPolicy, Zero Trust patterns |
| [wireguard-encryption-guide.md](./docs/security/wireguard-encryption-guide.md) | WireGuard transparent encryption — node-to-node encryption, key management |
| [fqdn-egress-policy.md](./docs/security/fqdn-egress-policy.md) | FQDN-based egress policies — DNS proxy, external API allowlisting |

---

### 📊 Observability (3 docs)

> Hubble, Prometheus 메트릭, Grafana 대시보드

| File | Description |
|------|-------------|
| [hubble-guide.md](./docs/observability/hubble-guide.md) | Hubble — real-time flow observation, UI, Prometheus metrics |
| [prometheus-metrics.md](./docs/observability/prometheus-metrics.md) | Prometheus metrics — Cilium Agent/Hubble metrics, alert rules, ServiceMonitor |
| [grafana-dashboard-guide.md](./docs/observability/grafana-dashboard-guide.md) | Grafana dashboards — official dashboard import, custom panels, PromQL queries |

---

### 🌐 Networking (4 docs)

> BPF 로드밸런싱, Egress Gateway, BGP, ENI 모드

| File | Description |
|------|-------------|
| [load-balancing-guide.md](./docs/networking/load-balancing-guide.md) | BPF load balancing — kube-proxy replacement, DSR, Maglev hashing |
| [egress-gateway-guide.md](./docs/networking/egress-gateway-guide.md) | Egress Gateway — route pod traffic through fixed EIP |
| [bgp-guide.md](./docs/networking/bgp-guide.md) | BGP Control Plane — advertise LoadBalancer IPs via BGP |
| [eni-mode-guide.md](./docs/networking/eni-mode-guide.md) | ENI mode deep dive — IPAM, instance IP limits, prefix delegation, subnet management |

---

### 🔗 Service Mesh (1 doc)

> 사이드카 없는 eBPF 기반 서비스 메시

| File | Description |
|------|-------------|
| [service-mesh-guide.md](./docs/service-mesh/service-mesh-guide.md) | Cilium Service Mesh — sidecar-less mTLS, L7 policies, traffic management |

---

### 🔧 Operations (3 docs)

> 트러블슈팅, E2E 실습, eBPF 디버깅

| File | Description |
|------|-------------|
| [troubleshooting-guide.md](./docs/operations/troubleshooting-guide.md) | Common issues and diagnostic commands |
| [e2e-practice.md](./docs/operations/e2e-practice.md) | End-to-End lab (install → policy → Hubble observation) |
| [cilium-debug-guide.md](./docs/operations/cilium-debug-guide.md) | eBPF debugging — cilium monitor, BPF map inspection, packet tracing |

---

## Manifest Structure

```
examples/
├── network-policy.yaml          # K8s NetworkPolicy examples
├── cilium-network-policy.yaml   # CiliumNetworkPolicy examples (L7, FQDN, gRPC)
└── egress-gateway.yaml          # CiliumEgressGatewayPolicy example

helm/
├── values.yaml                  # dev/test Helm values
└── values-prod.yaml             # production Helm values (HA, WireGuard, XDP)
```

---

## Key Concept Summary

**eBPF** is the core technology powering Cilium — programs run in kernel space for packet processing.

```
┌─────────────────────────────────────────────────────────────┐
│                        EKS Node                             │
│                                                             │
│  Pod A ──eBPF hook──▶ Cilium Agent ──▶ Policy Engine        │
│                            │                                │
│                     eBPF Map (kernel)                       │
│                            │                                │
│  Pod B ◀──eBPF hook────────┘                                │
│                                                             │
│  Hubble (Observer) ──▶ Hubble Relay ──▶ Hubble UI / CLI    │
└─────────────────────────────────────────────────────────────┘

Cilium Operator ──▶ IPAM (ENI 모드) ──▶ AWS EC2 ENI API
```

| Component | Role |
|-----------|------|
| **Cilium Agent** | Per-node daemon: loads eBPF programs, enforces policies |
| **Cilium Operator** | IPAM management, CRD handling, garbage collection |
| **eBPF Maps** | Kernel-space state for packet forwarding and filtering |
| **Hubble** | eBPF-based network flow observation (L3/L4/L7) |
| **Hubble Relay** | Aggregates flows across the entire cluster |
| **Hubble UI** | Real-time service map visualization |

---

## References

- [Cilium Documentation](https://docs.cilium.io/en/stable/)
- [Cilium GitHub](https://github.com/cilium/cilium)
- [Cilium Helm Chart](https://github.com/cilium/cilium/tree/main/install/kubernetes/cilium)
- [Cilium on EKS](https://docs.cilium.io/en/stable/installation/k8s-install-helm/)
- [eBPF Documentation](https://ebpf.io/)
