# FQDN 기반 Egress 정책 가이드

Cilium의 FQDN(Fully Qualified Domain Name) 기반 이그레스(Egress) 정책을 활용하여 파드의 외부 통신을 도메인 단위로 제어하는 방법에 대한 가이드.

---

## 1. 개요

### FQDN 기반 Egress 제어란

Cilium은 파드에서 외부로 나가는 트래픽(Egress)을 IP가 아닌 **도메인 이름(FQDN)** 기준으로 허용/차단하는 기능을 제공함. 기존 Kubernetes NetworkPolicy는 IP 기반 Egress 제어만 지원하지만, CiliumNetworkPolicy는 DNS 프록시(DNS Proxy)를 통해 도메인 레벨의 정밀한 제어가 가능함.

### IP 기반 Egress의 한계

클라우드 환경에서 IP 기반 이그레스 정책이 불충분한 이유:

```
문제점:
1. 클라우드 서비스 IP는 수시로 변경됨 (예: AWS S3, GitHub API)
2. CDN/로드밸런서 뒤의 서비스는 다수의 IP를 사용함
3. IP 대역이 넓어 과도한 허용 범위가 생김
4. IP 변경 시마다 정책을 수동 업데이트해야 함
```

FQDN 기반 정책은 Cilium이 자동으로 DNS 응답을 추적하여 해당 도메인의 현재 IP를 정책에 반영하므로, 위 문제를 근본적으로 해결함.

### 주요 활용 사례

| 사례 | 허용 대상 예시 |
|------|---------------|
| 외부 결제 API 연동 | `api.stripe.com` |
| AWS 서비스 접근 제한 | `*.amazonaws.com`, `s3.ap-northeast-2.amazonaws.com` |
| 외부 Git 저장소 접근 | `api.github.com`, `github.com` |
| SaaS 모니터링 연동 | `*.datadoghq.com`, `api.newrelic.com` |
| 외부 인증 서비스 | `accounts.google.com`, `login.microsoftonline.com` |

---

## 2. 설명

### 2.1 핵심 개념

#### DNS 프록시 인터셉션(DNS Proxy Interception)

Cilium의 FQDN 정책 동작 흐름:

```
┌──────────┐    DNS 요청     ┌──────────────────┐    DNS 요청     ┌──────────────┐
│          │ ──────────────→ │ Cilium DNS Proxy │ ──────────────→ │  CoreDNS /   │
│   파드   │                 │  (투명 인터셉트)   │                 │  kube-dns    │
│          │ ←────────────── │                  │ ←────────────── │              │
└──────────┘    DNS 응답     └──────────────────┘    DNS 응답     └──────────────┘
                                     │
                                     ▼
                          ┌─────────────────────┐
                          │ FQDN → IP 매핑 캐시  │
                          │ (eBPF 데이터플레인에  │
                          │  정책으로 반영)       │
                          └─────────────────────┘
```

1. 파드가 DNS 쿼리를 보내면, Cilium 에이전트(Agent)가 투명하게 인터셉트함
2. 실제 DNS 서버(CoreDNS/kube-dns)로 쿼리를 전달함
3. DNS 응답에서 도메인과 IP의 매핑 정보를 추출하여 캐시에 저장함
4. 해당 IP를 eBPF 데이터플레인(Dataplane)의 이그레스 정책에 동적 반영함

> **중요**: FQDN 정책이 동작하려면 파드가 반드시 DNS 쿼리를 먼저 수행해야 함. DNS를 거치지 않고 직접 IP로 접속하면 FQDN 정책이 적용되지 않음.

#### matchName vs matchPattern

| 필드 | 설명 | 예시 |
|------|------|------|
| `matchName` | 정확한 도메인 이름 매칭 | `api.github.com` |
| `matchPattern` | 와일드카드(*) 패턴 매칭 | `*.amazonaws.com` |

```yaml
# matchName: 정확히 일치하는 도메인만 허용
toFQDNs:
  - matchName: "api.github.com"

# matchPattern: 패턴에 일치하는 모든 하위 도메인 허용
toFQDNs:
  - matchPattern: "*.amazonaws.com"
```

**matchPattern 주의사항**:
- `*`는 단일 레벨(single-level)의 와일드카드임. `*.example.com`은 `sub.example.com`에 매칭되지만 `a.b.example.com`에는 매칭되지 않음
- 최상위 와일드카드(`*`)는 모든 도메인을 허용하므로 사용을 지양해야 함

#### DNS TTL 처리

Cilium은 DNS 응답의 TTL(Time to Live) 값을 기반으로 캐시 만료를 관리함:

```
DNS 응답: api.github.com → 140.82.121.6 (TTL: 60초)

┌─────────────────────────────────────────────────────────────┐
│ 시점        │ 동작                                           │
├─────────────┼───────────────────────────────────────────────-│
│ T+0초       │ IP 140.82.121.6 을 정책에 추가                  │
│ T+60초      │ TTL 만료, 새 DNS 쿼리 필요                      │
│ T+60초~     │ 새 DNS 응답이 올 때까지 기존 IP 유지 (유예 기간)  │
│             │ → tofqdns-min-ttl 설정으로 최소 TTL 보장 가능     │
└─────────────┴───────────────────────────────────────────────-┘
```

Cilium 에이전트 설정에서 최소 TTL을 지정하여 짧은 TTL로 인한 연결 끊김을 방지할 수 있음:

```yaml
# Helm values.yaml
tofqdns:
  minTTL: 3600   # 최소 1시간 동안 FQDN→IP 매핑 유지
```

#### kube-dns/CoreDNS와의 상호작용

Cilium DNS 프록시는 클러스터 DNS 서버 앞에서 투명하게 동작함. FQDN 정책 적용 시 DNS 트래픽 흐름:

```
1. FQDN 정책이 적용된 파드 → DNS 쿼리는 반드시 허용되어야 함
2. Cilium DNS 프록시가 쿼리를 인터셉트하여 응답을 기록함
3. CoreDNS/kube-dns로의 DNS 트래픽(UDP/TCP 53)은 별도로 허용 필요
```

> **주의**: FQDN 정책을 사용할 때는 반드시 DNS 트래픽(port 53)을 허용하는 규칙을 함께 포함해야 함. 그렇지 않으면 DNS 조회 자체가 차단되어 FQDN 정책이 동작하지 않음.

---

### 2.2 실무 적용 코드

#### 기본 FQDN Egress 정책

특정 파드에서 `api.github.com`으로의 이그레스만 허용하는 정책:

```yaml
# examples/fqdn-egress-basic.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-egress-to-github-api
  namespace: <NAMESPACE>
spec:
  endpointSelector:
    matchLabels:
      app: <APP_LABEL>
  egress:
    # DNS 트래픽 허용 (FQDN 정책의 필수 전제조건)
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    # 특정 FQDN으로의 HTTPS 이그레스 허용
    - toFQDNs:
        - matchName: "api.github.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

> `rules.dns`의 `matchPattern: "*"`는 모든 DNS 쿼리를 허용하되, 실제 이그레스 트래픽은 `toFQDNs`에서 지정한 도메인으로만 제한됨. DNS 쿼리 허용과 이그레스 트래픽 허용은 별개의 규칙임.

#### 와일드카드 패턴 허용 (*.amazonaws.com)

AWS 서비스 전체에 대한 이그레스를 허용하는 정책:

```yaml
# examples/fqdn-egress-wildcard.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-egress-to-aws-services
  namespace: <NAMESPACE>
spec:
  endpointSelector:
    matchLabels:
      app: <APP_LABEL>
  egress:
    # DNS 트래픽 허용
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    # AWS 서비스 전체 허용
    - toFQDNs:
        - matchPattern: "*.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

#### L4 + FQDN 결합 정책 (특정 도메인에 HTTPS만 허용)

다수의 외부 서비스에 대해 포트 레벨까지 제한하는 정책:

```yaml
# examples/fqdn-egress-l4-combined.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-egress-external-apis
  namespace: <NAMESPACE>
spec:
  endpointSelector:
    matchLabels:
      app: <APP_LABEL>
  egress:
    # DNS 트래픽 허용
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    # 결제 API: HTTPS만 허용
    - toFQDNs:
        - matchName: "api.stripe.com"
        - matchName: "api.stripe.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    # AWS S3: HTTPS만 허용
    - toFQDNs:
        - matchPattern: "*.s3.ap-northeast-2.amazonaws.com"
        - matchName: "s3.ap-northeast-2.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    # 모니터링: Datadog API
    - toFQDNs:
        - matchPattern: "*.datadoghq.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

#### Default-Deny Egress + FQDN Allowlist 패턴

프로덕션 환경에서 권장되는 패턴 — 모든 이그레스를 차단하고 필요한 도메인만 허용:

```yaml
# examples/fqdn-egress-default-deny.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny-egress
  namespace: <NAMESPACE>
spec:
  endpointSelector:
    matchLabels:
      app: <APP_LABEL>
  egress:
    # 클러스터 내부 DNS는 반드시 허용
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    # 클러스터 내부 서비스 간 통신 허용
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: <ALLOWED_NAMESPACE>
      toPorts:
        - ports:
            - port: "<SERVICE_PORT>"
              protocol: TCP
    # 허용된 외부 도메인만 이그레스 가능
    - toFQDNs:
        - matchName: "<ALLOWED_FQDN_1>"
        - matchName: "<ALLOWED_FQDN_2>"
        - matchPattern: "<ALLOWED_FQDN_PATTERN>"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

> CiliumNetworkPolicy에서 `egress` 섹션이 존재하면, 명시적으로 허용된 이그레스 트래픽만 통과됨. 별도의 deny 규칙 없이도 allowlist 방식으로 동작함.

#### DNS 가시성(Visibility) 정책

FQDN 정책 디버깅 및 모니터링을 위해 DNS 트래픽에 대한 가시성을 활성화하는 정책:

```yaml
# examples/fqdn-dns-visibility.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: dns-visibility
  namespace: <NAMESPACE>
spec:
  endpointSelector:
    matchLabels:
      app: <APP_LABEL>
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
```

위 정책 적용 후 Hubble을 통해 DNS 쿼리를 관찰할 수 있음:

```bash
# 특정 파드의 DNS 쿼리 모니터링
hubble observe --namespace <NAMESPACE> --pod <POD_NAME> --protocol DNS

# DNS 응답까지 포함하여 모니터링
hubble observe --namespace <NAMESPACE> --protocol DNS -o json | jq '.flow.l7.dns'
```

---

### 2.3 보안/성능 Best Practice

#### 1. Default-Deny Egress + Allowlist 패턴 적용

프로덕션 환경에서는 반드시 default-deny 기반으로 필요한 FQDN만 허용하는 구조를 권장함:

```
권장 구조:
├── default-deny egress (CiliumNetworkPolicy의 egress 섹션 정의)
├── DNS 허용 (kube-dns/CoreDNS로의 UDP/TCP 53)
├── 클러스터 내부 통신 허용 (필요한 서비스만)
└── 외부 FQDN allowlist (toFQDNs로 필요한 도메인만)
```

#### 2. 과도한 와일드카드 사용 지양

```yaml
# 나쁜 예: 너무 넓은 범위
toFQDNs:
  - matchPattern: "*.com"           # 모든 .com 도메인 허용 — 위험
  - matchPattern: "*"               # 모든 도메인 허용 — 사실상 정책 무효화

# 좋은 예: 필요한 범위로 제한
toFQDNs:
  - matchPattern: "*.s3.ap-northeast-2.amazonaws.com"   # 리전 한정 S3
  - matchName: "api.stripe.com"                          # 정확한 도메인
```

#### 3. DNS 캐시 고려사항

```yaml
# Cilium Helm values.yaml — DNS 관련 튜닝
tofqdns:
  minTTL: 3600          # FQDN→IP 매핑 최소 유지 시간 (초)
  preCache: ""          # 사전 캐시 파일 경로 (선택)

# DNS 프록시 설정
dnsProxy:
  dnsRejectResponseCode: "refused"   # 차단된 DNS 쿼리 응답 코드
  enableDnsCompression: true         # DNS 응답 압축
  idleConnectionGracePeriod: "0s"    # 유휴 연결 유예 기간
```

Cilium 에이전트의 `tofqdns-min-ttl` 설정으로 짧은 TTL 문제를 완화할 수 있음:

```bash
# 현재 설정 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg status | grep -i dns

# ConfigMap에서 확인
kubectl -n kube-system get configmap cilium-config -o yaml | grep tofqdns
```

#### 4. 정책 순서 및 결합 규칙

Cilium 정책은 **합집합(Union)** 방식으로 결합됨:

```
정책 A: api.github.com 허용
정책 B: api.stripe.com 허용
─────────────────────────────
결과: api.github.com AND api.stripe.com 모두 허용
```

- 동일 파드에 여러 FQDN 정책이 적용되면 모든 정책의 허용 목록이 합쳐짐
- 특정 도메인을 차단하려면 `CiliumClusterwideNetworkPolicy`의 deny 규칙 사용 (Cilium 1.14+)

---

## 3. 트러블슈팅

### 3.1 주요 이슈

#### 이슈 1: FQDN 정책이 동작하지 않음 (DNS 프록시 미인터셉트)

**증상**: FQDN 정책을 적용했으나 외부 도메인 접속이 전혀 되지 않거나, 차단해야 할 도메인이 차단되지 않음.

**원인**:
- DNS 트래픽(port 53) 허용 규칙이 누락됨
- Cilium DNS 프록시가 비활성화되어 있음
- kube-dns/CoreDNS 엔드포인트 라벨이 일치하지 않음

**해결 방법**:

```bash
# 1. DNS 프록시 활성화 여부 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg status | grep -i "dns"

# 2. kube-dns 파드의 라벨 확인
kubectl -n kube-system get pods -l k8s-app=kube-dns --show-labels

# 3. Cilium 에이전트 로그에서 DNS 프록시 관련 에러 확인
kubectl -n kube-system logs <CILIUM_POD> | grep -i "fqdn\|dns-proxy\|dnsproxy"

# 4. FQDN 정책에 DNS 허용 규칙이 포함되어 있는지 확인
kubectl get cnp <POLICY_NAME> -n <NAMESPACE> -o yaml | grep -A 10 "dns"
```

**정책 수정 예시** — DNS 허용 규칙 추가:

```yaml
egress:
  # 이 규칙이 반드시 포함되어야 함
  - toEndpoints:
      - matchLabels:
          k8s:io.kubernetes.pod.namespace: kube-system
          k8s-app: kube-dns
    toPorts:
      - ports:
          - port: "53"
            protocol: ANY
        rules:
          dns:
            - matchPattern: "*"
```

#### 이슈 2: 허용된 FQDN에 간헐적 연결 실패 (DNS TTL 만료)

**증상**: 허용된 도메인에 대해 대부분 정상 접속되지만, 간헐적으로 연결이 끊기거나 타임아웃 발생.

**원인**:
- DNS TTL이 매우 짧아(예: 5초) 캐시 만료 후 새 DNS 응답이 올 때까지 정책에 IP가 없음
- DNS 서버 응답 지연으로 캐시 갱신이 늦어짐

**해결 방법**:

```bash
# 1. FQDN 캐시 확인 — 현재 매핑된 IP와 TTL 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg fqdn cache list

# 2. 특정 도메인의 캐시 상태 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg fqdn cache list | grep "<TARGET_FQDN>"

# 3. 최소 TTL 설정 적용 (Helm)
# values.yaml에 다음 추가:
# tofqdns:
#   minTTL: 3600
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set tofqdns.minTTL=3600
```

#### 이슈 3: FQDN 정책 적용 후 파드 DNS 조회 실패

**증상**: FQDN 정책 적용 후 파드에서 모든 DNS 조회가 실패하며 `nslookup` / `dig` 결과가 타임아웃됨.

**원인**:
- 이그레스 정책에 DNS 허용 규칙이 없어서 DNS 쿼리 자체가 차단됨
- kube-dns 파드의 라벨 셀렉터가 클러스터 설정과 불일치함
- EKS 환경에서 CoreDNS 라벨이 `k8s-app: kube-dns`가 아닌 경우

**해결 방법**:

```bash
# 1. 파드에서 DNS 조회 테스트
kubectl exec -n <NAMESPACE> <POD_NAME> -- nslookup api.github.com

# 2. CoreDNS 파드의 실제 라벨 확인
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
# EKS에서 결과가 없으면 다른 라벨로 확인
kubectl -n kube-system get pods -o wide | grep -i dns

# 3. Cilium 엔드포인트에서 DNS 트래픽 드롭 여부 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg monitor --type drop | grep ":53"

# 4. Hubble로 DNS 트래픽 흐름 확인
hubble observe --namespace <NAMESPACE> --pod <POD_NAME> --port 53
```

#### 이슈 4: 와일드카드 FQDN 패턴이 과도하게 매칭됨

**증상**: `*.example.com` 패턴이 의도하지 않은 도메인까지 허용하거나, 예상한 하위 도메인이 매칭되지 않음.

**원인**:
- `*`는 단일 레벨만 매칭하므로 `a.b.example.com`은 `*.example.com`에 매칭되지 않음
- 패턴을 너무 넓게 설정하여 보안 경계가 약해짐

**해결 방법**:

```bash
# 1. 현재 FQDN 캐시에서 매칭된 도메인 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg fqdn cache list

# 2. 정책에 실제로 어떤 도메인이 매핑되었는지 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg policy get -o json | \
  jq '.[] | select(.labels[] | contains("fqdn"))'
```

**패턴 수정 예시**:

```yaml
# 단일 레벨만 매칭 (sub.example.com → O, a.sub.example.com → X)
toFQDNs:
  - matchPattern: "*.example.com"

# 2단계 하위 도메인까지 허용하려면 별도 규칙 추가
toFQDNs:
  - matchPattern: "*.example.com"
  - matchPattern: "*.*.example.com"
```

#### 이슈 5: Cilium 에이전트 재시작 후 FQDN 정책 일시 중단

**증상**: Cilium 에이전트(DaemonSet) 재시작 후 FQDN 정책이 일시적으로 동작하지 않아 외부 연결 실패.

**원인**:
- 에이전트 재시작 시 FQDN 캐시가 초기화됨
- 파드가 새로운 DNS 쿼리를 보내기 전까지 IP 매핑이 복원되지 않음

**해결 방법**:

```bash
# 1. FQDN 사전 캐시(pre-cache) 설정으로 복원 시간 단축
# Helm values.yaml:
# tofqdns:
#   preCache: "/var/run/cilium/dns-precache.json"

# 2. 에이전트 재시작 후 캐시 복원 상태 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg fqdn cache list

# 3. 수동으로 DNS 조회를 트리거하여 캐시 복원
kubectl exec -n <NAMESPACE> <POD_NAME> -- nslookup <TARGET_FQDN>
```

---

### 3.2 자주 발생하는 문제 (Q&A)

**Q1: FQDN 정책에서 `matchName`에 후행 점(trailing dot)을 붙여야 하는지?**

A: Cilium은 후행 점 유무를 모두 처리함. `matchName: "api.github.com"`과 `matchName: "api.github.com."`은 동일하게 동작함. 일관성을 위해 후행 점 없이 작성하는 것을 권장함.

---

**Q2: FQDN 정책과 CIDR 정책을 함께 사용할 수 있는지?**

A: 가능함. 동일 CiliumNetworkPolicy 내에서 `toFQDNs`와 `toCIDR`을 함께 사용할 수 있으며, 두 규칙은 합집합으로 동작함:

```yaml
egress:
  - toFQDNs:
      - matchName: "api.github.com"
  - toCIDR:
      - "10.0.0.0/8"    # 클러스터 내부 대역
```

---

**Q3: EKS 환경에서 CoreDNS로의 DNS 트래픽 허용 규칙이 동작하지 않음**

A: EKS의 CoreDNS 파드 라벨을 확인해야 함. EKS에서는 일반적으로 `k8s-app: kube-dns` 라벨을 사용하지만, 클러스터 버전에 따라 다를 수 있음:

```bash
# EKS CoreDNS 파드 라벨 확인
kubectl -n kube-system get pods -l k8s-app=kube-dns --show-labels

# 라벨이 다른 경우 정책의 toEndpoints 수정 필요
```

---

**Q4: FQDN 정책 적용 후 기존 연결(established connection)은 어떻게 되는지?**

A: 정책 적용 시점에 이미 맺어진 TCP 연결은 즉시 차단되지 않을 수 있음. Cilium은 conntrack(Connection Tracking) 테이블의 기존 엔트리를 유지함. 완전한 차단을 위해서는 파드를 재시작하거나 conntrack 엔트리가 만료될 때까지 대기해야 함:

```bash
# 특정 엔드포인트의 conntrack 상태 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg bpf ct list global | grep "<TARGET_IP>"
```

---

**Q5: `toFQDNs`를 사용하면 성능에 영향이 있는지?**

A: DNS 프록시를 통한 인터셉션으로 약간의 지연(latency)이 추가될 수 있음. 대규모 환경에서 고려할 점:

- FQDN 캐시 크기가 커지면 메모리 사용량 증가
- 대량의 DNS 쿼리가 발생하는 파드에서는 DNS 프록시에 부하 발생 가능
- `tofqdns-min-ttl`을 적절히 설정하여 불필요한 DNS 쿼리 반복을 줄이는 것을 권장함

---

## 4. 모니터링 및 확인

### Hubble을 통한 DNS 트래픽 관찰

```bash
# DNS 프로토콜 트래픽 모니터링
hubble observe --protocol DNS

# 특정 네임스페이스의 DNS 트래픽만 필터링
hubble observe --namespace <NAMESPACE> --protocol DNS

# DNS 쿼리/응답 상세 정보 (JSON 출력)
hubble observe --namespace <NAMESPACE> --protocol DNS -o json

# 특정 파드의 DNS 트래픽만 확인
hubble observe --pod <NAMESPACE>/<POD_NAME> --protocol DNS

# 드롭된 DNS 트래픽 확인 (정책 위반)
hubble observe --namespace <NAMESPACE> --protocol DNS --verdict DROPPED

# Hubble UI에서 실시간 DNS 흐름 확인
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
# 브라우저에서 http://localhost:12000 접속
```

### Cilium FQDN 캐시 확인

```bash
# 전체 FQDN 캐시 목록 조회
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg fqdn cache list

# 특정 도메인의 캐시 상태 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg fqdn cache list | grep "<TARGET_FQDN>"

# FQDN 캐시를 JSON 형식으로 확인 (TTL 포함)
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg fqdn cache list -o json

# 특정 엔드포인트의 FQDN 캐시만 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg fqdn cache list --endpoint <ENDPOINT_ID>
```

### Cilium 정책 상태 확인

```bash
# 현재 적용된 모든 정책 조회
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg policy get

# 정책을 JSON으로 출력하여 FQDN 규칙 세부사항 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg policy get -o json

# 특정 엔드포인트에 적용된 정책 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg endpoint get <ENDPOINT_ID> -o json | \
  jq '.status.policy'

# CiliumNetworkPolicy 리소스 상태 확인
kubectl get cnp -A
kubectl describe cnp <POLICY_NAME> -n <NAMESPACE>

# 정책 적용 후 드롭 카운터 확인
kubectl -n kube-system exec <CILIUM_POD> -- cilium-dbg monitor --type drop
```

### Prometheus DNS 메트릭

Cilium이 노출하는 주요 DNS 관련 Prometheus 메트릭:

| 메트릭 | 설명 |
|--------|------|
| `cilium_dns_queries_total` | 총 DNS 쿼리 수 |
| `cilium_dns_responses_total` | 총 DNS 응답 수 |
| `cilium_dns_response_types_total` | DNS 응답 유형별 카운트 (A, AAAA, CNAME 등) |
| `cilium_fqdn_gc_deletions_total` | FQDN 캐시 GC(Garbage Collection) 삭제 수 |
| `cilium_policy_l7_total` | L7 정책 판정 수 (DNS 포함) |

```bash
# Cilium 에이전트의 Prometheus 메트릭 엔드포인트 직접 조회
kubectl -n kube-system exec <CILIUM_POD> -- curl -s http://localhost:9962/metrics | grep cilium_dns

# Prometheus에서 PromQL 쿼리 예시
# DNS 쿼리 처리량 (초당)
# rate(cilium_dns_queries_total[5m])

# DNS 응답 중 에러 비율
# rate(cilium_dns_responses_total{rcode!="No Error"}[5m]) / rate(cilium_dns_responses_total[5m])
```

Grafana 대시보드 설정 시 위 메트릭을 활용하여 DNS 트래픽 패턴과 FQDN 정책 동작 상태를 시각화할 수 있음.

---

## 5. TIP

### 공식 문서 참고 링크

- [Cilium FQDN-based Policies](https://docs.cilium.io/en/v1.16/security/dns/) — DNS 기반 정책 공식 가이드
- [CiliumNetworkPolicy API Reference](https://docs.cilium.io/en/v1.16/network/kubernetes/policy/#ciliumnetworkpolicy) — 정책 스펙 레퍼런스
- [DNS Proxy Configuration](https://docs.cilium.io/en/v1.16/security/dns/#dns-proxy) — DNS 프록시 설정
- [Troubleshooting FQDN Policies](https://docs.cilium.io/en/v1.16/operations/troubleshooting/#fqdn-policies) — 공식 트러블슈팅 가이드

### AWS 서비스별 자주 사용되는 FQDN 패턴

EKS(ENI 모드) 환경에서 자주 허용해야 하는 AWS 서비스 FQDN 패턴:

| AWS 서비스 | FQDN 패턴 | 설명 |
|-----------|-----------|------|
| S3 | `s3.<REGION>.amazonaws.com` | S3 리전별 엔드포인트 |
| S3 | `<BUCKET_NAME>.s3.<REGION>.amazonaws.com` | S3 버킷 가상 호스팅 |
| ECR | `<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com` | 컨테이너 레지스트리 |
| ECR | `api.ecr.<REGION>.amazonaws.com` | ECR API |
| STS | `sts.<REGION>.amazonaws.com` | Security Token Service |
| SSM | `ssm.<REGION>.amazonaws.com` | Systems Manager |
| Secrets Manager | `secretsmanager.<REGION>.amazonaws.com` | 비밀 관리 |
| DynamoDB | `dynamodb.<REGION>.amazonaws.com` | DynamoDB |
| SQS | `sqs.<REGION>.amazonaws.com` | 메시지 큐 |
| SNS | `sns.<REGION>.amazonaws.com` | 알림 서비스 |
| CloudWatch | `monitoring.<REGION>.amazonaws.com` | 모니터링 |
| CloudWatch Logs | `logs.<REGION>.amazonaws.com` | 로그 수집 |
| KMS | `kms.<REGION>.amazonaws.com` | Key Management |

**EKS 노드/파드에서 공통적으로 필요한 FQDN**:

```yaml
# examples/fqdn-egress-eks-common.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-egress-eks-common
  namespace: <NAMESPACE>
spec:
  endpointSelector:
    matchLabels:
      app: <APP_LABEL>
  egress:
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    # ECR (컨테이너 이미지 풀)
    - toFQDNs:
        - matchName: "<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com"
        - matchName: "api.ecr.<REGION>.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    # STS (IRSA - IAM Roles for Service Accounts)
    - toFQDNs:
        - matchName: "sts.<REGION>.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    # S3 (필요 시)
    - toFQDNs:
        - matchPattern: "*.s3.<REGION>.amazonaws.com"
        - matchName: "s3.<REGION>.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

> `<REGION>`에는 EKS 클러스터가 위치한 리전(예: `ap-northeast-2`)을, `<ACCOUNT_ID>`에는 AWS 계정 ID를 입력해야 함.
