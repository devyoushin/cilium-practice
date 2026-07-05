# Cilium 설치 (EKS + Helm)

EKS 환경에서 Cilium을 ENI 모드로 설치합니다.
AWS VPC CNI(aws-node)를 Cilium로 완전히 대체하며, kube-proxy도 BPF로 교체합니다.

---

## 사전 조건 확인

```bash
# kubectl 연결 확인
kubectl get nodes

# Helm 버전 확인 (>= 3.10 권장)
helm version

# AWS CLI 설정 확인
aws sts get-caller-identity

# (선택) Cilium CLI 설치
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-darwin-amd64.tar.gz
tar xzvf cilium-darwin-amd64.tar.gz
sudo mv cilium /usr/local/bin
```

---

## 설치 전 주의사항

> EKS에서 Cilium을 CNI로 사용하려면 **새 노드 그룹**에 적용하거나,
> **기존 노드를 drain 후 교체**해야 합니다.
> 운영 중인 노드에 aws-node를 그냥 삭제하면 네트워크 단절이 발생합니다.

권장 절차:
1. 신규 노드 그룹 생성 (기존 워크로드는 기존 노드에서 유지)
2. 신규 노드 그룹에 Cilium 설치
3. 워크로드를 신규 노드로 마이그레이션
4. 기존 노드 그룹 제거

---

## 1. aws-node DaemonSet 비활성화

Cilium이 ENI를 직접 관리하기 때문에 AWS VPC CNI와 충돌을 피해야 합니다.

```bash
# aws-node 이미지를 pause로 교체하여 비활성화 (삭제하면 안 됨)
kubectl -n kube-system set image daemonset/aws-node \
  aws-node=public.ecr.aws/amazonlinux/amazonlinux:2023

# 확인: aws-node 파드가 더 이상 네트워크 설정을 하지 않는지 검증
kubectl get daemonset aws-node -n kube-system
```

---

## 2. kube-proxy 비활성화 (선택 — BPF 대체 사용 시)

```bash
# kube-proxy DaemonSet 삭제 (Cilium BPF가 대체)
kubectl -n kube-system delete daemonset kube-proxy

# kube-proxy ConfigMap도 제거
kubectl -n kube-system delete configmap kube-proxy
```

> BPF kube-proxy replacement를 사용하면 iptables 의존성이 제거되어
> 성능이 크게 향상됩니다 (대규모 클러스터에서 특히 효과적).

---

## 3. Helm 저장소 추가

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

# 사용 가능한 버전 확인
helm search repo cilium/cilium --versions | head -10
```

---

## 4. Helm values 파일 준비

```bash
cp helm/values.yaml my-values.yaml
```

`my-values.yaml`에서 반드시 수정할 항목:

```yaml
# EKS 클러스터 정보
eni:
  awsRegion: ap-northeast-2      # 실제 리전으로 변경

# (kube-proxy 대체 사용 시) API Server 엔드포인트
kubeProxyReplacement: true
k8sServiceHost: <API_SERVER_ENDPOINT>   # EKS API Server 주소
k8sServicePort: "443"
```

API Server 엔드포인트 확인:

```bash
aws eks describe-cluster \
  --name <CLUSTER_NAME> \
  --query "cluster.endpoint" \
  --output text
# 예: https://XXXXXXXX.gr7.ap-northeast-2.eks.amazonaws.com
# → k8sServiceHost: XXXXXXXX.gr7.ap-northeast-2.eks.amazonaws.com (https:// 제외)
```

---

## 5. Cilium 설치

```bash
CILIUM_CHART_VERSION="1.16.0"

helm install cilium cilium/cilium \
  --version ${CILIUM_CHART_VERSION} \
  --namespace kube-system \
  --values my-values.yaml \
  --timeout 10m \
  --wait
```

---

## 6. 설치 확인

```bash
# Cilium 파드 상태 확인
kubectl get pods -n kube-system -l k8s-app=cilium -w

# Cilium Operator 상태 확인
kubectl get pods -n kube-system -l io.cilium/app=operator
```

정상 설치 시 모든 파드가 `Running` 상태여야 합니다:

| 컴포넌트 | 종류 | 파드 수 |
|---|---|---|
| cilium | DaemonSet | 노드 수만큼 |
| cilium-operator | Deployment | 1~2개 |
| hubble-relay | Deployment | 1개 (Hubble 활성화 시) |
| hubble-ui | Deployment | 1개 (Hubble UI 활성화 시) |

```bash
# 빠른 상태 요약 (Cilium CLI)
cilium status --wait

# CLI 없이 확인
kubectl -n kube-system exec ds/cilium -- cilium status
```

---

## 7. 연결성 테스트

```bash
# Cilium CLI로 기본 연결 테스트 (파드 간 통신 검증)
cilium connectivity test

# 간단한 파드 투 파드 테스트
kubectl run test-a --image=nicolaka/netshoot --rm -it -- curl http://test-b
```

---

## 8. 노드 확인 — ENI 할당 상태

```bash
# 각 노드의 ENI 할당 상태 확인
kubectl get ciliumnodes -o wide

# 특정 노드 상세 확인
kubectl get ciliumnode <NODE_NAME> -o yaml | grep -A 20 "ipv4:"
```

---

## 업그레이드

```bash
# 업그레이드 전 현재 values 확인
helm get values cilium -n kube-system

# 업그레이드
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --values my-values.yaml \
  --version 1.16.1 \
  --timeout 10m

# 업그레이드 히스토리 확인
helm history cilium -n kube-system
```

> **주의**: Cilium Agent는 DaemonSet이므로 노드별 순차 롤링 업데이트됩니다.
> 업그레이드 중에는 네트워크 정책이 일시적으로 변동될 수 있습니다.

---

## 롤백

```bash
helm rollback cilium 1 -n kube-system
```

---

## 삭제

```bash
helm uninstall cilium -n kube-system

# Cilium CRD 삭제
kubectl delete crds \
  ciliumnetworkpolicies.cilium.io \
  ciliumclusterwidenetworkpolicies.cilium.io \
  ciliumnodes.cilium.io \
  ciliumendpoints.cilium.io \
  ciliumidentities.cilium.io \
  ciliumegressgatewaypolicies.cilium.io

# aws-node 복구 (Cilium 제거 후 VPC CNI로 되돌리는 경우)
kubectl -n kube-system rollout undo daemonset/aws-node
```

---

## 설치 시 자주 발생하는 문제

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| Cilium 파드 `Init:CrashLoopBackOff` | 커널 버전 미지원 | 노드 OS 확인 (AL2 커널 >= 4.14, 권장 5.10+) |
| 파드 간 통신 불가 | aws-node와 충돌 | aws-node 이미지 교체 후 노드 재시작 |
| `kube-proxy replacement: disabled` | API Server 주소 오류 | `k8sServiceHost` / `k8sServicePort` 재확인 |
| ENI 할당 실패 | IAM 권한 부족 | 노드 IAM Role에 `AmazonEKS_CNI_Policy` 확인 |
| Hubble Relay `Pending` | 리소스 부족 | 노드 용량 확인 |
| `cilium connectivity test` 실패 | 보안그룹 문제 | 노드 간 포트 4240(health), 4244(Hubble) 허용 |
