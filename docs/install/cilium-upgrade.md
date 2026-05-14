# Cilium 업그레이드 가이드

## 1. 개요

Cilium은 eBPF 기반 네트워킹, 보안, 관측 기능을 제공하는 CNI(Container Network Interface) 플러그인으로, 주기적인 업그레이드를 통해 보안 패치(security patch), 버그 수정(bug fix), 신규 기능을 반영해야 함. 특히 EKS 환경에서는 Kubernetes 버전과 Cilium 버전 간 호환성(compatibility)을 반드시 확인해야 하며, ENI 모드(AWS VPC native) 사용 시 AWS VPC CNI와의 충돌 방지를 위한 추가 검증이 필요함.

업그레이드를 방치할 경우 발생할 수 있는 문제:

- EOL(End of Life) 버전 사용으로 인한 보안 취약점(vulnerability) 노출
- EKS Kubernetes 버전 업그레이드 시 Cilium 비호환으로 인한 네트워크 장애
- 누적된 버전 차이로 인해 한 번에 여러 마이너 버전(minor version)을 건너뛰어야 하는 상황 발생

---

## 2. 설명

### 2.1 핵심 개념

#### Cilium 버전 정책 (Version Policy)

Cilium은 시맨틱 버저닝(Semantic Versioning)을 따르며, `MAJOR.MINOR.PATCH` 형식을 사용함.

- **마이너 버전(Minor Version)**: 최신 3개의 마이너 버전에 대해 보안 패치 및 버그 수정을 지원함. 예를 들어 1.16.x가 최신이면 1.15.x, 1.14.x까지 지원 대상.
- **패치 버전(Patch Version)**: 동일 마이너 버전 내에서 자유롭게 업그레이드 가능. 예: 1.16.0 → 1.16.5
- **마이너 버전 업그레이드**: 반드시 한 단계씩 순차적으로 수행해야 함. 예: 1.14.x → 1.15.x → 1.16.x (1.14.x → 1.16.x 직접 업그레이드 불가)

#### EKS 호환성 매트릭스 (Compatibility Matrix)

| Cilium 버전 | EKS Kubernetes 버전 | 비고 |
|---|---|---|
| 1.16.x | 1.28, 1.29, 1.30, 1.31 | 현재 권장 버전 |
| 1.15.x | 1.27, 1.28, 1.29, 1.30 | 유지보수 지원 중 |
| 1.14.x | 1.26, 1.27, 1.28, 1.29 | 유지보수 지원 중 |

> **참고**: 정확한 호환성 정보는 [Cilium 공식 문서](https://docs.cilium.io/en/stable/network/kubernetes/requirements/)에서 확인할 것.

#### eBPF 프로그램 리로드 (eBPF Program Reload)

업그레이드 시 Cilium 에이전트(agent)가 재시작되면서 eBPF 프로그램이 커널(kernel)에 다시 로드됨. 이 과정에서 알아야 할 사항:

- eBPF 맵(map)은 에이전트 재시작 시에도 커널에 유지됨 (pinned maps)
- 새 버전의 eBPF 프로그램이 기존 맵 포맷(format)과 호환되지 않으면 문제 발생 가능
- 롤링 업데이트(rolling update) 방식으로 노드 단위 순차 교체하여 전체 장애를 방지함
- 업그레이드 중 일시적으로 구버전과 신버전 에이전트가 공존하는 상태가 발생함

#### 업그레이드 전 체크리스트 (Pre-upgrade Checklist)

- [ ] 현재 Cilium 버전 및 EKS Kubernetes 버전 확인
- [ ] 대상 Cilium 버전과 현재 EKS 버전 간 호환성 확인
- [ ] 마이너 버전 간 업그레이드 시 한 단계씩 수행하는지 확인
- [ ] 현재 Cilium 상태가 정상인지 점검 (모든 에이전트 Ready 상태)
- [ ] 커넥티비티 테스트(connectivity test) 통과 확인
- [ ] 현재 Helm values 백업 완료
- [ ] CRD(Custom Resource Definition) 변경 사항 확인
- [ ] 스테이징 환경(staging environment)에서 사전 테스트 완료
- [ ] 유지보수 윈도우(maintenance window) 스케줄 확보

---

### 2.2 실무 적용 코드

#### Step 1: 업그레이드 전 상태 점검 (Pre-upgrade Checks)

현재 Cilium 클러스터 상태를 확인함.

```bash
# 현재 Cilium 버전 확인
cilium version -n kube-system

# Cilium 전체 상태 확인
cilium status -n kube-system

# Cilium 에이전트 Pod 상태 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium-agent -o wide

# Cilium Operator 상태 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium-operator

# 현재 설치된 Helm 릴리스 정보 확인
helm list -n kube-system -f cilium
```

커넥티비티 테스트를 수행하여 현재 네트워크 상태가 정상인지 확인함.

```bash
# 커넥티비티 테스트 수행
cilium connectivity test -n kube-system

# 특정 테스트만 수행하고 싶은 경우
cilium connectivity test -n kube-system --test pod-to-pod
cilium connectivity test -n kube-system --test pod-to-service
```

EKS 클러스터 버전 확인:

```bash
# EKS 클러스터 Kubernetes 버전 확인
kubectl version --short

# 또는 AWS CLI로 확인
aws eks describe-cluster --name <CLUSTER_NAME> --query 'cluster.version' --output text
```

#### Step 2: 현재 Helm Values 백업 (Backup Current Values)

업그레이드 실패 시 복원을 위해 현재 설정값을 반드시 백업함.

```bash
# 현재 Helm values를 파일로 추출
helm get values cilium -n kube-system -o yaml > cilium-values-backup-$(date +%Y%m%d%H%M%S).yaml

# Helm 릴리스 전체 매니페스트 백업
helm get manifest cilium -n kube-system > cilium-manifest-backup-$(date +%Y%m%d%H%M%S).yaml

# 현재 Helm 릴리스 리비전 번호 기록
helm history cilium -n kube-system
```

#### Step 3: Helm 기반 롤링 업그레이드 (Helm-based Rolling Upgrade)

Cilium Helm 리포지토리(repository)를 업데이트하고 업그레이드를 수행함.

```bash
# Helm 리포지토리 업데이트
helm repo update cilium

# 사용 가능한 Cilium 버전 목록 확인
helm search repo cilium/cilium --versions | head -20

# 업그레이드 대상 버전의 기본 values 확인 (변경 사항 파악용)
helm show values cilium/cilium --version <TARGET_VERSION> > cilium-default-values-<TARGET_VERSION>.yaml
```

업그레이드 실행 전 dry-run으로 변경 사항을 미리 확인함.

```bash
# dry-run으로 변경 사항 미리 확인
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version <TARGET_VERSION> \
  --values cilium-values-backup-<TIMESTAMP>.yaml \
  --dry-run --debug
```

실제 업그레이드를 수행함.

```bash
# Helm 업그레이드 수행
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version <TARGET_VERSION> \
  --values cilium-values-backup-<TIMESTAMP>.yaml \
  --wait \
  --timeout 10m
```

ENI 모드에서의 업그레이드 시 추가 설정이 필요한 경우:

```bash
# ENI 모드 전용 업그레이드 (커스텀 values 포함)
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version <TARGET_VERSION> \
  --set eni.enabled=true \
  --set ipam.mode=eni \
  --set egressMasqueradeInterfaces=eth0 \
  --set routingMode=native \
  --values cilium-values-backup-<TIMESTAMP>.yaml \
  --wait \
  --timeout 10m
```

#### Step 4: 업그레이드 후 검증 (Post-upgrade Verification)

```bash
# 업그레이드된 Cilium 버전 확인
cilium version -n kube-system

# Cilium 전체 상태 확인 (모든 에이전트가 Ready 상태인지)
cilium status -n kube-system

# 모든 Cilium Pod가 새 버전으로 교체되었는지 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium-agent \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

# Cilium Operator 이미지 버전 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium-operator \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

# CRD 버전 확인
kubectl get crd | grep cilium

# 커넥티비티 테스트 재수행
cilium connectivity test -n kube-system

# 엔드포인트 상태 확인
kubectl get ciliumendpoints -n <APP_NAMESPACE>

# Cilium 에이전트 로그에서 에러 확인
kubectl logs -n kube-system -l app.kubernetes.io/name=cilium-agent --tail=100 | grep -i error
```

#### 카나리 업그레이드 전략 (Canary Upgrade Strategy)

전체 클러스터를 한 번에 업그레이드하지 않고, 특정 노드에서 먼저 검증하는 전략.

```bash
# 1. 카나리 노드에 레이블(label) 추가
kubectl label node <CANARY_NODE_NAME> cilium-upgrade=canary

# 2. 기존 DaemonSet의 nodeAffinity를 수정하여 카나리 노드 제외
# (별도의 Cilium DaemonSet을 배포하는 방식이 더 안전함)

# 3. 카나리 노드의 Cilium Pod 삭제하여 새 버전으로 재생성 유도
# 먼저 maxUnavailable을 1로 제한
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version <TARGET_VERSION> \
  --values cilium-values-backup-<TIMESTAMP>.yaml \
  --set updateStrategy.rollingUpdate.maxUnavailable=1 \
  --wait \
  --timeout 15m

# 4. 카나리 노드에서 먼저 업그레이드가 진행되는지 모니터링
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium-agent \
  -o wide --watch

# 5. 카나리 노드의 Cilium 에이전트 상태 확인
kubectl exec -n kube-system <CANARY_CILIUM_POD> -- cilium status

# 6. 카나리 노드에서 실행 중인 워크로드의 네트워크 연결 테스트
kubectl exec -n <APP_NAMESPACE> <TEST_POD_ON_CANARY_NODE> -- curl -s http://<SERVICE_NAME>.<SERVICE_NAMESPACE>.svc.cluster.local

# 7. 검증 완료 후 나머지 노드에도 순차 업그레이드 진행 (위 helm upgrade가 롤링으로 처리)
```

#### 롤백 절차 (Rollback Procedure)

업그레이드 후 문제가 발생한 경우 Helm 롤백을 수행함.

```bash
# 현재 릴리스 히스토리 확인
helm history cilium -n kube-system

# 출력 예시:
# REVISION  UPDATED                   STATUS      CHART          APP VERSION  DESCRIPTION
# 1         2024-01-15 10:00:00       superseded  cilium-1.15.6  1.15.6       Install complete
# 2         2024-03-20 14:00:00       deployed    cilium-1.16.0  1.16.0       Upgrade complete

# 이전 리비전으로 롤백
helm rollback cilium <PREVIOUS_REVISION_NUMBER> -n kube-system --wait --timeout 10m

# 롤백 후 상태 확인
cilium status -n kube-system

# 롤백 후 버전 확인
helm list -n kube-system -f cilium

# 롤백 후 커넥티비티 테스트
cilium connectivity test -n kube-system
```

---

### 2.3 보안/성능 Best Practice

#### 커넥티비티 테스트 필수 수행

업그레이드 전후로 반드시 커넥티비티 테스트를 수행하여 네트워크 정책(Network Policy)이 정상 동작하는지 확인해야 함.

```bash
# 업그레이드 전 테스트 결과 저장
cilium connectivity test -n kube-system 2>&1 | tee connectivity-test-pre-upgrade-$(date +%Y%m%d%H%M%S).log

# 업그레이드 후 테스트 결과 저장
cilium connectivity test -n kube-system 2>&1 | tee connectivity-test-post-upgrade-$(date +%Y%m%d%H%M%S).log

# 두 결과 비교
diff connectivity-test-pre-upgrade-<PRE_TIMESTAMP>.log connectivity-test-post-upgrade-<POST_TIMESTAMP>.log
```

#### Hubble을 통한 드롭 패킷(Dropped Packet) 모니터링

업그레이드 중 패킷 드롭이 발생하는지 실시간으로 모니터링함.

```bash
# Hubble CLI로 드롭된 패킷 실시간 모니터링
hubble observe -n kube-system --verdict DROPPED --follow

# 특정 네임스페이스의 트래픽 모니터링
hubble observe -n <APP_NAMESPACE> --follow

# Hubble UI 포트포워딩 (브라우저에서 시각적 모니터링)
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

#### 유지보수 윈도우에서의 업그레이드 스케줄링

- 프로덕션 환경(production environment)에서는 반드시 트래픽이 적은 시간대에 수행
- PodDisruptionBudget(PDB)을 설정하여 동시에 중단되는 Pod 수를 제한
- 업그레이드 전 모든 관련 팀에 사전 공지

```yaml
# PodDisruptionBudget 설정 예시 (애플리케이션 측)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: <APP_NAME>-pdb
  namespace: <APP_NAMESPACE>
spec:
  minAvailable: "50%"
  selector:
    matchLabels:
      app: <APP_NAME>
```

#### 스테이징 환경에서 사전 테스트

프로덕션 업그레이드 전 반드시 스테이징 환경에서 동일한 절차를 수행하여 검증해야 함.

```bash
# 스테이징 클러스터 컨텍스트 전환
kubectl config use-context <STAGING_CLUSTER_CONTEXT>

# 스테이징에서 업그레이드 수행 (프로덕션과 동일한 절차)
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version <TARGET_VERSION> \
  --values cilium-values-backup-<TIMESTAMP>.yaml \
  --wait \
  --timeout 10m

# 스테이징에서 최소 24시간 모니터링 후 프로덕션 적용 결정
```

---

## 3. 트러블슈팅

### 3.1 주요 이슈

#### 이슈 1: 업그레이드 후 Cilium Pod가 CrashLoopBackOff 상태 (eBPF 맵 포맷 비호환)

**증상**: 업그레이드 후 Cilium 에이전트 Pod가 반복적으로 재시작되며 `CrashLoopBackOff` 상태에 빠짐.

**원인**: 새 버전의 Cilium이 기존 eBPF 맵 포맷과 호환되지 않는 경우 발생. 특히 마이너 버전을 건너뛰었을 때 자주 발생함.

**진단**:

```bash
# Pod 상태 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium-agent

# 에이전트 로그 확인
kubectl logs -n kube-system <CILIUM_AGENT_POD> --previous

# eBPF 맵 관련 에러 로그 검색
kubectl logs -n kube-system <CILIUM_AGENT_POD> --previous | grep -i "bpf\|ebpf\|map"
```

**해결 방법**:

```bash
# 방법 1: eBPF 맵 정리 후 재시작
# 해당 노드의 Cilium Pod에 접속하여 bpf 맵 초기화
kubectl exec -n kube-system <CILIUM_AGENT_POD> -- cilium bpf maps list

# 방법 2: Cilium Pod 삭제하여 깨끗한 상태에서 재시작
kubectl delete pod -n kube-system <CILIUM_AGENT_POD>

# 방법 3: 문제가 지속되면 롤백 수행
helm rollback cilium <PREVIOUS_REVISION_NUMBER> -n kube-system --wait --timeout 10m

# 방법 4: 롤백 후 올바른 순서로 업그레이드 재시도 (마이너 버전 단계별)
```

#### 이슈 2: 업그레이드 중 네트워크 연결 손실 (롤링 업데이트 속도 과다)

**증상**: 업그레이드 진행 중 일부 Pod 간 통신이 끊기거나 외부 트래픽이 실패함.

**원인**: `maxUnavailable` 값이 너무 크게 설정되어 동시에 여러 노드의 Cilium 에이전트가 재시작되면서 네트워크 중단이 발생함.

**진단**:

```bash
# 현재 DaemonSet 업데이트 전략 확인
kubectl get daemonset cilium -n kube-system -o jsonpath='{.spec.updateStrategy}'

# 업데이트 진행 상황 모니터링
kubectl rollout status daemonset/cilium -n kube-system

# Hubble로 드롭 패킷 확인
hubble observe -n kube-system --verdict DROPPED
```

**해결 방법**:

```bash
# maxUnavailable을 1로 설정하여 한 번에 하나의 노드만 업그레이드
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version <TARGET_VERSION> \
  --values cilium-values-backup-<TIMESTAMP>.yaml \
  --set updateStrategy.rollingUpdate.maxUnavailable=1 \
  --wait \
  --timeout 20m

# 긴급 시 롤아웃 일시 중지 (DaemonSet은 직접 pause 불가하므로 롤백으로 대응)
helm rollback cilium <PREVIOUS_REVISION_NUMBER> -n kube-system --wait
```

#### 이슈 3: 업그레이드 후 CRD 버전 불일치 (CRD Version Mismatch)

**증상**: 업그레이드 후 `CiliumNetworkPolicy`, `CiliumClusterwideNetworkPolicy` 등의 CRD가 정상 동작하지 않거나 API 에러가 발생함.

**원인**: Helm 업그레이드 시 CRD가 자동으로 업데이트되지 않는 경우 발생. Helm은 기본적으로 CRD를 업데이트하지 않음(install만 수행).

**진단**:

```bash
# 현재 CRD 버전 확인
kubectl get crd ciliumnetworkpolicies.cilium.io -o jsonpath='{.metadata.annotations}'

# CRD에 정의된 스키마 버전 확인
kubectl get crd ciliumnetworkpolicies.cilium.io -o jsonpath='{.spec.versions[*].name}'

# Cilium 에이전트 로그에서 CRD 관련 에러 확인
kubectl logs -n kube-system <CILIUM_AGENT_POD> | grep -i "crd\|custom resource"
```

**해결 방법**:

```bash
# CRD를 수동으로 업데이트
# 대상 버전의 CRD 매니페스트를 다운로드하여 적용
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/<TARGET_VERSION>/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumnetworkpolicies.yaml
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/<TARGET_VERSION>/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumclusterwidenetworkpolicies.yaml
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/<TARGET_VERSION>/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumendpoints.yaml
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/<TARGET_VERSION>/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumidentities.yaml
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/<TARGET_VERSION>/pkg/k8s/apis/cilium.io/client/crds/v2/ciliumnodes.yaml

# 또는 Helm 차트에서 CRD를 추출하여 적용
helm template cilium cilium/cilium --version <TARGET_VERSION> \
  --include-crds --set operator.replicas=0 \
  | kubectl apply -f - --server-side --force-conflicts

# CRD 업데이트 후 Cilium 에이전트 재시작
kubectl rollout restart daemonset/cilium -n kube-system
```

#### 이슈 4: 업그레이드 후 Hubble Relay 연결 불가

**증상**: 업그레이드 후 `hubble observe` 명령이 실패하거나 Hubble UI에서 플로우(flow) 데이터가 표시되지 않음.

**원인**: Hubble Relay가 새 버전의 Cilium 에이전트와 gRPC 연결을 맺지 못하는 경우 발생. TLS 인증서(certificate) 만료 또는 버전 간 gRPC API 변경이 원인일 수 있음.

**진단**:

```bash
# Hubble Relay Pod 상태 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=hubble-relay

# Hubble Relay 로그 확인
kubectl logs -n kube-system -l app.kubernetes.io/name=hubble-relay --tail=100

# Hubble 상태 확인
hubble status

# Hubble Relay와 에이전트 간 TLS 인증서 확인
kubectl get secret -n kube-system hubble-relay-client-certs
kubectl get secret -n kube-system hubble-server-certs
```

**해결 방법**:

```bash
# 방법 1: Hubble Relay Pod 재시작
kubectl rollout restart deployment/hubble-relay -n kube-system

# 방법 2: TLS 인증서 재생성 (cilium CLI 사용)
cilium hubble disable -n kube-system
cilium hubble enable -n kube-system

# 방법 3: Helm으로 Hubble 관련 설정 재적용
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version <TARGET_VERSION> \
  --values cilium-values-backup-<TIMESTAMP>.yaml \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.tls.auto.enabled=true \
  --set hubble.tls.auto.method=cronJob \
  --wait \
  --timeout 10m

# 방법 4: Hubble 인증서 시크릿 삭제 후 재생성
kubectl delete secret -n kube-system hubble-relay-client-certs hubble-server-certs
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout restart deployment/hubble-relay -n kube-system
```

---

### 3.2 자주 발생하는 문제 (Q&A)

**Q1: 마이너 버전을 건너뛸 수 있는가? (예: 1.14.x → 1.16.x)**

A: 불가능함. Cilium 공식 정책상 마이너 버전 업그레이드는 반드시 한 단계씩 순차적으로 수행해야 함. 1.14.x에서 1.16.x로 업그레이드하려면 다음 순서를 따를 것:

```bash
# Step 1: 1.14.x → 1.15.x (최신 패치 버전으로)
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version 1.15.<LATEST_PATCH> \
  --values cilium-values.yaml \
  --wait --timeout 10m

# 검증
cilium status -n kube-system
cilium connectivity test -n kube-system

# Step 2: 1.15.x → 1.16.x (최신 패치 버전으로)
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version 1.16.<LATEST_PATCH> \
  --values cilium-values.yaml \
  --wait --timeout 10m

# 최종 검증
cilium status -n kube-system
cilium connectivity test -n kube-system
```

**Q2: 업그레이드 시 커스텀 Helm values는 어떻게 처리하는가?**

A: 업그레이드 전에 반드시 현재 values를 백업하고, 새 버전의 기본 values와 비교하여 deprecated 또는 변경된 옵션이 있는지 확인해야 함.

```bash
# 현재 커스텀 values 추출
helm get values cilium -n kube-system -o yaml > current-values.yaml

# 새 버전의 기본 values 확인
helm show values cilium/cilium --version <TARGET_VERSION> > new-default-values.yaml

# 두 파일을 비교하여 변경된 키 확인
diff current-values.yaml new-default-values.yaml

# deprecated 옵션이 있다면 새 옵션명으로 변경 후 업그레이드
# 예: 1.15 → 1.16에서 특정 옵션이 변경된 경우
# old: containerRuntime.integration=crio
# new: cri.runtime=crio (예시)

# 수정된 values 파일로 업그레이드
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version <TARGET_VERSION> \
  --values current-values-updated.yaml \
  --wait --timeout 10m
```

**Q3: 롤백이 실패하면 어떻게 대응하는가?**

A: Helm 롤백이 실패하는 경우 다음 단계를 순서대로 시도할 것.

```bash
# 1. Helm 롤백 실패 시 강제 롤백 시도
helm rollback cilium <PREVIOUS_REVISION_NUMBER> -n kube-system --force --wait --timeout 15m

# 2. 그래도 실패하면 Helm 릴리스를 삭제하고 이전 버전으로 재설치
# 주의: 이 방법은 일시적인 네트워크 중단이 발생할 수 있음
# 먼저 백업된 values 파일이 있는지 확인
cat cilium-values-backup-<TIMESTAMP>.yaml

# Helm 릴리스 삭제 (CRD는 유지됨)
helm uninstall cilium -n kube-system --wait --timeout 10m

# 이전 버전으로 재설치
helm install cilium cilium/cilium \
  --namespace kube-system \
  --version <PREVIOUS_VERSION> \
  --values cilium-values-backup-<TIMESTAMP>.yaml \
  --wait --timeout 10m

# 3. 최후의 수단: Cilium을 완전히 제거하고 재설치
# 주의: 이 작업은 네트워크 정책(NetworkPolicy)이 일시적으로 적용되지 않음
# 모든 CiliumNetworkPolicy를 먼저 백업
kubectl get ciliumnetworkpolicies --all-namespaces -o yaml > cnp-backup.yaml
kubectl get ciliumclusterwidenetworkpolicies -o yaml > ccnp-backup.yaml
```

---

## 4. 모니터링 및 확인

### 클러스터 상태 확인 명령어

```bash
# Cilium 에이전트 상태 종합 확인
cilium status -n kube-system

# 모든 노드의 Cilium 버전 일괄 확인
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium-agent \
  -o jsonpath='{range .items[*]}Node: {.spec.nodeName} | Image: {.spec.containers[0].image}{"\n"}{end}'

# Helm 릴리스 상태 확인
helm list -n kube-system -f cilium -o yaml

# Cilium 에이전트별 상세 버전 정보
kubectl exec -n kube-system <CILIUM_AGENT_POD> -- cilium version
```

### Hubble 플로우 모니터링 (업그레이드 중)

업그레이드 진행 중 실시간으로 트래픽 플로우를 모니터링하여 문제를 조기에 감지함.

```bash
# 전체 드롭 패킷 모니터링
hubble observe --verdict DROPPED --follow

# 특정 네임스페이스의 트래픽 모니터링
hubble observe -n <APP_NAMESPACE> --follow

# DNS 관련 이슈 모니터링
hubble observe -n kube-system --protocol DNS --follow

# 특정 Pod 간 트래픽 확인
hubble observe --from-pod <NAMESPACE>/<POD_NAME> --follow
hubble observe --to-pod <NAMESPACE>/<POD_NAME> --follow

# Hubble 메트릭 요약
hubble observe -n kube-system --last 5m -o compact | tail -50
```

### Prometheus 메트릭 모니터링

업그레이드 전후로 모니터링해야 할 핵심 Prometheus 메트릭(metric).

```bash
# Cilium 에이전트의 Prometheus 메트릭 엔드포인트 직접 확인
kubectl exec -n kube-system <CILIUM_AGENT_POD> -- \
  curl -s http://localhost:9962/metrics | grep cilium_endpoint_state

kubectl exec -n kube-system <CILIUM_AGENT_POD> -- \
  curl -s http://localhost:9962/metrics | grep cilium_agent_bootstrap_seconds
```

주요 모니터링 대상 메트릭:

| 메트릭 | 설명 | 정상 기준 |
|---|---|---|
| `cilium_endpoint_state` | 엔드포인트 상태별 개수 | `ready` 상태가 대다수일 것 |
| `cilium_agent_bootstrap_seconds` | 에이전트 부트스트랩 소요 시간 | 이전 버전 대비 크게 증가하지 않을 것 |
| `cilium_datapath_errors_total` | 데이터 경로 에러 수 | 업그레이드 전후로 급증하지 않을 것 |
| `cilium_drop_count_total` | 드롭된 패킷 수 | 업그레이드 중 일시적 증가 가능하나 이후 안정화 |
| `cilium_policy_import_errors_total` | 정책 임포트 에러 수 | 0을 유지할 것 |
| `cilium_unreachable_nodes` | 도달 불가 노드 수 | 0을 유지할 것 |
| `cilium_bpf_map_ops_total` | eBPF 맵 작업 수 | 에러 라벨이 없을 것 |

Grafana 대시보드(dashboard)용 PromQL 쿼리 예시:

```promql
# 엔드포인트 상태 분포 (업그레이드 중 변화 모니터링)
sum by (state) (cilium_endpoint_state)

# 에이전트 부트스트랩 시간 (히스토그램)
histogram_quantile(0.99, rate(cilium_agent_bootstrap_seconds_bucket[5m]))

# 패킷 드롭률 변화
rate(cilium_drop_count_total[5m])

# 데이터 경로 에러율
rate(cilium_datapath_errors_total[5m])

# 정책 임포트 에러
increase(cilium_policy_import_errors_total[1h])
```

---

## 5. TIP

### 공식 문서 참조 링크

- [Cilium Upgrade Guide (공식)](https://docs.cilium.io/en/stable/operations/upgrade/)
- [Cilium Release Notes](https://github.com/cilium/cilium/releases)
- [Cilium Version Compatibility Matrix](https://docs.cilium.io/en/stable/network/kubernetes/requirements/)
- [Cilium Helm Chart Values Reference](https://docs.cilium.io/en/stable/helm-reference/)
- [EKS에서 Cilium 운영 가이드](https://docs.cilium.io/en/stable/installation/k8s-install-helm/)

### 권장 업그레이드 주기 (Recommended Upgrade Cadence)

- **패치 버전**: 보안 패치가 포함된 경우 릴리스 후 1~2주 이내에 적용. 일반 버그 수정은 월 1회 주기로 적용 권장.
- **마이너 버전**: 새 마이너 버전 릴리스 후 최소 2~4주간 커뮤니티 안정성 확인 기간을 거친 뒤 스테이징에 적용. 스테이징에서 1~2주 검증 후 프로덕션 적용.
- **EOL 방지**: 현재 사용 중인 마이너 버전이 지원 종료되기 최소 1개월 전에 업그레이드 계획을 수립할 것.

### 업그레이드 자동화 스크립트 예시

반복적인 업그레이드 절차를 스크립트로 자동화한 예시:

```bash
#!/bin/bash
# cilium-upgrade.sh
# 사용법: ./cilium-upgrade.sh <TARGET_VERSION>

set -euo pipefail

TARGET_VERSION="${1:?Usage: $0 <TARGET_VERSION>}"
NAMESPACE="kube-system"
RELEASE_NAME="cilium"
BACKUP_DIR="./cilium-upgrade-backups/$(date +%Y%m%d%H%M%S)"

echo "=== Cilium 업그레이드 시작: 대상 버전 ${TARGET_VERSION} ==="

# 1. 백업 디렉토리 생성
mkdir -p "${BACKUP_DIR}"

# 2. 현재 상태 백업
echo "[1/6] 현재 설정 백업 중..."
helm get values ${RELEASE_NAME} -n ${NAMESPACE} -o yaml > "${BACKUP_DIR}/values.yaml"
helm get manifest ${RELEASE_NAME} -n ${NAMESPACE} > "${BACKUP_DIR}/manifest.yaml"
helm history ${RELEASE_NAME} -n ${NAMESPACE} > "${BACKUP_DIR}/history.txt"

# 3. 업그레이드 전 상태 점검
echo "[2/6] 업그레이드 전 상태 점검 중..."
cilium status -n ${NAMESPACE}

# 4. 커넥티비티 테스트
echo "[3/6] 업그레이드 전 커넥티비티 테스트 중..."
cilium connectivity test -n ${NAMESPACE} 2>&1 | tee "${BACKUP_DIR}/connectivity-pre.log"

# 5. Helm 업그레이드 수행
echo "[4/6] Helm 업그레이드 수행 중..."
helm upgrade ${RELEASE_NAME} cilium/cilium \
  --namespace ${NAMESPACE} \
  --version "${TARGET_VERSION}" \
  --values "${BACKUP_DIR}/values.yaml" \
  --wait \
  --timeout 10m

# 6. 업그레이드 후 상태 확인
echo "[5/6] 업그레이드 후 상태 확인 중..."
cilium status -n ${NAMESPACE}
cilium version -n ${NAMESPACE}

# 7. 업그레이드 후 커넥티비티 테스트
echo "[6/6] 업그레이드 후 커넥티비티 테스트 중..."
cilium connectivity test -n ${NAMESPACE} 2>&1 | tee "${BACKUP_DIR}/connectivity-post.log"

echo "=== Cilium 업그레이드 완료: ${TARGET_VERSION} ==="
echo "백업 위치: ${BACKUP_DIR}"
```
