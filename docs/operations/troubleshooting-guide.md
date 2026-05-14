# 트러블슈팅 가이드

---

## 기본 진단 명령어

```bash
# 전체 상태 한눈에 확인 (Cilium CLI)
cilium status --wait

# CLI 없이 확인
kubectl -n kube-system exec ds/cilium -- cilium status

# Cilium 파드 로그
kubectl -n kube-system logs ds/cilium -c cilium-agent --tail=100

# Cilium Operator 로그
kubectl -n kube-system logs deploy/cilium-operator --tail=100

# 모든 엔드포인트 상태
kubectl -n kube-system exec ds/cilium -- cilium endpoint list

# 노드 상태
kubectl get ciliumnodes -o wide
```

---

## 자주 발생하는 문제

### 1. Cilium 파드 `Init:CrashLoopBackOff`

**증상**: Cilium DaemonSet 파드가 초기화 중 반복 실패

**원인 및 해결**:

```bash
# 초기화 컨테이너 로그 확인
kubectl -n kube-system logs ds/cilium -c mount-cgroup
kubectl -n kube-system logs ds/cilium -c apply-sysctl-overwrites
kubectl -n kube-system logs ds/cilium -c clean-cilium-state

# 원인 1: 커널 버전 미지원
# → 노드 OS 확인 (AL2: 커널 5.10+ 권장, AL2023 권장)
uname -r

# 원인 2: 이전 Cilium 상태 파일 잔존
# → clean-cilium-state 컨테이너 로그 확인
# → 필요 시 노드 드레인 후 /var/run/cilium 디렉토리 정리

# 원인 3: aws-node와 충돌
kubectl -n kube-system get daemonset aws-node -o jsonpath='{.spec.template.spec.containers[0].image}'
# → pause 이미지가 아니라면 비활성화 필요
```

---

### 2. 파드 간 통신 불가

**증상**: 정책 없음에도 파드 A에서 파드 B로 ping/curl 실패

```bash
# 1. 엔드포인트 상태 확인
kubectl -n kube-system exec ds/cilium -- cilium endpoint list
# STATUS가 "ready"가 아닌 경우 문제 있음

# 2. Hubble로 흐름 확인
hubble observe --from-pod default/pod-a --to-pod default/pod-b --follow
# DROPPED이면 정책 차단, 흐름 없음이면 라우팅 문제

# 3. 정책 추적
kubectl -n kube-system exec ds/cilium -- \
  cilium policy trace \
  --src-k8s-pod default:pod-a \
  --dst-k8s-pod default:pod-b \
  --dport 80/TCP

# 4. 노드 간 라우팅 확인 (ENI 모드)
kubectl get ciliumnode <NODE_NAME> -o yaml | grep -A 10 "ipv4:"

# 5. 연결 테스트 (Cilium CLI)
cilium connectivity test --test pod-to-pod
```

---

### 3. DNS 오류 (Name Resolution Failure)

**증상**: 서비스 이름으로 통신 불가, `nslookup`/`dig` 실패

```bash
# 1. DNS 정책 확인 (Egress 정책에서 UDP 53 허용 여부)
kubectl get networkpolicy -A
kubectl get cnp -A

# 2. CoreDNS 파드 상태 확인
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 3. Hubble로 DNS 흐름 확인
hubble observe --protocol dns --follow

# 4. CoreDNS로의 연결 테스트
kubectl run -it --rm debug --image=nicolaka/netshoot -- \
  dig +short kubernetes.default.svc.cluster.local

# 5. Cilium DNS 프록시 상태 (L7 DNS 정책 사용 시)
kubectl -n kube-system exec ds/cilium -- \
  cilium proxy-stats
```

**해결**: Egress 정책 적용 시 DNS 허용 추가

```yaml
egress:
  - toEndpoints:
      - matchLabels:
          k8s:k8s-app: kube-dns
          k8s:io.kubernetes.pod.namespace: kube-system
    toPorts:
      - ports:
          - port: "53"
            protocol: ANY
        rules:
          dns:
            - matchPattern: "*"
```

---

### 4. kube-proxy Replacement 오류

**증상**: `KubeProxyReplacement: Disabled` 또는 서비스 IP로 접근 불가

```bash
# kube-proxy replacement 상태 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium status | grep KubeProxyReplacement

# API Server 연결 확인
kubectl -n kube-system exec ds/cilium -- \
  curl -k https://<API_SERVER_ENDPOINT>:443/healthz

# 서비스 BPF 맵 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium service list

# 서비스가 BPF Map에 없으면 values 재확인
# k8sServiceHost, k8sServicePort 값 점검
```

---

### 5. Hubble Relay 연결 실패

**증상**: `hubble status` 실패, `Error: failed to connect to 'localhost:4245'`

```bash
# Hubble Relay 파드 상태 확인
kubectl get pods -n kube-system -l k8s-app=hubble-relay

# Relay 로그 확인
kubectl -n kube-system logs deploy/hubble-relay

# 포트 포워드 재시작
pkill -f "port-forward.*hubble-relay"
kubectl port-forward svc/hubble-relay -n kube-system 4245:80 &

# Hubble 상태 재확인
hubble status
```

---

### 6. ENI IP 할당 실패

**증상**: 파드가 `ContainerCreating`에서 멈춤, `FailedCreatePodSandBox`

```bash
# Cilium Operator 로그 확인 (IPAM 관련)
kubectl -n kube-system logs deploy/cilium-operator | grep -i "eni\|ipam\|error"

# CiliumNode 상태 확인
kubectl get ciliumnode <NODE_NAME> -o yaml | grep -A 20 "ipam:"

# 노드의 ENI 상태 확인
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=<INSTANCE_ID>"

# IAM 권한 확인 (노드 역할에 필요한 정책)
aws iam list-attached-role-policies \
  --role-name <NODE_IAM_ROLE_NAME>
```

**필요한 IAM 권한**:
- `ec2:CreateNetworkInterface`
- `ec2:DescribeNetworkInterfaces`
- `ec2:AttachNetworkInterface`
- `ec2:DeleteNetworkInterface`
- `ec2:AssignPrivateIpAddresses`
- `ec2:UnassignPrivateIpAddresses`

---

### 7. 네트워크 정책이 적용되지 않음

```bash
# 정책 적용 확인
kubectl describe cnp <POLICY_NAME> -n <NAMESPACE>

# 엔드포인트가 정책을 인식하는지 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium endpoint get <ENDPOINT_ID> | jq '.spec.policy'

# 파드 재시작 후 레이블 재확인
kubectl get pod <POD_NAME> --show-labels

# Cilium Agent가 정책을 동기화했는지 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium policy get
```

---

## 진단 리포트 수집

Cilium 지원팀 또는 GitHub 이슈 제출 시 필요한 정보:

```bash
# 전체 진단 번들 수집 (Cilium CLI)
cilium sysdump --output-filename cilium-sysdump.zip

# 포함 내용:
# - cilium status
# - cilium endpoint list
# - kubectl get pods/nodes/services
# - Cilium Agent/Operator 로그
# - eBPF Map 상태
# - 네트워크 정책 목록
```

---

## 노드 수준 디버깅

```bash
# 노드에 직접 접속 후 eBPF 도구 사용
kubectl -n kube-system exec -it ds/cilium -- bash

# eBPF 프로그램 목록
bpftool prog list

# 특정 인터페이스의 eBPF 프로그램
bpftool net list dev eth0

# eBPF Map 내용 덤프
bpftool map dump name cilium_policy

# 커넥션 추적 상태
cilium bpf ct list global | head -30
```

---

## 유용한 디버깅 파드

```bash
# netshoot — 네트워크 디버깅 도구 모음
kubectl run debug --image=nicolaka/netshoot --rm -it -- bash

# 내부에서 사용 가능한 도구:
# ping, curl, dig, nslookup, tcpdump, ss, ip, iperf3, ...

# 특정 파드와 같은 네트워크 네임스페이스에서 디버깅
kubectl debug -it <POD_NAME> --image=nicolaka/netshoot --target=<CONTAINER_NAME>
```

---

## 참고 링크

- [Cilium 트러블슈팅 공식 문서](https://docs.cilium.io/en/stable/operations/troubleshooting/)
- [Cilium Sysdump](https://docs.cilium.io/en/stable/operations/troubleshooting/#sysdump)
- [eBPF 디버깅](https://docs.cilium.io/en/stable/operations/troubleshooting/#bpf)
