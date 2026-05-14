# Cilium eBPF 디버깅 가이드

## 1. 개요

Cilium은 커널 공간(kernel space)에서 eBPF 프로그램을 실행하여 네트워킹, 보안, 관찰 가능성(observability)을 구현하는 CNI. 기존 `kubectl` 기반 트러블슈팅만으로는 eBPF 데이터패스(datapath) 내부에서 발생하는 문제를 진단하기 어려우며, 전용 디버깅 도구가 필요함.

본 가이드가 다루는 범위:

- **cilium CLI**: 엔드포인트(endpoint) 상태 조회, 정책(policy) 확인, 서비스(service) 매핑 검증
- **BPF 맵(map) 검사**: 커널에 로드된 eBPF 맵의 직접 조회를 통한 데이터패스 상태 확인
- **cilium monitor**: 실시간 eBPF 이벤트 추적(tracing)으로 패킷 드롭(drop), 정책 판정(policy verdict) 분석
- **cilium debuginfo**: 전체 진단 덤프(diagnostic dump) 수집
- **패킷 레벨 디버깅**: tcpdump 및 정책 트레이스(policy trace)를 활용한 심층 분석

> 일반적인 네트워크 트러블슈팅으로 해결되지 않는 문제에 대해 eBPF 수준까지 내려가 원인을 파악하는 것이 본 가이드의 목적.

---

## 2. 설명

### 2.1 핵심 개념

#### 디버깅 계층(Debugging Layers)

Cilium 디버깅은 추상화 수준에 따라 네 가지 계층으로 구분됨.

| 계층 | 도구 | 설명 | 사용 시점 |
|------|------|------|-----------|
| **Level 1** (최상위) | `cilium CLI` | 엔드포인트, 서비스, 정책 상태 조회 | 초기 진단 및 상태 확인 |
| **Level 2** | BPF 맵 검사 | 커널 내 eBPF 맵 직접 조회 | 데이터패스 내부 상태 확인 |
| **Level 3** | `cilium monitor` | 실시간 eBPF 이벤트 스트리밍 | 패킷 드롭/정책 판정 추적 |
| **Level 4** (최하위) | `cilium debuginfo` | 전체 진단 리포트 생성 | 지원 요청 및 종합 분석 |

#### Cilium 드롭 사유(Drop Reasons) 참조 테이블

`cilium monitor --type drop` 출력에서 확인되는 주요 드롭 사유 코드 목록.

| 드롭 사유 코드 | 의미 | 주요 원인 |
|----------------|------|-----------|
| `POLICY_DENIED` | 정책에 의한 차단 | 네트워크 정책(NetworkPolicy) 또는 CiliumNetworkPolicy에서 해당 트래픽을 허용하지 않음 |
| `INVALID_SOURCE_IP` | 유효하지 않은 소스 IP | 소스 IP가 클러스터 CIDR 범위에 속하지 않거나, IP 스푸핑(spoofing) 감지 |
| `NO_TUNNEL_OR_ENCAP_MAPPING` | 터널/캡슐화 매핑 없음 | 오버레이(overlay) 모드에서 대상 노드에 대한 터널 엔트리 부재 |
| `UNKNOWN_L3_TARGET` | 알 수 없는 L3 대상 | 목적지 IP에 대한 라우팅 정보 없음, IP 캐시(ipcache)에 엔트리 부재 |
| `STALE_OR_UNROUTABLE` | 오래된 또는 라우팅 불가 엔트리 | CT(connection tracking) 테이블의 만료된 엔트리 또는 라우팅 테이블 불일치 |
| `UNSUPPORTED_L3_PROTOCOL` | 지원하지 않는 L3 프로토콜 | IPv4/IPv6 이외의 프로토콜 패킷 |
| `NO_MAPPING_FOR_NAT` | NAT 매핑 없음 | 마스커레이드(masquerade) 또는 서비스 NAT 엔트리 누락 |
| `CT_TRUNCATED_OR_INVALID` | CT 엔트리 손상 또는 유효하지 않음 | 커넥션 트래킹 테이블 손상 |
| `FRAG_NEEDED` | 단편화(fragmentation) 필요 | MTU 초과 패킷, 경로 MTU 탐색(PMTUD) 실패 |
| `IS_CLUSTER_IP` | ClusterIP 직접 접근 | ClusterIP로의 직접 접근 시도 (kube-proxy 없는 모드) |

#### 엔드포인트 상태(Endpoint States)

`cilium endpoint list` 출력에서 확인되는 엔드포인트 상태 및 의미.

| 상태 | 의미 | 조치 필요 여부 |
|------|------|----------------|
| `ready` | 정상 동작 중, 정책 적용 완료 | 없음 |
| `waiting-for-identity` | 아이덴티티(identity) 할당 대기 중 | 일시적인 상태이며, 장시간 지속 시 kvstore 또는 CRD 기반 아이덴티티 할당 문제 확인 필요 |
| `not-ready` | 엔드포인트 초기화 미완료 | BPF 프로그램 로드 실패 또는 초기화 오류 확인 필요 |
| `disconnecting` | 엔드포인트 제거 진행 중 | 파드(pod) 종료 중이므로 정상적인 전환 상태 |
| `disconnected` | 엔드포인트 연결 해제 완료 | 정리(cleanup) 대기 중인 상태 |
| `invalid` | 유효하지 않은 엔드포인트 | 설정 오류 또는 BPF 프로그램 컴파일 실패, 로그 확인 필요 |
| `restoring` | 에이전트 재시작 후 복원 중 | Cilium 에이전트 재시작 시 일시적으로 나타나는 정상 상태 |

---

### 2.2 실무 적용 코드

> 아래 명령은 Cilium 에이전트 파드 내부에서 실행하는 것을 기본으로 함. `kubectl exec`를 통해 접근하거나, `cilium` CLI가 로컬에 설치된 경우 직접 실행 가능.

```bash
# Cilium 에이전트 파드 셸 접근 (특정 노드의 에이전트)
kubectl -n kube-system exec -it <CILIUM_POD_NAME> -- bash

# 또는 DaemonSet 기반 접근 (임의 노드)
kubectl -n kube-system exec -it ds/cilium -- bash
```

#### 2.2.1 cilium monitor (실시간 eBPF 이벤트 추적)

`cilium monitor`는 커널의 eBPF 프로그램이 생성하는 이벤트를 실시간으로 스트리밍하는 도구. 패킷 드롭, 정책 판정, 트레이스(trace) 이벤트를 확인할 수 있음.

```bash
# 특정 엔드포인트와 관련된 모든 트래픽 추적
# ENDPOINT_ID: cilium endpoint list로 확인 가능
cilium monitor --related-to <ENDPOINT_ID>

# 드롭(drop)된 패킷만 필터링
# 가장 자주 사용하는 패턴 — 왜 패킷이 버려졌는지 확인
cilium monitor --type drop

# 정책 판정(policy verdict)만 필터링
# allow/deny 결정 과정을 실시간으로 확인
cilium monitor --type policy-verdict

# 특정 파드의 트래픽만 추적
cilium monitor --from-pod <NAMESPACE>/<POD_NAME>

# 드롭 이벤트를 상세 모드(verbose)로 확인
cilium monitor --type drop -v

# L7(애플리케이션 계층) 이벤트 추적 (HTTP, DNS, Kafka 등)
cilium monitor --type l7

# 특정 엔드포인트의 드롭 이벤트만 필터링
cilium monitor --related-to <ENDPOINT_ID> --type drop

# 트레이스 이벤트 (패킷 경로 추적)
cilium monitor --type trace

# 출력을 JSON 형식으로 변환 (후처리에 유용)
cilium monitor --type drop -o json
```

**cilium monitor 출력 해석 예시:**

```
xx drop (Policy denied) flow 0x0 to endpoint <ENDPOINT_ID>, identity <SRC_IDENTITY>-><DST_IDENTITY>:
    <SRC_IP>:<SRC_PORT> -> <DST_IP>:<DST_PORT> TCP Flags: SYN
```

- `drop (Policy denied)`: 드롭 사유 - 정책에 의한 차단
- `identity <SRC_IDENTITY>-><DST_IDENTITY>`: 소스 및 대상의 Cilium 아이덴티티
- `TCP Flags: SYN`: 드롭된 패킷의 TCP 플래그

---

#### 2.2.2 BPF 맵 조사

커널에 로드된 eBPF 맵을 직접 조회하여 데이터패스의 실제 상태를 확인하는 명령 모음.

```bash
# === 엔드포인트(Endpoint) 관련 ===

# 엔드포인트 목록 조회 (ID, 아이덴티티, 정책 상태, 상태 포함)
cilium endpoint list

# 특정 엔드포인트 상세 정보 (정책, 라벨, 네트워킹 설정)
cilium endpoint get <ENDPOINT_ID>

# 엔드포인트의 BPF 로그 확인
cilium endpoint log <ENDPOINT_ID>

# 엔드포인트 헬스 체크
cilium endpoint health <ENDPOINT_ID>


# === 로드밸런싱(Load Balancing) 관련 ===

# BPF 로드밸런서 엔트리 전체 조회
# 서비스 ClusterIP -> 백엔드(backend) 파드 매핑 확인
cilium bpf lb list

# 특정 서비스에 대한 로드밸런서 엔트리 필터링
cilium bpf lb list | grep <SERVICE_IP>

# 백엔드 목록 조회
cilium bpf lb list --backends

# 서비스 목록 (cilium 관점)
cilium service list

# 특정 서비스 상세 정보
cilium service get <SERVICE_ID>


# === 커넥션 트래킹(Connection Tracking) 관련 ===

# 글로벌 CT(connection tracking) 테이블 조회
cilium bpf ct list global

# 특정 IP의 CT 엔트리 필터링
cilium bpf ct list global | grep <POD_IP>

# CT 테이블 플러시 (주의: 기존 연결 초기화됨)
cilium bpf ct flush global


# === 정책(Policy) 관련 ===

# 모든 엔드포인트의 BPF 정책 맵 조회
cilium bpf policy get --all

# 특정 엔드포인트의 정책 맵
cilium bpf policy get <ENDPOINT_ID>

# 현재 적용된 정책 목록
cilium policy get

# 정책 셀렉터(selector) 상태 확인
cilium policy selectors


# === 터널(Tunnel) 관련 ===

# 터널 맵 조회 (오버레이 모드 사용 시)
# ENI 모드에서는 일반적으로 비어 있음
cilium bpf tunnel list


# === NAT 관련 ===

# NAT 엔트리 조회
cilium bpf nat list

# 특정 IP의 NAT 엔트리 필터링
cilium bpf nat list | grep <POD_IP>


# === IP 캐시(Identity Mapping) 관련 ===

# IP 캐시 전체 조회 (IP -> 아이덴티티 매핑)
cilium bpf ipcache list

# 특정 IP의 아이덴티티 확인
cilium bpf ipcache list | grep <POD_IP>

# 아이덴티티 상세 정보
cilium identity get <IDENTITY_ID>

# 아이덴티티 목록 조회
cilium identity list
```

---

#### 2.2.3 cilium debuginfo (진단 리포트)

문제 재현이 어렵거나 종합적인 상태 분석이 필요한 경우, 전체 진단 정보를 수집하는 명령.

```bash
# 전체 진단 덤프를 tar 파일로 저장
cilium debuginfo -f /tmp/cilium-debug.tar

# 진단 파일을 로컬로 복사
kubectl cp kube-system/<CILIUM_POD_NAME>:/tmp/cilium-debug.tar ./cilium-debug.tar

# Cilium 상태 상세 조회
cilium status --verbose

# 현재 적용된 정책 전체 덤프
cilium policy get

# 서비스 목록 전체 덤프
cilium service list

# 노드 목록 및 상태 확인
cilium node list

# BPF 프로그램 목록 (커널에 로드된 eBPF 프로그램)
cilium bpf prog list

# cilium-bugtool을 통한 종합 진단 (에이전트 외부에서 실행)
kubectl -n kube-system exec <CILIUM_POD_NAME> -- cilium-bugtool
kubectl cp kube-system/<CILIUM_POD_NAME>:/tmp/cilium-bugtool-<TIMESTAMP>.tar \
  ./cilium-bugtool.tar

# Cilium 에이전트 로그 수집
kubectl -n kube-system logs <CILIUM_POD_NAME> --since=1h > cilium-agent.log

# 모든 노드의 Cilium 에이전트 로그 수집
for pod in $(kubectl -n kube-system get pods -l k8s-app=cilium \
  -o jsonpath='{.items[*].metadata.name}'); do
  kubectl -n kube-system logs "$pod" --since=1h > "cilium-${pod}.log"
done
```

---

#### 2.2.4 패킷 레벨 디버깅

eBPF 레벨 디버깅으로도 충분하지 않을 때, 패킷 캡처(packet capture) 및 정책 트레이스를 수행하는 명령.

```bash
# === tcpdump 기반 패킷 캡처 ===

# Cilium 에이전트 파드에서 특정 파드 IP의 트래픽 캡처
kubectl -n kube-system exec ds/cilium -- \
  tcpdump -i any -n host <POD_IP>

# 특정 인터페이스에서 캡처 (lxc: 파드 veth, cilium_vxlan: 오버레이)
kubectl -n kube-system exec ds/cilium -- \
  tcpdump -i lxc+ -n host <POD_IP>

# 특정 포트의 트래픽만 캡처
kubectl -n kube-system exec ds/cilium -- \
  tcpdump -i any -n host <POD_IP> and port <PORT>

# DNS 트래픽 캡처 (포트 53)
kubectl -n kube-system exec ds/cilium -- \
  tcpdump -i any -n port 53

# pcap 파일로 저장 후 로컬로 복사 (Wireshark 분석용)
kubectl -n kube-system exec ds/cilium -- \
  tcpdump -i any -n host <POD_IP> -w /tmp/capture.pcap -c 1000
kubectl cp kube-system/<CILIUM_POD_NAME>:/tmp/capture.pcap ./capture.pcap


# === 정책 트레이스(Policy Trace) ===

# 소스/대상 아이덴티티 기반 정책 판정 시뮬레이션
cilium policy trace \
  --src-identity <SRC_IDENTITY_ID> \
  --dst-identity <DST_IDENTITY_ID> \
  --dport <DESTINATION_PORT>

# 소스/대상 엔드포인트 기반 정책 트레이스
cilium policy trace \
  --src-endpoint <SRC_ENDPOINT_ID> \
  --dst-endpoint <DST_ENDPOINT_ID> \
  --dport <DESTINATION_PORT>

# 라벨 기반 정책 트레이스
cilium policy trace \
  --src-k8s-pod <SRC_NAMESPACE>/<SRC_POD_NAME> \
  --dst-k8s-pod <DST_NAMESPACE>/<DST_POD_NAME> \
  --dport <DESTINATION_PORT>


# === 커널 수준 eBPF 디버깅 ===

# 현재 로드된 BPF 프로그램 확인
kubectl -n kube-system exec ds/cilium -- bpftool prog list

# 특정 BPF 프로그램의 통계
kubectl -n kube-system exec ds/cilium -- bpftool prog show id <PROG_ID>

# BPF 맵 통계
kubectl -n kube-system exec ds/cilium -- bpftool map list
```

---

### 2.3 보안/성능 Best Practice

#### cilium monitor 사용 시 주의사항

- **운영 환경(production)에서 최소한으로 사용**: `cilium monitor`는 커널의 eBPF 이벤트를 유저 공간(user space)으로 복사하므로, 트래픽이 많은 환경에서 CPU 오버헤드 발생 가능
- **필터를 반드시 적용**: `--type`, `--related-to`, `--from-pod` 등 필터 없이 전체 이벤트를 수신하면 성능 영향이 크게 증가함
- **짧은 시간 동안만 실행**: 장시간 실행 금지. 필요한 이벤트를 포착하면 즉시 종료할 것

#### 디버그 로깅(Debug Logging) 관리

```bash
# 디버그 로깅 일시적 활성화
cilium config set debug true

# 디버그 로깅 비활성화 (반드시 작업 후 복원)
cilium config set debug false
```

- 디버그 로깅은 **일시적으로만** 활성화할 것. 영구 활성화 시 로그 볼륨 폭증 및 스토리지 고갈 위험
- ConfigMap을 통한 활성화보다 `cilium config set`을 통한 런타임 변경 권장 (에이전트 재시작 불필요)

#### 진단 정보 수집 원칙

- **지원 티켓(support ticket) 제출 전 반드시 `cilium debuginfo` 수집**: 재현이 어려운 문제의 경우 사후 분석에 필수적임
- **`cilium-bugtool` 활용**: debuginfo보다 더 포괄적인 진단 정보(커널 설정, 시스템 정보 포함) 수집 가능
- **타임스탬프(timestamp) 기록**: 문제 발생 시점의 정확한 시간을 기록하여 로그 분석 시 활용

#### 관찰 도구 선택 기준

| 목적 | 권장 도구 | 사유 |
|------|-----------|------|
| 지속적인 트래픽 모니터링 | Hubble (`hubble observe`) | 낮은 오버헤드, 구조화된 출력, 필터링 용이 |
| 실시간 심층 디버깅 | `cilium monitor` | eBPF 이벤트 직접 접근, 세밀한 분석 가능 |
| 문제 재현 및 분석 | `cilium debuginfo` + 로그 | 종합적인 상태 스냅샷 제공 |
| 패킷 수준 분석 | tcpdump + Wireshark | eBPF 이전/이후 패킷 비교 가능 |

---

## 3. 트러블슈팅

### 3.1 주요 이슈

#### 시나리오 1: Pod-to-Pod 통신 차단

**증상**: 파드 A에서 파드 B로 통신이 되지 않음. `curl` 또는 `wget` 타임아웃 발생.

```bash
# 1단계: 소스/대상 파드의 엔드포인트 ID 및 상태 확인
cilium endpoint list | grep <POD_A_IP>
cilium endpoint list | grep <POD_B_IP>

# 2단계: 실시간 드롭 이벤트 확인
cilium monitor --type drop --related-to <ENDPOINT_ID_B>

# 예상 출력:
# xx drop (Policy denied) flow 0x0 to endpoint <ENDPOINT_ID_B>,
#   identity <IDENTITY_A>-><IDENTITY_B>:
#   <POD_A_IP>:<PORT> -> <POD_B_IP>:<PORT> TCP Flags: SYN

# 3단계: 드롭 사유가 "Policy denied"인 경우, 정책 트레이스 실행
cilium policy trace \
  --src-k8s-pod <NAMESPACE_A>/<POD_A_NAME> \
  --dst-k8s-pod <NAMESPACE_B>/<POD_B_NAME> \
  --dport <DESTINATION_PORT>

# 4단계: 현재 적용된 정책 확인
cilium policy get

# 5단계: 대상 엔드포인트의 정책 맵에서 소스 아이덴티티 허용 여부 확인
cilium bpf policy get <ENDPOINT_ID_B>

# 해결: 누락된 CiliumNetworkPolicy 또는 NetworkPolicy 생성/수정
kubectl apply -f <NETWORK_POLICY_YAML>

# 검증: 정책 적용 후 드롭 이벤트가 더 이상 발생하지 않는지 확인
cilium monitor --type policy-verdict --related-to <ENDPOINT_ID_B>
```

---

#### 시나리오 2: 서비스 로드밸런싱 미동작

**증상**: 서비스(Service) ClusterIP로 접근하면 응답이 없거나, 특정 백엔드로만 트래픽이 전달됨.

```bash
# 1단계: Cilium이 인식하는 서비스 목록 확인
cilium service list | grep <SERVICE_NAME>
# 또는 ClusterIP로 검색
cilium service list | grep <SERVICE_CLUSTER_IP>

# 2단계: BPF 로드밸런서 맵에서 백엔드 매핑 확인
cilium bpf lb list | grep <SERVICE_CLUSTER_IP>

# 정상 출력 예시:
# <SERVICE_CLUSTER_IP>:<PORT> (3) [...]
#   => backend 1: <BACKEND_POD_IP_1>:<TARGET_PORT>
#   => backend 2: <BACKEND_POD_IP_2>:<TARGET_PORT>

# 3단계: 백엔드 파드가 실제로 존재하고 Ready 상태인지 확인
kubectl get endpoints <SERVICE_NAME> -n <NAMESPACE>

# 4단계: 백엔드가 BPF 맵에 없는 경우, 엔드포인트 상태 확인
cilium endpoint list | grep <BACKEND_POD_IP>

# 5단계: CT 테이블에서 관련 연결 상태 확인
cilium bpf ct list global | grep <SERVICE_CLUSTER_IP>

# 해결 방법 1: CT 테이블 플러시로 오래된 연결 정리
cilium bpf ct flush global

# 해결 방법 2: Cilium 에이전트 재시작 (최후의 수단)
kubectl -n kube-system rollout restart daemonset/cilium
```

---

#### 시나리오 3: DNS 해석(Resolution) 실패

**증상**: 파드 내부에서 `nslookup` 또는 `dig` 명령이 실패하거나 타임아웃 발생.

```bash
# 1단계: DNS 관련 L7 이벤트 확인
cilium monitor --type l7 --related-to <ENDPOINT_ID>

# 2단계: DNS 프록시(proxy) 상태 확인
cilium status --verbose | grep -A 5 "DNS"

# 3단계: DNS 트래픽이 Cilium DNS 프록시를 경유하는지 확인
cilium bpf lb list | grep ":53"

# 4단계: kube-dns/coredns 서비스 엔드포인트 확인
cilium service list | grep kube-dns

# 5단계: 패킷 레벨에서 DNS 요청/응답 확인
kubectl -n kube-system exec ds/cilium -- \
  tcpdump -i any -n port 53 -v

# 6단계: DNS 프록시 로그 확인 (디버그 모드 필요)
cilium config set debug true
kubectl -n kube-system logs <CILIUM_POD_NAME> | grep -i dns
# 작업 완료 후 반드시 비활성화
cilium config set debug false

# 해결 방법 1: DNS 정책이 존재하는 경우, DNS 규칙 확인
cilium policy get | grep -A 10 "dns"

# 해결 방법 2: FQDN 정책 캐시 확인
cilium fqdn cache list

# 해결 방법 3: cilium의 DNS 프록시 바인딩 포트 확인
cilium status | grep "DNS"
```

---

#### 시나리오 4: 외부 연결(External Connectivity) 장애

**증상**: 파드에서 외부 인터넷(예: `google.com`)으로 통신이 되지 않음. 클러스터 내부 통신은 정상.

```bash
# 1단계: NAT 엔트리 확인 (마스커레이드 설정)
cilium bpf nat list | grep <POD_IP>

# 2단계: 마스커레이드(masquerade) 설정 확인
cilium status --verbose | grep -i masq

# 3단계: 외부로 향하는 트래픽의 드롭 이벤트 확인
cilium monitor --type drop --from-pod <NAMESPACE>/<POD_NAME>

# 4단계: 라우팅 확인 (ENI 모드에서 중요)
kubectl -n kube-system exec ds/cilium -- ip route show

# 5단계: BPF 라우팅 맵 확인
cilium bpf tunnel list

# 6단계: 노드의 iptables/NAT 규칙 확인 (ENI 모드)
kubectl -n kube-system exec ds/cilium -- iptables -t nat -L -n

# 7단계: 패킷 캡처로 소스 IP 변환 확인
kubectl -n kube-system exec ds/cilium -- \
  tcpdump -i any -n host <EXTERNAL_IP> -v

# 해결 방법 1: 마스커레이드 설정이 비활성화된 경우 Helm 값 확인
# values.yaml에서 enableIPv4Masquerade: true 확인

# 해결 방법 2: ENI 모드에서 서브넷 라우팅 테이블 확인
# AWS 콘솔에서 VPC 라우팅 테이블에 인터넷 게이트웨이(IGW) 경로 확인

# 해결 방법 3: 보안 그룹(Security Group) 아웃바운드 규칙 확인
```

---

#### 시나리오 5: 아이덴티티(Identity) 할당 문제

**증상**: 새로 생성된 파드가 `waiting-for-identity` 상태에서 벗어나지 않음. 네트워크 정책이 적용되지 않음.

```bash
# 1단계: 엔드포인트 상태 확인
cilium endpoint list | grep "waiting-for-identity"

# 2단계: IP 캐시에서 해당 파드의 아이덴티티 매핑 확인
cilium bpf ipcache list | grep <POD_IP>

# 3단계: 아이덴티티 목록에서 해당 라벨의 아이덴티티 존재 여부 확인
cilium identity list | grep <POD_LABEL_VALUE>

# 4단계: Cilium 에이전트 로그에서 아이덴티티 관련 오류 확인
kubectl -n kube-system logs <CILIUM_POD_NAME> | grep -i "identity"

# 5단계: kvstore(etcd) 또는 CRD 상태 확인
cilium status --verbose | grep -A 5 "KVStore"

# CRD 기반 아이덴티티인 경우
kubectl get ciliumidentities -A

# 6단계: 특정 아이덴티티 상세 정보
cilium identity get <IDENTITY_ID>

# 해결 방법 1: Cilium 에이전트 재시작으로 아이덴티티 재할당 트리거
kubectl -n kube-system delete pod <CILIUM_POD_NAME>

# 해결 방법 2: kvstore 연결 문제인 경우 etcd 상태 확인
cilium status | grep -i kvstore

# 해결 방법 3: 아이덴티티 할당 상한(limit) 확인
cilium status --verbose | grep -i identity
```

---

#### 시나리오 6 (보너스): eBPF 프로그램 로드 실패

**증상**: Cilium 에이전트가 시작되지만 엔드포인트가 `not-ready` 또는 `invalid` 상태. 에이전트 로그에 BPF 컴파일 오류 발생.

```bash
# 1단계: 에이전트 로그에서 BPF 관련 오류 확인
kubectl -n kube-system logs <CILIUM_POD_NAME> | grep -i "bpf\|compile\|verifier"

# 2단계: BPF 프로그램 로드 상태 확인
cilium bpf prog list

# 3단계: 커널 버전 및 BPF 기능 확인
kubectl -n kube-system exec ds/cilium -- uname -r
kubectl -n kube-system exec ds/cilium -- cat /proc/config.gz | \
  gunzip | grep CONFIG_BPF

# 4단계: BPF 파일시스템 마운트 확인
kubectl -n kube-system exec ds/cilium -- mount | grep bpf

# 해결: 커널 버전 업그레이드 또는 Cilium 설정에서 호환되지 않는 기능 비활성화
```

---

### 3.2 자주 발생하는 문제 (Q&A)

**Q1: 디버그 로깅(debug logging)을 활성화하는 방법은?**

```bash
# 방법 1: 런타임에서 즉시 활성화 (에이전트 재시작 불필요, 권장)
kubectl -n kube-system exec <CILIUM_POD_NAME> -- cilium config set debug true

# 방법 2: ConfigMap을 통한 활성화 (에이전트 재시작 필요)
kubectl -n kube-system edit configmap cilium-config
# debug: "true" 설정 후 저장

# 방법 3: Helm 값을 통한 활성화
# helm upgrade cilium cilium/cilium --set debug.enabled=true

# 비활성화 (반드시 작업 후 복원)
kubectl -n kube-system exec <CILIUM_POD_NAME> -- cilium config set debug false
```

> 주의: 디버그 로깅은 로그 볼륨을 10배 이상 증가시킬 수 있음. 운영 환경에서는 반드시 일시적으로만 활성화하고, 분석 완료 후 즉시 비활성화할 것.

---

**Q2: cilium monitor 출력을 해석하는 방법은?**

`cilium monitor` 출력의 주요 필드 해석:

```
# 형식: <방향> <이벤트유형> (<사유>) flow <플로우해시> to/from endpoint <EPID>,
#        identity <소스ID>-><대상ID>: <소스IP>:<소스포트> -> <대상IP>:<대상포트> <프로토콜>

# 예시 1: 정책에 의한 드롭
-> drop (Policy denied) flow 0x12ab to endpoint 1234, identity 5678->9012:
   10.0.1.5:43210 -> 10.0.2.10:80 TCP Flags: SYN

# 예시 2: 정책 허용 판정
-> policy-verdict:allow filter EGRESS, identity 5678->9012:
   10.0.1.5:43210 -> 10.0.2.10:80 TCP Flags: SYN

# 예시 3: L7 이벤트 (HTTP)
-> l7 response: proxy 1234 identity 5678->9012 HTTP/1.1 200 GET http://example/api
```

| 필드 | 설명 |
|------|------|
| `->` / `<-` | 트래픽 방향 (인바운드/아웃바운드) |
| `drop` / `policy-verdict` / `trace` | 이벤트 유형 |
| `(Policy denied)` | 드롭 사유 코드 |
| `flow 0x12ab` | 플로우 해시 (같은 연결의 패킷 식별) |
| `endpoint 1234` | 관련 엔드포인트 ID |
| `identity 5678->9012` | 소스/대상 Cilium 보안 아이덴티티 |

---

**Q3: cilium monitor와 hubble observe 중 어느 것을 사용해야 하는가?**

| 기준 | `cilium monitor` | `hubble observe` |
|------|-------------------|-------------------|
| **추상화 수준** | 낮음 (eBPF 이벤트 직접 접근) | 높음 (구조화된 플로우 데이터) |
| **성능 영향** | 상대적으로 높음 | 낮음 (링 버퍼 기반) |
| **출력 형식** | 로우 이벤트 텍스트 | 구조화된 플로우 (JSON, compact) |
| **필터링** | 기본 필터 (type, endpoint) | 풍부한 필터 (namespace, pod, verdict, HTTP 등) |
| **사용 시점** | 심층 디버깅, 드롭 사유 분석, BPF 레벨 문제 | 일상적인 트래픽 모니터링, 플로우 분석 |
| **실행 위치** | Cilium 에이전트 파드 내부 | Hubble Relay를 통해 클러스터 외부에서도 실행 가능 |

**권장 사용 전략:**

1. 먼저 `hubble observe`로 트래픽 패턴 파악
2. 문제 범위를 좁힌 후 `cilium monitor`로 심층 분석
3. 필요 시 `tcpdump`로 패킷 수준 검증

---

**Q4: 특정 파드의 eBPF 엔드포인트 ID를 찾는 방법은?**

```bash
# 방법 1: cilium endpoint list에서 파드 이름으로 검색
cilium endpoint list | grep <POD_NAME>

# 방법 2: 파드 IP로 검색
POD_IP=$(kubectl get pod <POD_NAME> -n <NAMESPACE> -o jsonpath='{.status.podIP}')
cilium endpoint list | grep "$POD_IP"

# 방법 3: 라벨로 검색
cilium endpoint list | grep <LABEL_VALUE>
```

---

**Q5: BPF 맵이 가득 찬 경우 어떻게 확인하는가?**

```bash
# CT 맵 크기 확인
cilium bpf ct list global | wc -l

# NAT 맵 크기 확인
cilium bpf nat list | wc -l

# 맵 크기 제한 확인 (Cilium 설정)
cilium status --verbose | grep -i "map"

# 맵이 가득 찬 경우 에이전트 로그에서 관련 경고 확인
kubectl -n kube-system logs <CILIUM_POD_NAME> | grep -i "map full\|map size"
```

---

## 4. 모니터링 및 확인

### cilium status 기반 확인

```bash
# Cilium 전체 상태 요약
cilium status

# 상세 상태 (컨트롤러, KVStore, 클러스터 메시, 등)
cilium status --verbose

# 출력에서 확인해야 할 핵심 항목:
# - KVStore: Ok (Connected)         ← kvstore 연결 상태
# - Kubernetes: Ok                  ← K8s API 서버 연결 상태
# - Controller Status: X/Y healthy  ← 내부 컨트롤러 정상 비율
# - Proxy Status: OK                ← L7 프록시 상태
# - Cluster health: X/Y reachable   ← 노드 간 헬스체크 상태
```

### cilium debuginfo 기반 진단

```bash
# 전체 진단 덤프 생성 및 수집
cilium debuginfo -f /tmp/cilium-debug.tar

# 특정 컴포넌트만 확인할 경우
cilium status --verbose       # 전반적 상태
cilium policy get             # 정책 상태
cilium service list           # 서비스 매핑
cilium endpoint list          # 엔드포인트 상태
cilium node list              # 노드 상태
```

### Prometheus 메트릭(Metrics) 기반 디버깅

Cilium이 노출하는 Prometheus 메트릭 중 디버깅에 유용한 항목.

| 메트릭 | 설명 | 디버깅 활용 |
|--------|------|-------------|
| `cilium_drop_count_total` | 드롭된 패킷 수 (사유별) | `reason` 라벨로 드롭 사유 분류 가능 |
| `cilium_forward_count_total` | 포워딩된 패킷 수 | 정상 트래픽 기준선(baseline) 파악 |
| `cilium_policy_verdict_total` | 정책 판정 수 (allow/deny) | 정책 위반 추세 분석 |
| `cilium_endpoint_state` | 엔드포인트 상태별 수 | `not-ready` 또는 `invalid` 수 모니터링 |
| `cilium_bpf_map_ops_total` | BPF 맵 연산 수 | 맵 연산 오류 감지 |
| `cilium_agent_bootstrap_seconds` | 에이전트 부트스트랩 소요 시간 | 에이전트 시작 지연 문제 분석 |
| `cilium_k8s_client_api_calls_total` | K8s API 호출 수 | API 서버 과부하 여부 확인 |

```promql
# PromQL 예시: 드롭 사유별 패킷 수 (최근 5분)
rate(cilium_drop_count_total[5m])

# 정책 거부(deny) 판정 추세
rate(cilium_policy_verdict_total{verdict="denied"}[5m])

# not-ready 상태의 엔드포인트 수
cilium_endpoint_state{state="not-ready"}

# BPF 맵 연산 오류
rate(cilium_bpf_map_ops_total{outcome="error"}[5m])
```

### Hubble 기반 플로우 모니터링

```bash
# 실시간 플로우 관찰
hubble observe --follow

# 특정 네임스페이스의 드롭된 플로우만 확인
hubble observe --namespace <NAMESPACE> --verdict DROPPED

# 특정 파드의 플로우 확인
hubble observe --pod <NAMESPACE>/<POD_NAME>

# HTTP 플로우만 필터링
hubble observe --protocol http

# JSON 출력 (후처리에 유용)
hubble observe --namespace <NAMESPACE> -o json

# Hubble UI를 통한 시각적 모니터링
kubectl -n kube-system port-forward svc/hubble-ui 12000:80
# 브라우저에서 http://localhost:12000 접속
```

---

## 5. TIP

### 공식 문서 링크

- [Cilium Troubleshooting Guide](https://docs.cilium.io/en/stable/operations/troubleshooting/)
- [Cilium Monitor Reference](https://docs.cilium.io/en/stable/observability/visibility/)
- [BPF and XDP Reference](https://docs.cilium.io/en/stable/bpf/)
- [Cilium CLI Reference](https://docs.cilium.io/en/stable/cmdref/)
- [Hubble Documentation](https://docs.cilium.io/en/stable/observability/hubble/)

### 증상별 디버깅 명령 빠른 참조 테이블(Quick Reference)

| 증상 | 1차 명령 | 2차 명령 | 3차 명령 |
|------|----------|----------|----------|
| 파드 간 통신 차단 | `cilium monitor --type drop` | `cilium policy trace --src-k8s-pod ... --dst-k8s-pod ...` | `cilium bpf policy get <EP_ID>` |
| 서비스 접근 불가 | `cilium service list` | `cilium bpf lb list` | `cilium bpf ct list global` |
| DNS 해석 실패 | `cilium monitor --type l7` | `cilium fqdn cache list` | `tcpdump -i any port 53` |
| 외부 연결 실패 | `cilium bpf nat list` | `cilium monitor --type drop` | `tcpdump -i any host <EXT_IP>` |
| 엔드포인트 not-ready | `cilium endpoint list` | `cilium endpoint log <EP_ID>` | `kubectl logs <CILIUM_POD>` |
| 아이덴티티 미할당 | `cilium bpf ipcache list` | `cilium identity list` | `cilium status --verbose` |
| 높은 패킷 드롭률 | `cilium_drop_count_total` 메트릭 | `cilium monitor --type drop` | `cilium debuginfo` |
| 에이전트 비정상 | `cilium status` | `cilium status --verbose` | `cilium-bugtool` |
| 정책 미적용 | `cilium policy get` | `cilium endpoint get <EP_ID>` | `cilium policy selectors` |
| BPF 맵 가득 참 | `cilium bpf ct list global \| wc -l` | `cilium status --verbose` | ConfigMap에서 맵 크기 조정 |

### 에스컬레이션(Escalation) 기준

다음 상황에서는 Cilium 커뮤니티 또는 지원팀에 에스컬레이션 권장:

1. **커널 패닉(kernel panic)** 또는 **노드 행(hang)**: eBPF 관련 커널 버그 가능성. `dmesg` 및 `cilium-bugtool` 출력 첨부 필수
2. **BPF 검증기(verifier) 오류**: 커널 호환성 문제. `cilium bpf prog list` 및 커널 버전 정보 제공
3. **대규모 클러스터에서의 성능 저하**: 아이덴티티 수, CT 맵 크기, 정책 규모에 따른 스케일링 이슈
4. **에이전트 반복 크래시(CrashLoopBackOff)**: 에이전트 로그 및 `cilium-bugtool` 수집 후 GitHub Issue 생성
5. **업그레이드 후 비정상 동작**: 이전/이후 버전, `cilium debuginfo`, 업그레이드 방법(Helm, operator) 정보 제공

**에스컬레이션 시 필수 첨부 자료:**

```bash
# 1. cilium-bugtool 진단 파일
kubectl -n kube-system exec <CILIUM_POD_NAME> -- cilium-bugtool
kubectl cp kube-system/<CILIUM_POD_NAME>:/tmp/cilium-bugtool-<TIMESTAMP>.tar ./bugtool.tar

# 2. 에이전트 로그
kubectl -n kube-system logs <CILIUM_POD_NAME> --since=2h > agent.log

# 3. 클러스터 정보
kubectl version
cilium version
kubectl get nodes -o wide
cilium status --verbose
```

**커뮤니티 지원 채널:**

- GitHub Issues: [https://github.com/cilium/cilium/issues](https://github.com/cilium/cilium/issues)
- Slack: [https://cilium.herokuapp.com/](https://cilium.herokuapp.com/) (`#troubleshooting` 채널)
