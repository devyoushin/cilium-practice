# Cilium 아키텍처

---

## 전체 구조

```
┌──────────────────────────────────────────────────────────────────┐
│                         EKS 클러스터                              │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   Node A     │    │   Node B     │    │   Node C     │       │
│  │              │    │              │    │              │       │
│  │ Cilium Agent │    │ Cilium Agent │    │ Cilium Agent │       │
│  │  (DaemonSet) │    │  (DaemonSet) │    │  (DaemonSet) │       │
│  │      │       │    │      │       │    │      │       │       │
│  │  eBPF Maps   │    │  eBPF Maps   │    │  eBPF Maps   │       │
│  │  (kernel)    │    │  (kernel)    │    │  (kernel)    │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│           │                  │                  │                │
│           └──────────────────┴──────────────────┘                │
│                              │                                   │
│                    Cilium Operator                               │
│                    (IPAM / CRD 관리)                             │
│                              │                                   │
│                    Hubble Relay                                  │
│                    (흐름 집계)                                    │
│                              │                                   │
│                    Hubble UI / Prometheus                        │
└──────────────────────────────────────────────────────────────────┘
                              │
                     AWS ENI API (IPAM)
```

---

## 핵심 컴포넌트

### Cilium Agent (DaemonSet)

노드마다 1개씩 실행되는 핵심 컴포넌트입니다.

**주요 역할**:
- eBPF 프로그램을 컴파일하여 커널에 로드
- Kubernetes API 변화(파드, 서비스, 정책)를 감시하고 eBPF Map 업데이트
- 네트워크 정책(NetworkPolicy, CiliumNetworkPolicy) 적용
- 엔드포인트(파드) 생애주기 관리
- Hubble 관찰 데이터 생성

```bash
# Agent 로그 확인
kubectl -n kube-system logs ds/cilium -c cilium-agent

# Agent 내부 상태 조회
kubectl -n kube-system exec ds/cilium -- cilium status
kubectl -n kube-system exec ds/cilium -- cilium endpoint list
```

---

### Cilium Operator (Deployment)

클러스터 전체 작업을 담당하는 컨트롤 플레인입니다.

**주요 역할**:
- **IPAM**: ENI 모드에서 AWS ENI 생성/삭제 및 IP 할당
- CiliumIdentity 가비지 컬렉션 (미사용 ID 정리)
- CiliumNode 객체 관리
- 노드간 IP 충돌 방지

```bash
# Operator 로그 확인
kubectl -n kube-system logs deploy/cilium-operator
```

---

### eBPF (Extended Berkeley Packet Filter)

Cilium의 핵심 기술로, **커널 공간**에서 직접 패킷을 처리합니다.

```
전통적 방식:
  패킷 → 커널 네트워크 스택 → iptables → 사용자 공간 → 다시 커널 → 목적지
                                 (수천 개의 규칙 순차 검색)

Cilium eBPF 방식:
  패킷 → eBPF hook → eBPF Map 조회 → 즉시 포워딩/드롭
                    (O(1) 해시 맵 조회, 커널 공간에서 완결)
```

| eBPF 구성요소 | 설명 |
|---|---|
| **eBPF Programs** | 커널 이벤트(패킷 수신, 전송)에 부착되는 소형 C 프로그램 |
| **eBPF Maps** | 프로그램과 에이전트 간 공유 데이터 구조체 (해시맵, LRU 등) |
| **eBPF Verifier** | 프로그램 로드 전 안전성 검증 (무한루프/메모리 접근 오류 방지) |
| **JIT Compiler** | eBPF 바이트코드를 CPU 네이티브 코드로 컴파일 |

```bash
# 로드된 eBPF 프로그램 목록
kubectl -n kube-system exec ds/cilium -- bpftool prog list

# eBPF Map 목록
kubectl -n kube-system exec ds/cilium -- bpftool map list
```

---

### Hubble (관찰 레이어)

eBPF를 이용하여 **커널 레벨**에서 모든 네트워크 흐름을 캡처합니다.

```
Pod A → eBPF hook → Cilium Agent (Hubble Server)
                          │
                    Hubble Relay (클러스터 전체 집계)
                    ┌─────┴──────┐
              Hubble CLI     Hubble UI
              (CLI 쿼리)    (실시간 서비스 맵)
                          │
                    Prometheus (메트릭 노출)
                          │
                    Grafana (대시보드)
```

---

## ENI 모드 (AWS VPC 네이티브)

EKS에서 Cilium을 사용하는 가장 일반적인 방식입니다.

```
Cilium Operator
    │
    ├── AWS EC2 API 호출 → 새 ENI 생성 → 노드에 연결
    │
    └── Private IP 할당 → CiliumNode 객체에 기록
                              │
                        Cilium Agent
                              │
                        파드에 IP 직접 할당 (IPVLAN/VETH)
```

**장점**:
- AWS VPC CIDR 내에서 파드 IP 직접 할당 (오버레이 없음)
- VPC Flow Logs에서 파드 IP 직접 확인 가능
- AWS 보안그룹과 호환성 유지
- 낮은 레이턴시 (터널링 없음)

**제약**:
- 노드당 연결 가능한 ENI / IP 수 제한 (인스턴스 타입별 상이)

```bash
# 인스턴스 타입별 ENI/IP 한도 확인
aws ec2 describe-instance-types \
  --instance-types m5.large \
  --query "InstanceTypes[].NetworkInfo"
```

---

## ID 기반 보안 모델

Cilium은 IP 주소 대신 **보안 ID(Security Identity)** 를 기반으로 정책을 적용합니다.

```
파드에 레이블 부여 → Cilium이 레이블 조합으로 Identity 계산 → 숫자 ID 할당
                                                                    │
                                                         eBPF Map에 저장
                                                                    │
                                                    다른 노드의 파드와 통신 시
                                                    IP 대신 Identity로 정책 검사
```

```bash
# 현재 클러스터의 Identity 목록
kubectl get ciliumidentities

# 특정 파드의 Identity 확인
kubectl -n kube-system exec ds/cilium -- cilium endpoint list
```

**왜 IP 기반이 아닌가?**
- 파드 IP는 재시작마다 변경됨 → IP 기반 정책은 유지보수 어려움
- 레이블 기반 Identity는 파드가 재시작되어도 동일한 보안 정책 적용
- 마이크로서비스 환경에서 동적 확장/축소에 유연하게 대응

---

## 패킷 흐름 (같은 노드)

```
Pod A (eth0)
    │
    ▼
veth pair (lxcXXXXXX) ── TC ingress eBPF hook
    │
    ▼
eBPF: Policy 검사 → Identity 조회 → 허용/거부
    │ (허용)
    ▼
veth pair (lxcYYYYYY)
    │
    ▼
Pod B (eth0)
```

## 패킷 흐름 (다른 노드, ENI 모드)

```
Pod A (Node 1)
    │
    ▼
eBPF: 목적지 IP → 상대 노드 확인 → ENI를 통해 직접 전송
    │ (오버레이 없음, VPC 라우팅)
    ▼
Node 2의 ENI 수신
    │
    ▼
eBPF: 수신 처리 → Policy 검사 → Pod B 전달
    │
    ▼
Pod B (Node 2)
```

---

## 참고 링크

- [Cilium 아키텍처 공식 문서](https://docs.cilium.io/en/stable/overview/component-overview/)
- [eBPF 설명](https://docs.cilium.io/en/stable/overview/intro/)
- [ENI 모드 문서](https://docs.cilium.io/en/stable/installation/cni-chaining-aws-cni/)
- [Security Identity](https://docs.cilium.io/en/stable/04-security/policy/language/#identity)
