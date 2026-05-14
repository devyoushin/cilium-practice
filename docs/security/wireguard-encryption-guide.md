# WireGuard 투명 암호화 가이드

## 1. 개요

### 1.1 WireGuard 투명 암호화란

Cilium의 WireGuard 투명 암호화(Transparent Encryption)는 노드 간(node-to-node) 모든 파드 트래픽을 커널 레벨(kernel-level)에서 자동으로 암호화하는 기능. 애플리케이션 코드나 설정 변경 없이, Cilium이 WireGuard 터널을 자동 구성하여 클러스터 내부 통신의 기밀성(confidentiality)을 보장.

### 1.2 필요성

- 클러스터 내부 트래픽은 기본적으로 평문(plaintext)으로 전송되며, 같은 네트워크에 접근 가능한 공격자가 패킷을 스니핑(sniffing)할 수 있음
- 컴플라이언스(compliance) 요건 (PCI-DSS, HIPAA 등)에서 전송 중 데이터 암호화(encryption in transit)를 요구하는 경우가 다수
- 애플리케이션마다 mTLS를 설정하는 것은 운영 부담이 큼. WireGuard는 인프라 레벨에서 일괄 적용 가능

### 1.3 WireGuard vs IPsec 비교

| 항목 | WireGuard | IPsec |
|------|-----------|-------|
| 구현 방식 | 커널 모듈(kernel module), 간결한 코드베이스 | 커널 내장(built-in), 복잡한 프로토콜 스택 |
| 성능 | 높음 (ChaCha20-Poly1305, 단일 라운드트립) | 중간 (AES-GCM, 핸드셰이크 오버헤드 존재) |
| 키 관리 | Cilium이 자동 관리 (노드 단위 키 페어) | Cilium이 자동 관리 (PSK 또는 인증서 기반) |
| 설정 복잡도 | 낮음 (Helm 값 2줄) | 중간 (추가 설정 필요) |
| 커널 요구사항 | Linux 5.6+ (또는 백포트된 커널) | 대부분의 Linux 커널에서 지원 |
| 암호화 대상 | 노드 간 파드 트래픽 전체 | 노드 간 파드 트래픽 전체 |
| FIPS 140-2 준수 | 미지원 | 지원 (AES-GCM 사용 시) |

> FIPS 준수가 필수 요건인 환경에서는 IPsec를 선택해야 함. 그 외 대부분의 경우 WireGuard가 더 단순하고 높은 성능을 제공.

---

## 2. 설명

### 2.1 핵심 개념

#### WireGuard의 동작 방식

Cilium에서 WireGuard를 활성화하면 다음과 같은 흐름으로 동작:

1. 각 노드에서 Cilium 에이전트(agent)가 WireGuard 인터페이스(`cilium_wg0`)를 생성
2. 노드마다 고유한 공개키/비밀키 쌍(public/private key pair)을 자동 생성
3. 각 노드의 공개키를 `CiliumNode` 커스텀 리소스(Custom Resource)에 저장
4. 다른 노드의 공개키를 참조하여 피어(peer) 설정을 자동 구성
5. 노드 간 파드 트래픽이 `cilium_wg0` 인터페이스를 통과하며 자동 암호화/복호화

```
[Pod A on Node 1] --> [cilium_wg0 (암호화)] --> [네트워크] --> [cilium_wg0 (복호화)] --> [Pod B on Node 2]
```

#### 키 관리 (Key Management)

- Cilium 에이전트가 노드별 키 페어를 자동 생성 및 관리
- 키는 에이전트 재시작 시 자동 재생성 (기본 동작)
- `CiliumNode` 리소스의 어노테이션(annotation)에 공개키가 저장됨
- 키 로테이션(key rotation)은 에이전트 재시작 시 자동 수행

#### 암호화 대상 트래픽

| 트래픽 유형 | 암호화 여부 |
|-------------|-------------|
| 파드-파드 (다른 노드) | 암호화됨 |
| 파드-파드 (같은 노드) | 암호화되지 않음 (로컬 통신) |
| 파드-서비스 (다른 노드) | 암호화됨 |
| 노드-노드 (호스트 네트워크) | `encryption.nodeEncryption: true` 설정 시 암호화됨 |
| 파드-외부 | 암호화되지 않음 |

#### 성능 영향 (Performance Impact)

일반적인 벤치마크 결과 (참고용, 환경에 따라 상이):

| 구성 | 처리량 (Throughput) | 지연시간 (Latency) |
|------|---------------------|--------------------|
| 암호화 없음 | 기준값 (baseline) | 기준값 |
| WireGuard | 기준 대비 약 5~15% 감소 | 약 10~20us 증가 |
| IPsec (AES-GCM) | 기준 대비 약 15~30% 감소 | 약 20~40us 증가 |

> 실제 수치는 인스턴스 타입, 네트워크 구성, 트래픽 패턴에 따라 달라짐. 프로덕션 적용 전 반드시 자체 벤치마크 수행 권장.

---

### 2.2 실무 적용 코드

#### Helm Values 설정

신규 설치 시 WireGuard 암호화를 활성화하는 `values-wireguard.yaml`:

```yaml
# values-wireguard.yaml
encryption:
  enabled: true
  type: wireguard
  # 호스트 네트워크 트래픽도 암호화할 경우 활성화
  # nodeEncryption: true

# EKS ENI 모드 기본 설정 (기존 설정 유지)
eni:
  enabled: true

ipam:
  mode: eni

egressMasqueradeInterfaces: eth0

routingMode: native

kubeProxyReplacement: true
```

#### 신규 클러스터에 Cilium + WireGuard 설치

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm install cilium cilium/cilium \
  --version 1.16.5 \
  --namespace kube-system \
  --values values-wireguard.yaml \
  --set cluster.name=<CLUSTER_NAME> \
  --set eni.enabled=true \
  --set ipam.mode=eni \
  --set egressMasqueradeInterfaces=eth0 \
  --set routingMode=native \
  --set encryption.enabled=true \
  --set encryption.type=wireguard
```

#### 기존 클러스터에 WireGuard 활성화 (Helm Upgrade)

```bash
# 현재 설정 확인
helm get values cilium -n kube-system -o yaml > current-values.yaml

# WireGuard 활성화 적용
helm upgrade cilium cilium/cilium \
  --version 1.16.5 \
  --namespace kube-system \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard

# Cilium 에이전트 파드가 순차적으로 재시작되는지 확인
kubectl -n kube-system rollout status daemonset/cilium
```

> 기존 클러스터에 적용 시, Cilium 에이전트가 롤링 재시작(rolling restart)됨. 일시적으로 네트워크 연결이 끊길 수 있으므로 유지보수 윈도우(maintenance window)에서 수행 권장.

#### 암호화 상태 확인

```bash
# Cilium CLI를 통한 암호화 상태 확인
cilium encrypt status

# 예상 출력:
# Encryption: WireGuard
# Keys in use: 1
# Max Seq. Number: N/A
# Errors: 0
```

#### 노드 키 확인

```bash
# 각 노드의 WireGuard 공개키 확인
kubectl get ciliumnodes -o yaml | grep -A 2 wireguard

# 특정 노드의 공개키 확인
kubectl get ciliumnode <NODE_NAME> -o jsonpath='{.metadata.annotations.network\.cilium\.io/wg-pub-key}'

# Cilium 에이전트 파드에서 WireGuard 인터페이스 확인
kubectl -n kube-system exec -it <CILIUM_POD_NAME> -- cilium encrypt status
```

#### WireGuard 인터페이스 직접 확인

```bash
# 노드에 SSH 접근 후 WireGuard 인터페이스 상태 확인
# (EKS 노드에 SSM 또는 SSH로 접근)
ip link show cilium_wg0

# WireGuard 피어 목록 및 핸드셰이크 상태 확인
wg show cilium_wg0

# 예상 출력:
# interface: cilium_wg0
#   public key: <BASE64_PUBLIC_KEY>
#   private key: (hidden)
#   listening port: 51871
#
# peer: <PEER_PUBLIC_KEY>
#   endpoint: <PEER_NODE_IP>:51871
#   allowed ips: <POD_CIDR>
#   latest handshake: 42 seconds ago
#   transfer: 1.23 MiB received, 4.56 MiB sent
```

---

### 2.3 보안/성능 Best Practice

#### 프로덕션 환경에서의 권장 사항

1. **프로덕션 환경에서 반드시 암호화 활성화**
   - 내부 트래픽도 암호화하는 것이 제로 트러스트(Zero Trust) 원칙에 부합
   - 클라우드 환경에서 VPC 내부 트래픽도 암호화되지 않으면 동일 네트워크 내 다른 테넌트가 접근 가능

2. **암호화 오버헤드 모니터링**
   - CPU 사용량 증가 모니터링 (특히 Cilium 에이전트 파드)
   - 네트워크 지연시간 변화 추적
   - 처리량 감소 여부 확인

   ```bash
   # 암호화 전후 CPU 사용량 비교
   kubectl top pods -n kube-system -l k8s-app=cilium
   ```

3. **CiliumNetworkPolicy와 조합하여 심층 방어(Defense-in-Depth) 구현**
   - WireGuard는 전송 중 암호화를 담당
   - CiliumNetworkPolicy는 트래픽 허용/차단을 담당
   - 두 기능을 함께 사용하여 보안 레이어를 다중화

   ```yaml
   # 예시: WireGuard + CiliumNetworkPolicy 조합
   apiVersion: cilium.io/v2
   kind: CiliumNetworkPolicy
   metadata:
     name: restrict-backend-access
     namespace: <APP_NAMESPACE>
   spec:
     endpointSelector:
       matchLabels:
         app: backend
     ingress:
       - fromEndpoints:
           - matchLabels:
               app: frontend
         toPorts:
           - ports:
               - port: "8080"
                 protocol: TCP
   ```

4. **키 로테이션 고려사항**
   - Cilium 에이전트 재시작 시 키가 자동 재생성됨
   - 정기적인 에이전트 롤링 재시작으로 키 로테이션 수행 가능
   - 키 로테이션 주기는 조직의 보안 정책에 따라 결정

   ```bash
   # 수동 키 로테이션 (Cilium 에이전트 롤링 재시작)
   kubectl -n kube-system rollout restart daemonset/cilium

   # 재시작 완료 대기
   kubectl -n kube-system rollout status daemonset/cilium
   ```

5. **노드 암호화(Node Encryption) 활성화 검토**
   - 호스트 네트워크(host networking)를 사용하는 파드가 있다면 `nodeEncryption` 활성화 고려
   - 단, 추가적인 성능 오버헤드 발생 가능

   ```yaml
   encryption:
     enabled: true
     type: wireguard
     nodeEncryption: true
   ```

---

## 3. 트러블슈팅

### 3.1 주요 이슈

#### 이슈 1: WireGuard가 트래픽을 암호화하지 않음 (커널 모듈 미로드)

**증상:**
- `cilium encrypt status` 명령 시 `Encryption: Disabled` 또는 오류 출력
- 파드 간 통신은 정상이지만 암호화되지 않음

**원인:**
- WireGuard 커널 모듈이 노드에 로드되지 않음
- EKS AMI 버전이 WireGuard를 지원하지 않는 커널 사용

**진단:**

```bash
# 노드에서 WireGuard 모듈 로드 여부 확인
# (SSM 또는 SSH로 노드 접근 후)
lsmod | grep wireguard

# 커널 버전 확인 (5.6+ 필요)
uname -r

# Cilium 에이전트 로그에서 WireGuard 관련 오류 확인
kubectl -n kube-system logs <CILIUM_POD_NAME> | grep -i wireguard
```

**해결:**

```bash
# EKS 최적화 AMI 사용 시, Amazon Linux 2에서 WireGuard 모듈 설치
# (노드 UserData 또는 별도 DaemonSet으로 실행)
yum install -y kernel-modules-extra
modprobe wireguard

# 또는 Amazon Linux 2023 / Bottlerocket AMI 사용 권장 (WireGuard 기본 포함)
# EKS 관리형 노드 그룹의 AMI를 AL2023으로 변경
```

---

#### 이슈 2: WireGuard 활성화 후 노드 간 연결 장애 (Cross-node Connectivity Failure)

**증상:**
- WireGuard 활성화 후 다른 노드의 파드와 통신 불가
- 같은 노드 내 파드 간 통신은 정상

**원인:**
- WireGuard 포트(UDP 51871)가 보안 그룹(Security Group)에서 차단됨
- 노드 간 MTU 불일치

**진단:**

```bash
# Cilium 에이전트 로그 확인
kubectl -n kube-system logs <CILIUM_POD_NAME> | grep -i "wireguard\|wg0\|handshake"

# WireGuard 핸드셰이크 상태 확인 (노드 접근 필요)
wg show cilium_wg0

# 핸드셰이크가 수행되지 않았다면 ("latest handshake: (none)") 포트 차단 의심

# 연결 테스트
kubectl exec -it <TEST_POD> -- ping <REMOTE_POD_IP>
```

**해결:**

```bash
# AWS 보안 그룹에 WireGuard 포트 허용 추가
# EKS 노드 보안 그룹에서 UDP 51871 포트를 노드 간 허용

aws ec2 authorize-security-group-ingress \
  --group-id <NODE_SECURITY_GROUP_ID> \
  --protocol udp \
  --port 51871 \
  --source-group <NODE_SECURITY_GROUP_ID>
```

```yaml
# Terraform 사용 시 보안 그룹 규칙 예시
resource "aws_security_group_rule" "wireguard" {
  type                     = "ingress"
  from_port                = 51871
  to_port                  = 51871
  protocol                 = "udp"
  security_group_id        = <NODE_SECURITY_GROUP_ID>
  source_security_group_id = <NODE_SECURITY_GROUP_ID>
  description              = "Cilium WireGuard encryption"
}
```

---

#### 이슈 3: 암호화 활성화 후 성능 저하 (Performance Degradation)

**증상:**
- WireGuard 활성화 후 네트워크 처리량(throughput) 현저히 감소
- CPU 사용량 급증

**원인:**
- 인스턴스 타입의 CPU가 부족하여 암호화/복호화 처리에 병목 발생
- MTU 미조정으로 인한 패킷 분할(fragmentation) 발생

**진단:**

```bash
# CPU 사용량 확인
kubectl top pods -n kube-system -l k8s-app=cilium

# MTU 확인
kubectl -n kube-system exec -it <CILIUM_POD_NAME> -- ip link show cilium_wg0

# iperf3로 처리량 측정 (별도 테스트 파드 필요)
# 서버 측
kubectl exec -it <IPERF_SERVER_POD> -- iperf3 -s

# 클라이언트 측 (다른 노드에서)
kubectl exec -it <IPERF_CLIENT_POD> -- iperf3 -c <IPERF_SERVER_POD_IP> -t 30
```

**해결:**

```yaml
# MTU 조정 (WireGuard 오버헤드 60바이트 고려)
# values-wireguard.yaml에 추가
MTU: 1400  # 기본 1500에서 WireGuard 오버헤드를 뺀 값

# 또는 Helm 설정
# --set MTU=1400
```

```bash
# 더 높은 CPU를 가진 인스턴스 타입으로 노드 그룹 변경 검토
# c5/c6i 계열은 AES-NI 지원으로 암호화 성능 우수
```

---

#### 이슈 4: 키 로테이션 실패 (Key Rotation Failure)

**증상:**
- 에이전트 재시작 후 일부 노드 간 통신 불가
- `CiliumNode` 리소스의 공개키가 업데이트되지 않음

**원인:**
- `CiliumNode` 리소스에 대한 RBAC 권한 부족
- 에이전트가 비정상 종료되어 키 업데이트가 불완전

**진단:**

```bash
# CiliumNode 리소스에서 키 정보 확인
kubectl get ciliumnodes -o wide

# 특정 노드의 키 어노테이션 확인
kubectl get ciliumnode <NODE_NAME> -o yaml | grep wg-pub-key

# Cilium 에이전트 로그에서 키 관련 오류 확인
kubectl -n kube-system logs <CILIUM_POD_NAME> | grep -i "key\|rotate\|wireguard"

# RBAC 확인
kubectl auth can-i update ciliumnodes --as=system:serviceaccount:kube-system:cilium
```

**해결:**

```bash
# Cilium 에이전트 롤링 재시작으로 키 재생성 강제 수행
kubectl -n kube-system rollout restart daemonset/cilium
kubectl -n kube-system rollout status daemonset/cilium

# 특정 노드만 문제인 경우 해당 노드의 Cilium 파드만 삭제
kubectl -n kube-system delete pod <CILIUM_POD_NAME>

# CiliumNode 리소스 키 정보 수동 정리 (최후의 수단)
kubectl annotate ciliumnode <NODE_NAME> network.cilium.io/wg-pub-key-
# 이후 에이전트가 자동으로 새 키를 생성하여 어노테이션에 등록
```

---

### 3.2 자주 발생하는 문제 (Q&A)

**Q1: WireGuard를 활성화하면 기존 연결(existing connection)이 끊기는가?**

A: Cilium 에이전트가 롤링 재시작되므로 일시적인 연결 중단이 발생할 수 있음. 그러나 대부분의 경우 수 초 내에 복구됨. 중요한 워크로드가 있다면 유지보수 윈도우에서 수행하고, PodDisruptionBudget(PDB)을 설정하여 영향을 최소화할 것.

**Q2: 같은 노드 내 파드 간 통신도 암호화되는가?**

A: 기본적으로 같은 노드 내 파드 간 통신은 암호화되지 않음. 로컬 통신은 커널 내부에서 처리되므로 네트워크를 통과하지 않아 스니핑 위험이 낮음. 같은 노드 내 통신도 암호화가 필요하다면 서비스 메시(service mesh) 도입을 검토할 것.

**Q3: WireGuard와 Cilium의 ENI 모드는 호환되는가?**

A: 호환됨. ENI 모드에서도 WireGuard 투명 암호화가 정상 동작. 단, ENI 모드에서는 `routingMode: native`가 일반적이며, 이 환경에서 WireGuard가 노드 간 파드 트래픽을 암호화하는 데 문제가 없음.

**Q4: EKS Fargate 파드에서도 WireGuard 암호화가 적용되는가?**

A: 적용되지 않음. Fargate 파드는 별도의 가상 머신에서 실행되며 Cilium 에이전트가 설치되지 않으므로 WireGuard 암호화 대상에 포함되지 않음. WireGuard 암호화는 Cilium 에이전트가 실행되는 관리형 노드 그룹(managed node group) 또는 자체 관리 노드(self-managed node)에서만 동작.

**Q5: WireGuard 활성화 시 추가 비용이 발생하는가?**

A: WireGuard 자체는 추가 비용이 없음. 그러나 암호화/복호화 처리로 인해 CPU 사용량이 증가하므로, 필요에 따라 더 높은 사양의 인스턴스 타입을 사용해야 할 수 있으며 이로 인해 간접적인 비용 증가 가능성이 있음.

---

## 4. 모니터링 및 확인

### 4.1 cilium encrypt status 명령

```bash
# Cilium CLI 사용
cilium encrypt status

# 예상 출력 (정상):
# Encryption: WireGuard
#   Listening Port: 51871
#   Interface Count: 1
#   Peer Count: <NUMBER_OF_OTHER_NODES>
#   Keys in use: 1

# Cilium 에이전트 파드 내부에서 실행
kubectl -n kube-system exec -it <CILIUM_POD_NAME> -- cilium encrypt status
```

### 4.2 WireGuard 인터페이스 확인

```bash
# cilium_wg0 인터페이스 존재 여부 확인
kubectl -n kube-system exec -it <CILIUM_POD_NAME> -- ip link show cilium_wg0

# 인터페이스 통계(statistics) 확인
kubectl -n kube-system exec -it <CILIUM_POD_NAME> -- ip -s link show cilium_wg0

# 예상 출력:
# 5: cilium_wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 ...
#     link/none
#     RX: bytes  packets  errors  dropped ...
#     TX: bytes  packets  errors  dropped ...
```

### 4.3 Prometheus 메트릭

Cilium이 노출하는 WireGuard 관련 Prometheus 메트릭:

```yaml
# Cilium 에이전트의 Prometheus 메트릭 엔드포인트에서 확인 가능
# 기본 포트: 9962

# 주요 메트릭:
# cilium_wireguard_peers          - WireGuard 피어 수
# cilium_wireguard_bytes_sent     - 전송된 암호화 바이트
# cilium_wireguard_bytes_received - 수신된 암호화 바이트
```

```yaml
# Prometheus ServiceMonitor 예시 (kube-prometheus-stack 사용 시)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cilium-metrics
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: cilium
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

```promql
# Grafana에서 활용 가능한 PromQL 쿼리 예시

# 노드별 WireGuard 피어 수
cilium_wireguard_peers

# WireGuard를 통한 초당 전송 바이트 (전체 클러스터)
sum(rate(cilium_wireguard_bytes_sent[5m]))

# WireGuard를 통한 초당 수신 바이트 (노드별)
rate(cilium_wireguard_bytes_received[5m])

# 암호화된 트래픽 비율 확인 (암호화된 바이트 / 전체 바이트)
sum(rate(cilium_wireguard_bytes_sent[5m])) / sum(rate(cilium_datapath_sent_bytes_total[5m]))
```

### 4.4 Hubble Flow 관찰

```bash
# Hubble CLI로 암호화된 플로우 관찰
hubble observe --protocol tcp --to-namespace <TARGET_NAMESPACE>

# 특정 파드 간 통신 관찰
hubble observe --from-pod <SOURCE_NAMESPACE>/<SOURCE_POD> \
               --to-pod <DEST_NAMESPACE>/<DEST_POD>

# Hubble UI에서 확인
# Hubble UI 포트포워딩
kubectl -n kube-system port-forward svc/hubble-ui 12000:80

# 브라우저에서 http://localhost:12000 접속 후
# 네임스페이스별 트래픽 플로우 시각화 확인
```

```bash
# Hubble에서 암호화 상태를 포함한 상세 플로우 확인
hubble observe -o json | jq '.flow | {
  src: .source.namespace + "/" + .source.pod_name,
  dst: .destination.namespace + "/" + .destination.pod_name,
  encrypted: .is_reply,
  verdict: .verdict
}'
```

---

## 5. TIP

### 공식 문서 참고 링크

- Cilium WireGuard 공식 문서: https://docs.cilium.io/en/v1.16/security/network/encryption-wireguard/
- Cilium 암호화 개요: https://docs.cilium.io/en/v1.16/security/network/encryption/
- WireGuard 프로젝트 공식 사이트: https://www.wireguard.com/

### IPsec을 대신 사용해야 하는 경우

다음 조건에 해당하면 WireGuard 대신 IPsec 사용을 권장:

1. **FIPS 140-2 준수가 필수인 환경** - WireGuard의 ChaCha20-Poly1305는 FIPS 인증을 받지 않았으므로, FIPS 준수가 요구되는 정부/금융 환경에서는 AES-GCM 기반의 IPsec을 사용해야 함
2. **Linux 커널 5.6 미만을 사용하는 노드** - WireGuard 커널 모듈이 포함되지 않은 구형 커널 환경에서는 IPsec이 더 안정적
3. **기존에 IPsec 기반 VPN 인프라와 통합이 필요한 경우** - 기존 IPsec 정책과의 일관성 유지가 중요한 환경

### 추가 팁

- WireGuard 활성화 전 테스트 환경에서 충분히 검증한 후 프로덕션에 적용할 것
- Cilium 버전 업그레이드 시 WireGuard 관련 릴리스 노트를 반드시 확인할 것
- `cilium connectivity test` 명령으로 암호화 활성화 후 전체 연결성 테스트 수행 권장

```bash
# Cilium 연결성 테스트 (암호화 포함)
cilium connectivity test

# 특정 테스트만 실행
cilium connectivity test --test encryption
```
