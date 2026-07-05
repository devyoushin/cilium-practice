# ENI 모드 심화 가이드

Cilium의 ENI (Elastic Network Interface) 모드는 AWS VPC 네이티브 네트워킹을 활용하여
파드에 **실제 VPC IP 주소**를 할당하는 IPAM 모드.
오버레이 없이 AWS 네트워크와 직접 통합되므로 성능과 호환성 측면에서 유리한 선택.

---

## 1. 개요

### ENI 모드란

```
기존 오버레이 방식:
  파드 (10.244.x.x) → VXLAN 캡슐화 → 노드 (10.0.1.x) → AWS VPC
  → 파드 IP가 VPC에서 라우팅 불가
  → 오버레이 오버헤드 존재

ENI 모드:
  파드 (10.0.1.50) → 직접 → AWS VPC
  → 파드 IP = VPC 서브넷의 실제 IP
  → 오버레이 없음, 네이티브 성능
```

ENI 모드에서 Cilium Operator (Cilium 오퍼레이터)는 AWS EC2 API를 호출하여
각 노드에 ENI를 연결하고, ENI의 보조 IP (Secondary IP)를 파드에 할당.

### EKS에서 ENI 모드를 사용하는 이유

- **네이티브 VPC 통합**: 파드 IP가 VPC CIDR 범위에 속하므로 Security Group (보안 그룹), NACL, VPC Flow Logs 등과 자연스럽게 연동
- **오버레이 제거**: VXLAN/Geneve 캡슐화 없이 직접 라우팅되므로 네트워크 지연 (Latency) 감소
- **AWS 서비스 호환**: RDS, ElastiCache 등 VPC 내부 서비스에 파드가 직접 접근 가능
- **Cilium 고급 기능 활용**: aws-node (VPC CNI)를 대체하면서 Cilium의 NetworkPolicy, Hubble 관측성 등을 함께 사용 가능

### 모드 비교

| 항목 | ENI 모드 | Overlay (VXLAN/Geneve) | Direct Routing |
|------|----------|----------------------|----------------|
| 파드 IP | VPC 서브넷의 실제 IP | 클러스터 내부 가상 IP | VPC 서브넷의 실제 IP |
| 캡슐화 | 없음 | VXLAN 또는 Geneve 헤더 추가 | 없음 |
| 성능 | 네이티브 수준 | 캡슐화 오버헤드 존재 | 네이티브 수준 |
| IP 소비 | VPC 서브넷 IP 소비 | 최소 (클러스터 내부 CIDR 사용) | VPC 서브넷 IP 소비 |
| AWS 서비스 연동 | 직접 통신 가능 | NAT 또는 추가 설정 필요 | 라우팅 설정 필요 |
| 확장성 제약 | ENI 및 IP 수 제한 | 거의 무제한 | 라우팅 테이블 크기 제한 |
| 설정 복잡도 | IAM, 서브넷 태깅 필요 | 비교적 단순 | BGP 또는 라우팅 설정 필요 |

---

## 2. 설명

### 2.1 핵심 개념

#### ENI IPAM 동작 원리

```
┌─────────────────────────────────────────────────────────┐
│  Cilium Operator                                        │
│  ┌──────────────────────────────────┐                   │
│  │  IPAM Controller                 │                   │
│  │  1. 노드 감시 (Watch Nodes)      │                   │
│  │  2. IP 부족 감지                  │                   │
│  │  3. AWS EC2 API 호출             │                   │
│  │     → CreateNetworkInterface     │                   │
│  │     → AttachNetworkInterface     │                   │
│  │     → AssignPrivateIpAddresses   │                   │
│  │  4. CiliumNode 리소스 업데이트    │                   │
│  └──────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  Worker Node                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  eth0     │  │  eth1     │  │  eth2     │             │
│  │ (Primary) │  │ (ENI #2)  │  │ (ENI #3)  │             │
│  │ 10.0.1.10 │  │ 10.0.2.20 │  │ 10.0.3.30 │             │
│  │           │  │ +10.0.2.21│  │ +10.0.3.31│             │
│  │           │  │ +10.0.2.22│  │ +10.0.3.32│             │
│  │           │  │ +10.0.2.23│  │ +10.0.3.33│             │
│  └──────────┘  └──────────┘  └──────────┘              │
│       │              │              │                    │
│       ▼              ▼              ▼                    │
│   노드 통신      파드 IP 할당    파드 IP 할당            │
└─────────────────────────────────────────────────────────┘
```

**동작 순서:**

1. Cilium Operator가 각 노드의 IP 사용량을 감시
2. 가용 IP가 사전 할당 임계값 (Pre-Allocation Threshold) 이하로 떨어지면 새 ENI 생성 또는 기존 ENI에 보조 IP 추가
3. 할당된 IP가 CiliumNode 커스텀 리소스에 기록
4. Cilium Agent가 파드 생성 시 CiliumNode의 가용 IP 풀에서 IP를 꺼내 파드에 할당
5. 파드가 삭제되면 IP가 풀로 반환

#### IP 사전 할당 및 웜 풀 (Warm Pool)

```
Pre-Allocation 동작 원리:

  설정: pre-allocate = 8 (기본값)
  min-allocate = 0

  시나리오:
    현재 할당된 IP: 15개
    사용 중인 IP:   12개
    남은 IP:        3개  ← pre-allocate(8)보다 적음!

    → Operator가 추가 IP 확보 요청
    → 목표: 사용 중(12) + pre-allocate(8) = 20개 확보
    → 5개 IP 추가 할당
```

#### 인스턴스 유형별 ENI 및 IP 제한

| 인스턴스 유형 (Instance Type) | 최대 ENI 수 | ENI당 최대 IPv4 수 | 최대 파드 IP 수 (Primary IP 제외) |
|------|---------|-------------|------|
| m5.large | 3 | 10 | 27 (3 × 10 - 3) |
| m5.xlarge | 4 | 15 | 56 (4 × 15 - 4) |
| m5.2xlarge | 4 | 15 | 56 (4 × 15 - 4) |
| c5.xlarge | 4 | 15 | 56 (4 × 15 - 4) |
| r5.large | 3 | 10 | 27 (3 × 10 - 3) |

> **참고**: 각 ENI의 Primary IP는 ENI 자체에 사용되므로 파드에 할당 불가.
> Prefix Delegation (접두사 위임) 활성화 시 ENI당 IP 수가 크게 증가.

#### 파드 IP 주소 할당

```
파드 IP = VPC 서브넷의 실제 Private IP

예시:
  VPC CIDR: 10.0.0.0/16
  Pod 서브넷: 10.0.64.0/18 (16,384개 IP)

  Node A (10.0.1.10):
    eth1 → 10.0.64.11 → Pod-1
    eth1 → 10.0.64.12 → Pod-2
    eth2 → 10.0.65.20 → Pod-3

  Node B (10.0.2.20):
    eth1 → 10.0.64.50 → Pod-4
    eth1 → 10.0.64.51 → Pod-5
```

#### 서브넷 및 보안 그룹 고려 사항

- **서브넷 분리**: 노드용 서브넷과 파드용 서브넷을 분리하여 IP 관리 효율화
- **서브넷 태깅**: Cilium Operator가 파드용 서브넷을 식별할 수 있도록 태그 필요
- **보안 그룹**: ENI에 연결되는 보안 그룹이 파드 트래픽을 허용해야 함
- **AZ 배치**: ENI는 노드와 동일한 가용 영역 (Availability Zone)의 서브넷에서만 생성 가능

#### aws-node (VPC CNI) 제거 프로세스

```bash
# 1. aws-node DaemonSet 확인
kubectl get ds -n kube-system aws-node

# 2. aws-node 비활성화 (삭제 전 스케일 다운)
kubectl -n kube-system patch daemonset aws-node \
  --type='strategic' \
  -p='{"spec":{"template":{"spec":{"nodeSelector":{"io.cilium/aws-node-enabled":"true"}}}}}'

# 3. Cilium 설치 후 aws-node 완전 삭제
kubectl delete ds -n kube-system aws-node

# 4. 기존 ENI 정리 (선택적, 노드 재시작 후 자동 정리)
# aws-node이 생성한 ENI는 Cilium이 관리하지 않으므로 수동 확인 필요
aws ec2 describe-network-interfaces \
  --filters "Name=tag:node.k8s.amazonaws.com/instance_id,Values=<INSTANCE_ID>" \
  --query "NetworkInterfaces[*].{ID:NetworkInterfaceId,Status:Status}"
```

> **주의**: 프로덕션 환경에서는 노드를 순차적으로 교체 (Rolling Replace)하는 방식이 안전.
> aws-node 삭제 후 Cilium이 IPAM을 인수하기까지 일시적으로 파드 네트워크가 중단될 수 있음.

---

### 2.2 실무 적용 코드 (YAML)

#### 기본 ENI 모드 Helm 값

```yaml
# helm/values-eni.yaml
eni:
  enabled: true
  awsEnablePrefixDelegation: false    # /28 접두사 위임 비활성화 (기본)
  awsReleaseExcessIPs: true           # 초과 IP 반환 활성화
  subnetIDsFilter: []                 # 특정 서브넷 ID로 필터링
  subnetTagsFilter: {}                # 서브넷 태그로 필터링
  instanceTagFilter: {}               # 인스턴스 태그로 필터링
  updateEC2AdapterLimitViaAPI: true   # EC2 API로 인스턴스 ENI 제한 조회

ipam:
  mode: eni
  operator:
    clusterPoolIPv4PodCIDRList: []    # ENI 모드에서는 사용하지 않음

# aws-node 비활성화 (Cilium이 CNI를 대체)
cni:
  chainingMode: "none"
  exclusive: true

# EKS 환경 필수 설정
routingMode: native
enableIPv4Masquerade: true
endpointRoutes:
  enabled: true
```

#### ENI 관련 Helm 값 상세 설명

```yaml
eni:
  # 서브넷 태그 필터 — Operator가 이 태그를 가진 서브넷에서만 ENI를 생성
  subnetTagsFilter:
    "cilium.io/pod-subnet": "true"
    "kubernetes.io/cluster/<CLUSTER_NAME>": "shared"

  # 인스턴스 태그 필터 — 특정 태그가 있는 인스턴스에만 ENI 할당
  instanceTagFilter:
    "cilium-managed": "true"

  # 초과 IP 반환 — 사용하지 않는 IP를 VPC에 반환
  # 대규모 스케일 다운 시 IP 낭비 방지
  awsReleaseExcessIPs: true

  # EC2 API로 인스턴스 ENI 제한을 동적으로 조회
  # false로 설정 시 내장된 제한 테이블 사용
  updateEC2AdapterLimitViaAPI: true

  # ENI에 연결할 보안 그룹 (미지정 시 노드의 기본 보안 그룹 사용)
  securityGroupTagsFilter:
    "cilium.io/pod-sg": "true"
```

#### IP 사전 할당 튜닝

```yaml
# helm/values-eni-tuning.yaml

# 기본 사전 할당 설정
operator:
  # 노드당 사전 할당할 IP 수 (기본값: 8)
  # 값이 클수록 IP 확보가 빠르지만, VPC IP를 더 많이 소비
  extraArgs:
    - "--aws-instance-limit-mapping=m5.large=3,10"  # 커스텀 ENI 제한 오버라이드

ipam:
  operator:
    # 노드당 최소 할당 IP 수
    clusterPoolIPv4MaskSize: "24"

eni:
  # Prefix Delegation 활성화 — ENI당 /28 접두사 할당
  # 기존: ENI당 보조 IP 개별 할당 (예: m5.large → ENI당 9개)
  # PD 활성화: ENI당 /28 접두사 할당 (예: m5.large → ENI당 9 × 16 = 144개)
  awsEnablePrefixDelegation: true
```

#### Prefix Delegation 효과

```
Prefix Delegation 비활성화 (기본):
  m5.large (3 ENI, ENI당 10 IP):
    → 최대 파드 IP: 27개

Prefix Delegation 활성화:
  m5.large (3 ENI, ENI당 9 슬롯 × /28 접두사):
    → 슬롯당 16개 IP
    → 최대 파드 IP: 3 × 9 × 16 = 432개

  ※ 실제로는 maxPodsPerNode 제한에 의해 제한될 수 있음
```

#### IAM 정책 (Cilium Operator용)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CiliumENIIPAM",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DeleteNetworkInterface",
        "ec2:AttachNetworkInterface",
        "ec2:ModifyNetworkInterfaceAttribute",
        "ec2:AssignPrivateIpAddresses",
        "ec2:UnassignPrivateIpAddresses",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeVpcs",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
# IAM 정책 생성
aws iam create-policy \
  --policy-name CiliumENIPolicy \
  --policy-document file://cilium-eni-policy.json

# IRSA (IAM Roles for Service Accounts) 설정
eksctl create iamserviceaccount \
  --cluster=<CLUSTER_NAME> \
  --namespace=kube-system \
  --name=cilium-operator \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/CiliumENIPolicy \
  --approve \
  --override-existing-serviceaccounts
```

#### Prefix Delegation 활성화 시 추가 IAM 권한

```json
{
  "Sid": "CiliumPrefixDelegation",
  "Effect": "Allow",
  "Action": [
    "ec2:AssignIpv6Addresses",
    "ec2:UnassignIpv6Addresses",
    "ec2:GetIpamPoolAllocations"
  ],
  "Resource": "*"
}
```

#### 서브넷 태깅

```bash
# 파드 전용 서브넷에 태그 추가
# Cilium Operator가 이 태그를 기반으로 ENI를 생성할 서브넷을 결정

aws ec2 create-tags \
  --resources <POD_SUBNET_ID_AZ_A> <POD_SUBNET_ID_AZ_B> <POD_SUBNET_ID_AZ_C> \
  --tags Key=cilium.io/pod-subnet,Value=true \
         Key=kubernetes.io/cluster/<CLUSTER_NAME>,Value=shared

# 태그 확인
aws ec2 describe-subnets \
  --filters "Name=tag:cilium.io/pod-subnet,Values=true" \
  --query "Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,AvailableIPs:AvailableIpAddressCount}" \
  --output table
```

#### Helm 설치 전체 명령어

```bash
# Cilium Helm 차트 추가
helm repo add cilium https://helm.cilium.io/
helm repo update

# ENI 모드로 Cilium 설치
helm install cilium cilium/cilium \
  --version 1.16.x \
  --namespace kube-system \
  --set eni.enabled=true \
  --set ipam.mode=eni \
  --set routingMode=native \
  --set enableIPv4Masquerade=true \
  --set endpointRoutes.enabled=true \
  --set cni.chainingMode=none \
  --set cni.exclusive=true \
  --set eni.awsReleaseExcessIPs=true \
  --set eni.subnetTagsFilter."cilium\.io/pod-subnet"=true \
  --set operator.replicas=2 \
  --set egressMasqueradeInterfaces=eth0

# 설치 확인
cilium status --wait
```

---

### 2.3 보안/성능 Best Practice

#### 1. Prefix Delegation 활용으로 IP 밀도 향상

```yaml
# 대규모 클러스터에서 노드당 파드 수가 많을 때 필수
eni:
  awsEnablePrefixDelegation: true

# kubelet maxPods도 함께 조정
# EKS 관리형 노드 그룹 Launch Template에서 설정
# --max-pods=110 (기본 ENI 모드 제한보다 높게)
```

#### 2. IPAM 가용 IP 모니터링

```bash
# 각 노드의 IP 할당 현황 확인
kubectl get ciliumnodes -o custom-columns=\
NAME:.metadata.name,\
ALLOCATED:.status.ipam.operator-status.eni.allocatedIPs,\
AVAILABLE:.status.ipam.operator-status.eni.availableIPs

# Prometheus Alert Rule 예시
# IP 고갈 사전 경고
```

```yaml
# PrometheusRule 리소스
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cilium-ipam-alerts
  namespace: kube-system
spec:
  groups:
    - name: cilium-ipam
      rules:
        - alert: CiliumIPAMIPsLow
          expr: cilium_ipam_available < 5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "노드 {{ $labels.node }}의 가용 IP가 5개 미만"
            description: "현재 가용 IP: {{ $value }}개. 서브넷 확장 또는 노드 추가 필요."

        - alert: CiliumIPAMIPsExhausted
          expr: cilium_ipam_available == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "노드 {{ $labels.node }}의 IP가 고갈됨"
            description: "파드 스케줄링 불가 상태. 즉시 대응 필요."
```

#### 3. 파드 서브넷과 노드 서브넷 분리

```
권장 VPC 설계:

VPC CIDR: 10.0.0.0/16

노드 서브넷 (작은 CIDR):
  10.0.1.0/24  (AZ-a) → 254개 노드 IP
  10.0.2.0/24  (AZ-b) → 254개 노드 IP
  10.0.3.0/24  (AZ-c) → 254개 노드 IP

파드 서브넷 (큰 CIDR):
  10.0.64.0/18  (AZ-a) → 16,382개 파드 IP
  10.0.128.0/18 (AZ-b) → 16,382개 파드 IP
  10.0.192.0/18 (AZ-c) → 16,382개 파드 IP

→ 파드 서브넷에 cilium.io/pod-subnet=true 태그 부착
```

#### 4. 보안 그룹 설계

```bash
# 파드 전용 보안 그룹 생성
aws ec2 create-security-group \
  --group-name cilium-pod-sg \
  --description "Security group for Cilium ENI pods" \
  --vpc-id <VPC_ID>

# 필수 인그레스 규칙: 클러스터 내부 통신 허용
aws ec2 authorize-security-group-ingress \
  --group-id <POD_SG_ID> \
  --protocol -1 \
  --source-group <POD_SG_ID>

# 노드 → 파드 통신 허용
aws ec2 authorize-security-group-ingress \
  --group-id <POD_SG_ID> \
  --protocol -1 \
  --source-group <NODE_SG_ID>

# 파드 → 외부 이그레스 허용
aws ec2 authorize-security-group-egress \
  --group-id <POD_SG_ID> \
  --protocol -1 \
  --cidr 0.0.0.0/0
```

#### 5. 인스턴스 유형 선택 가이드

```
소규모 클러스터 (파드 50개 미만/노드):
  → m5.large (ENI 3개, 최대 27 파드 IP)
  → 비용 효율적

중규모 클러스터 (파드 50-100개/노드):
  → m5.xlarge (ENI 4개, 최대 56 파드 IP)
  → 또는 Prefix Delegation 활성화 시 m5.large로도 충분

대규모 클러스터 (파드 100개 이상/노드):
  → m5.2xlarge 이상 + Prefix Delegation 필수
  → 또는 Nitro 기반 인스턴스 (c5n, m5n 등) 권장
```

---

## 3. 트러블슈팅

### 3.1 주요 이슈

#### 이슈 1: IP 주소 고갈 (InsufficientFreeAddressesInSubnet)

```bash
# 증상: 파드가 Pending 상태로 남음
kubectl get pods -A | grep Pending

# Cilium Operator 로그 확인
kubectl -n kube-system logs deploy/cilium-operator -c cilium-operator | \
  grep -i "insufficient\|exhausted\|no available"

# 예상 로그:
# level=error msg="Failed to allocate IPs" error="InsufficientFreeAddressesInSubnet"
# level=warning msg="Subnet has insufficient free addresses"
```

```bash
# 원인 확인: 서브넷 가용 IP 수 확인
aws ec2 describe-subnets \
  --subnet-ids <POD_SUBNET_ID> \
  --query "Subnets[*].{ID:SubnetId,Available:AvailableIpAddressCount,CIDR:CidrBlock}" \
  --output table

# 해결 방법 1: VPC에 보조 CIDR 추가 후 새 서브넷 생성
aws ec2 associate-vpc-cidr-block \
  --vpc-id <VPC_ID> \
  --cidr-block 100.64.0.0/16

aws ec2 create-subnet \
  --vpc-id <VPC_ID> \
  --cidr-block 100.64.0.0/18 \
  --availability-zone <AZ>

# 새 서브넷에 태그 추가
aws ec2 create-tags \
  --resources <NEW_SUBNET_ID> \
  --tags Key=cilium.io/pod-subnet,Value=true

# 해결 방법 2: Prefix Delegation 활성화로 IP 효율 향상
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set eni.awsEnablePrefixDelegation=true
```

#### 이슈 2: ENI 연결 실패 (최대 ENI 수 도달)

```bash
# 증상: Operator 로그에서 ENI 생성은 성공하나 연결 실패
kubectl -n kube-system logs deploy/cilium-operator -c cilium-operator | \
  grep -i "AttachNetworkInterface\|ENI limit"

# 예상 로그:
# level=error msg="Failed to attach ENI" error="AttachmentLimitExceeded"

# 원인 확인: 인스턴스에 연결된 ENI 수 확인
aws ec2 describe-instances \
  --instance-ids <INSTANCE_ID> \
  --query "Reservations[].Instances[].{Type:InstanceType,ENIs:NetworkInterfaces[].NetworkInterfaceId}" \
  --output json

# 인스턴스 유형별 최대 ENI 수 확인
aws ec2 describe-instance-types \
  --instance-types <INSTANCE_TYPE> \
  --query "InstanceTypes[*].{Type:InstanceType,MaxENI:NetworkInfo.MaximumNetworkInterfaces,IPv4PerENI:NetworkInfo.Ipv4AddressesPerInterface}" \
  --output table

# 해결: 더 큰 인스턴스 유형으로 교체하거나 Prefix Delegation 활성화
```

#### 이슈 3: IP 미할당으로 파드 스케줄링 실패

```bash
# 증상: 파드 이벤트에서 IP 할당 실패 메시지
kubectl describe pod <POD_NAME> -n <NAMESPACE>
# Events:
#   Warning  FailedCreatePodSandBox  ... Failed to allocate for ENI: no available IP

# CiliumNode 리소스에서 IP 현황 확인
kubectl get ciliumnode <NODE_NAME> -o yaml | \
  grep -A 20 "ipam:"

# Cilium Agent 로그 확인
kubectl -n kube-system exec ds/cilium -- cilium status | grep IPAM

# 해결: Operator의 사전 할당 값 증가
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set operator.extraArgs='{--excess-ip-release-delay=180}'
```

#### 이슈 4: 서브넷 CIDR 부족으로 클러스터 확장 불가

```bash
# 증상: 새 노드 추가 시 해당 AZ의 파드 서브넷에 IP가 부족
# Operator 로그:
# level=error msg="No suitable subnet found" availabilityZone="ap-northeast-2a"

# 현재 서브넷 사용량 확인
aws ec2 describe-subnets \
  --filters "Name=tag:cilium.io/pod-subnet,Values=true" \
  --query "Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Available:AvailableIpAddressCount}" \
  --output table

# 해결: 보조 CIDR (RFC 6598 대역) 활용
# 100.64.0.0/16 대역은 AWS에서 VPC 보조 CIDR로 사용 가능
aws ec2 associate-vpc-cidr-block \
  --vpc-id <VPC_ID> \
  --cidr-block 100.64.0.0/16

# 각 AZ에 새 서브넷 생성
for AZ in a b c; do
  aws ec2 create-subnet \
    --vpc-id <VPC_ID> \
    --cidr-block "100.64.${AZ_INDEX}.0/18" \
    --availability-zone "<REGION>${AZ}" \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=cilium.io/pod-subnet,Value=true}]"
done
```

#### 이슈 5: IAM 권한 오류로 ENI 작업 실패

```bash
# 증상: Operator 로그에서 AccessDenied 또는 UnauthorizedOperation
kubectl -n kube-system logs deploy/cilium-operator -c cilium-operator | \
  grep -i "AccessDenied\|UnauthorizedOperation\|forbidden"

# 예상 로그:
# level=error msg="Failed to create ENI" error="UnauthorizedOperation:
#   You are not authorized to perform this operation"

# IRSA 설정 확인
kubectl get sa cilium-operator -n kube-system -o yaml | grep eks.amazonaws.com

# IAM 역할의 정책 확인
aws iam list-attached-role-policies \
  --role-name <CILIUM_OPERATOR_ROLE_NAME>

aws iam get-policy-version \
  --policy-arn <POLICY_ARN> \
  --version-id v1

# STS 호출로 현재 역할 확인
kubectl -n kube-system exec deploy/cilium-operator -c cilium-operator -- \
  sh -c 'wget -qO- http://169.254.169.254/latest/meta-data/iam/info'

# 해결: IAM 정책에 필요한 EC2 권한 추가 (2.2절의 IAM 정책 참조)
```

#### 이슈 6: ENI 보안 그룹 미스매치

```bash
# 증상: 파드 간 통신은 되나 외부 서비스 접근 불가
# 또는 특정 포트로의 통신 차단

# ENI에 연결된 보안 그룹 확인
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=<INSTANCE_ID>" \
  --query "NetworkInterfaces[*].{ID:NetworkInterfaceId,SG:Groups[*].GroupId,Subnet:SubnetId}" \
  --output json

# 보안 그룹 규칙 확인
aws ec2 describe-security-group-rules \
  --filter "Name=group-id,Values=<SG_ID>" \
  --output table

# 해결: Helm 값에서 보안 그룹 태그 필터 설정
# eni.securityGroupTagsFilter를 통해 파드 ENI에 올바른 보안 그룹 연결
```

---

### 3.2 자주 발생하는 문제 (Q&A)

**Q1. 노드당 최대 파드 수는 몇 개인가?**

```
기본 ENI 모드:
  인스턴스별 (최대 ENI 수 × ENI당 IP 수 - ENI 수) 만큼 가능
  예) m5.xlarge: 4 × 15 - 4 = 56개

Prefix Delegation 활성화 시:
  (최대 ENI 수 × (ENI당 IP 슬롯 - 1) × 16) 만큼 가능
  예) m5.xlarge: 4 × 14 × 16 = 896개 (실질적으로 kubelet maxPods에 의해 제한)

확인 명령:
  kubectl get ciliumnode <NODE_NAME> -o jsonpath='{.spec.ipam.pool}'

  # 또는 인스턴스 유형별 제한 확인
  aws ec2 describe-instance-types \
    --instance-types m5.xlarge \
    --query "InstanceTypes[*].NetworkInfo.{MaxENI:MaximumNetworkInterfaces,IPv4PerENI:Ipv4AddressesPerInterface}" \
    --output table
```

**Q2. IP 용량을 확장하려면 어떻게 해야 하는가?**

```
방법 1: Prefix Delegation 활성화 (가장 간단)
  helm upgrade cilium cilium/cilium \
    --namespace kube-system \
    --reuse-values \
    --set eni.awsEnablePrefixDelegation=true

방법 2: VPC 보조 CIDR 추가 후 서브넷 확장
  aws ec2 associate-vpc-cidr-block --vpc-id <VPC_ID> --cidr-block 100.64.0.0/16
  → 새 서브넷 생성 후 cilium.io/pod-subnet=true 태그 부착

방법 3: 더 큰 인스턴스 유형으로 교체
  m5.large (27 파드 IP) → m5.xlarge (56 파드 IP)

방법 4: 노드 수 증가 (수평 확장)
  → 인스턴스당 파드 수는 줄이되 전체 파드 수 확보
```

**Q3. 여러 서브넷을 사용할 수 있는가?**

```
가능. Cilium Operator는 태그 기반으로 여러 서브넷을 자동 탐색.

설정 방법:
  1. 각 AZ에 파드용 서브넷을 생성
  2. 모든 파드 서브넷에 동일한 태그 부착 (cilium.io/pod-subnet=true)
  3. Helm 값에서 태그 필터 설정:
     eni:
       subnetTagsFilter:
         "cilium.io/pod-subnet": "true"

동작 방식:
  - Operator는 노드의 AZ와 일치하는 서브넷에서 ENI를 생성
  - 하나의 서브넷이 가득 차면 동일 AZ의 다른 서브넷을 자동 사용
  - 서브넷 ID 직접 지정도 가능:
    eni:
      subnetIDsFilter:
        - "subnet-0abc1234"
        - "subnet-0def5678"
```

---

## 4. 모니터링 및 확인

### cilium status (IPAM 섹션)

```bash
# Cilium Agent의 IPAM 상태 확인
kubectl -n kube-system exec ds/cilium -- cilium status

# 출력 예시:
# IPAM:          IPv4: 12/27 allocated from 10.0.64.0/18
# Allocated:     12 IPs
# Available:     15 IPs
# ENIs:          3 attached, 2 in use

# 특정 노드의 Agent에서 상세 IPAM 상태 확인
kubectl -n kube-system exec <CILIUM_POD_NAME> -- cilium status --verbose | \
  grep -A 30 "IPAM"
```

### kubectl get ciliumnodes (노드별 IP 할당 현황)

```bash
# 전체 노드의 IP 할당 요약
kubectl get ciliumnodes

# 특정 노드의 상세 IPAM 정보
kubectl get ciliumnode <NODE_NAME> -o yaml

# 주요 확인 항목:
# spec.ipam.pool — 할당 가능한 IP 풀
# spec.ipam.pre-allocate — 사전 할당 설정
# spec.eni.first-interface-index — ENI 시작 인덱스
# status.ipam.used — 현재 사용 중인 IP와 할당된 파드 정보

# 전체 노드의 IP 사용량 한눈에 보기
kubectl get ciliumnodes -o custom-columns=\
NODE:.metadata.name,\
INSTANCE-TYPE:.spec.eni.instance-type,\
AVAILABLE-ENIS:.spec.eni.availability-zone,\
USED-IPS:.status.ipam.used

# JSON으로 상세 조회
kubectl get ciliumnode <NODE_NAME> -o jsonpath='{.status.ipam}' | jq .
```

### Prometheus IPAM 메트릭

```bash
# 주요 Cilium IPAM 메트릭

# 1. 가용 IP 수 (노드별)
cilium_ipam_available

# 2. IP 할당 요청 총 횟수
cilium_ipam_allocation_ops

# 3. IP 해제 요청 총 횟수
cilium_ipam_release_ops

# 4. IP 부족 해결 시도 횟수
cilium_ipam_deficit_resolver_total

# 5. IP 부족으로 인한 실패 횟수
cilium_ipam_deficit_resolver_unresolvedips_total

# 6. 현재 사용 중인 IP 수
cilium_ipam_used
```

```yaml
# Grafana 대시보드용 PromQL 쿼리 예시

# 노드별 IP 사용률
(cilium_ipam_used / (cilium_ipam_used + cilium_ipam_available)) * 100

# IP 고갈 임박 노드 조회 (가용 IP 5개 미만)
cilium_ipam_available < 5

# IP 할당 실패율
rate(cilium_ipam_deficit_resolver_unresolvedips_total[5m])

# ENI당 할당된 IP 수 추이
sum by (node) (cilium_ipam_used)
```

```bash
# Prometheus 엔드포인트에서 직접 메트릭 확인
kubectl -n kube-system exec ds/cilium -- \
  curl -s http://localhost:9962/metrics | grep cilium_ipam
```

### AWS Console에서 ENI 확인

```bash
# CLI로 Cilium이 생성한 ENI 목록 조회
aws ec2 describe-network-interfaces \
  --filters \
    "Name=tag:io.cilium/eni-type,Values=secondary" \
    "Name=vpc-id,Values=<VPC_ID>" \
  --query "NetworkInterfaces[*].{
    ID:NetworkInterfaceId,
    InstanceId:Attachment.InstanceId,
    SubnetId:SubnetId,
    AZ:AvailabilityZone,
    PrivateIPs:PrivateIpAddresses[*].PrivateIpAddress,
    Status:Status
  }" \
  --output json

# 특정 노드의 ENI 상세 확인
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=<INSTANCE_ID>" \
  --query "NetworkInterfaces[*].{
    ID:NetworkInterfaceId,
    Description:Description,
    SG:Groups[*].GroupName,
    IPCount:PrivateIpAddresses|length(@)
  }" \
  --output table

# ENI 사용량 대시보드 (전체 클러스터)
aws ec2 describe-network-interfaces \
  --filters "Name=tag:io.cilium/eni-type,Values=secondary" \
  --query "length(NetworkInterfaces)"
```

---

## 5. TIP

### 공식 문서 참조

- Cilium ENI IPAM 공식 문서: https://docs.cilium.io/en/v1.16/network/concepts/ipam/eni/
- Cilium AWS ENI Helm 설정: https://docs.cilium.io/en/v1.16/installation/k8s-install-helm/
- AWS ENI 제한 공식 문서: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html

### 클러스터 규모별 인스턴스 유형 추천

```
소규모 (노드 10개 미만, 총 파드 500개 미만):
  → m5.large + 기본 ENI 모드
  → 노드당 최대 27개 파드
  → 서브넷: /24 이상 (AZ당)
  → 예상 비용: ENI 자체는 무료, IP 주소 비용만 발생

중규모 (노드 10-50개, 총 파드 500-5,000개):
  → m5.xlarge + Prefix Delegation 권장
  → 노드당 최대 56개 파드 (PD 없이) 또는 110+ (PD 활성화)
  → 서브넷: /20 이상 (AZ당)
  → 노드 수를 줄여 비용 최적화 가능

대규모 (노드 50개 이상, 총 파드 5,000개 이상):
  → m5.2xlarge 이상 + Prefix Delegation 필수
  → 서브넷: /18 이상 (AZ당), 보조 CIDR (100.64.0.0/16) 활용 권장
  → 전용 파드 서브넷 필수
  → Cilium Operator replica 2개 이상 운영
```

### 비용 고려 사항

```
ENI 관련 비용 항목:
  1. ENI 자체: 무료 (인스턴스에 연결된 상태)
  2. VPC IP 주소: AWS에서 2024년부터 퍼블릭 IPv4 과금 ($0.005/시간)
     → 파드에 할당되는 프라이빗 IP는 무과금
  3. 인스턴스 비용: ENI/IP 제한이 높은 인스턴스일수록 비용 증가

비용 최적화 전략:
  1. Prefix Delegation 활성화 → 작은 인스턴스에서 더 많은 파드 실행
     → 노드 수 감소 → EC2 비용 절감
  2. awsReleaseExcessIPs: true 설정 → 미사용 IP 반환
  3. pre-allocate 값 최적화 → 과도한 IP 사전 할당 방지
  4. Spot Instance 혼합 사용 시 ENI 제한 확인 필수

비용 계산 예시 (파드 1,000개 기준):
  전략 A: m5.large × 40대 (27 파드/노드) = 40 × $0.096/h = $3.84/h
  전략 B: m5.xlarge × 18대 (56 파드/노드) = 18 × $0.192/h = $3.46/h
  전략 C: m5.xlarge + PD × 10대 (110 파드/노드) = 10 × $0.192/h = $1.92/h
  → Prefix Delegation 활용 시 약 50% 비용 절감 가능
```

### 추가 팁

```
1. EKS 관리형 노드 그룹 사용 시 --max-pods 설정 주의
   → ENI 모드에서는 기본 maxPods가 인스턴스 ENI 제한에 맞춰 설정됨
   → Prefix Delegation 활성화 시 maxPods를 수동으로 높여야 효과를 발휘

2. Cilium Operator는 반드시 2개 이상의 레플리카로 운영
   → Operator 장애 시 IP 할당/해제가 중단됨
   → Leader Election (리더 선출)으로 중복 작업 방지

3. ENI 모드와 NetworkPolicy는 독립적으로 동작
   → ENI 모드에서도 CiliumNetworkPolicy, CiliumClusterwideNetworkPolicy 정상 사용 가능
   → eBPF 기반이므로 iptables보다 높은 성능

4. 노드 교체 시 ENI 자동 정리
   → 인스턴스 종료 시 연결된 ENI도 자동 삭제
   → deleteOnTermination 속성이 기본 true
```
