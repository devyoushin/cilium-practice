# Cilium Prometheus 메트릭 가이드

Cilium과 Hubble (Hubble)은 eBPF 기반의 풍부한 Prometheus 메트릭 (Prometheus Metrics)을 노출함.
이 문서는 메트릭 활성화, 핵심 지표, 알람 규칙, 트러블슈팅을 다룸.

---

## 1. 개요

### 메트릭이 필요한 이유

- Cilium Agent와 Hubble은 네트워킹, 정책, 엔드포인트 (Endpoint), BPF, IPAM, 암호화 등 **다양한 카테고리의 메트릭**을 Prometheus 형식으로 노출
- 프로덕션 환경에서 패킷 드롭 (Packet Drop), 정책 위반 (Policy Violation), IP 주소 고갈 (IPAM Exhaustion) 등을 **사전 감지**하려면 메트릭 기반 모니터링이 필수
- 메트릭 없이 운영하면 장애 발생 후 원인 분석에 수십 배 시간 소요

### 메트릭 엔드포인트 (Metric Endpoint)

| 컴포넌트 | 포트 | 경로 | 설명 |
|----------|------|------|------|
| Cilium Agent | `:9962` | `/metrics` | Agent 핵심 메트릭 |
| Hubble | `:9965` | `/metrics` | Hubble 플로우 메트릭 |
| Cilium Operator | `:9963` | `/metrics` | Operator 메트릭 |
| Hubble Relay | `:9966` | `/metrics` | Relay 상태 메트릭 |

---

## 2. 설명

### 2.1 핵심 개념

#### Cilium Agent 메트릭 vs Hubble 메트릭

| 구분 | Cilium Agent 메트릭 | Hubble 메트릭 |
|------|---------------------|---------------|
| 수집 주체 | cilium-agent 프로세스 | hubble-server (agent 내장) |
| 포트 | `:9962` | `:9965` |
| 관점 | 인프라 수준 (BPF, IPAM, 엔드포인트) | 트래픽 수준 (플로우, DNS, HTTP) |
| 카디널리티 (Cardinality) | 상대적으로 낮음 | 활성화 메트릭에 따라 매우 높을 수 있음 |
| 주 용도 | Agent 상태, 리소스 소비 모니터링 | 애플리케이션 트래픽 분석 |

#### 메트릭 카테고리 (Metric Categories)

| 카테고리 | 설명 | 대표 메트릭 |
|----------|------|-------------|
| 네트워킹 (Networking) | 패킷 전달/드롭 통계 | `cilium_forward_count_total` |
| 정책 (Policy) | 정책 판정 결과 | `cilium_policy_verdict` |
| 엔드포인트 (Endpoint) | Pod 엔드포인트 상태 | `cilium_endpoint_state` |
| BPF | BPF 맵 연산/에러 | `cilium_bpf_map_ops_total` |
| IPAM | IP 주소 할당 | `cilium_ipam_available` |
| 암호화 (Encryption) | WireGuard/IPsec 통계 | `cilium_wireguard_interfaces` |

---

### 2.2 실무 적용 코드

#### Helm Values로 Prometheus 메트릭 활성화

```yaml
# helm/values-prometheus.yaml
prometheus:
  enabled: true
  port: 9962
  serviceMonitor:
    enabled: true
    labels:
      release: <PROMETHEUS_RELEASE_NAME>    # Prometheus Operator의 release label
    interval: "15s"
    relabelings: []
    metricRelabelings: []

operator:
  prometheus:
    enabled: true
    port: 9963
    serviceMonitor:
      enabled: true
      labels:
        release: <PROMETHEUS_RELEASE_NAME>

hubble:
  enabled: true
  metrics:
    enabled:
      - dns:query;ignoreAAAA
      - drop
      - tcp
      - flow
      - icmp
      - httpV2:exemplars=true;labelsContext=source_namespace,destination_namespace
    serviceMonitor:
      enabled: true
      labels:
        release: <PROMETHEUS_RELEASE_NAME>
      interval: "15s"
  relay:
    enabled: true
    prometheus:
      enabled: true
      port: 9966
      serviceMonitor:
        enabled: true
```

```bash
# Helm 업그레이드로 메트릭 활성화 적용
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  -f helm/values-prometheus.yaml
```

#### ServiceMonitor YAML (수동 생성 시)

Prometheus Operator를 사용하지만 Helm의 `serviceMonitor.enabled`를 쓰지 않는 경우, 수동으로 ServiceMonitor를 생성함.

```yaml
# manifests/cilium-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cilium-agent
  namespace: kube-system
  labels:
    release: <PROMETHEUS_RELEASE_NAME>
spec:
  selector:
    matchLabels:
      k8s-app: cilium
  namespaceSelector:
    matchNames:
      - kube-system
  endpoints:
    - port: prometheus
      interval: 15s
      path: /metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: hubble-metrics
  namespace: kube-system
  labels:
    release: <PROMETHEUS_RELEASE_NAME>
spec:
  selector:
    matchLabels:
      k8s-app: hubble
  namespaceSelector:
    matchNames:
      - kube-system
  endpoints:
    - port: hubble-metrics
      interval: 15s
      path: /metrics
```

```bash
kubectl apply -f manifests/cilium-servicemonitor.yaml
```

#### 핵심 Cilium Agent 메트릭

| 메트릭 | 타입 | 설명 | 알람 기준 |
|--------|------|------|-----------|
| `cilium_endpoint_state` | Gauge | 상태별 엔드포인트 수 (ready, not-ready, disconnecting 등) | `not-ready > 0` 이 5분 이상 지속 |
| `cilium_policy_verdict` | Counter | 정책 판정 결과 (forwarded, dropped, denied) | `dropped` 비율 급증 (5분간 50% 이상 증가) |
| `cilium_drop_count_total` | Counter | 드롭된 패킷 수 (reason 레이블로 원인 구분) | 분당 100 이상 지속 |
| `cilium_drop_bytes_total` | Counter | 드롭된 바이트 수 | `drop_count`와 함께 분석 |
| `cilium_forward_count_total` | Counter | 전달된 패킷 수 (direction: ingress/egress) | 갑작스런 0 전환 |
| `cilium_forward_bytes_total` | Counter | 전달된 바이트 수 | 트래픽 패턴 분석용 |
| `cilium_bpf_map_ops_total` | Counter | BPF 맵 연산 수 (operation, outcome 레이블) | `outcome=fail` 발생 시 |
| `cilium_bpf_map_pressure` | Gauge | BPF 맵 사용률 (0~1) | `> 0.9` 시 긴급 알람 |
| `cilium_ipam_available` | Gauge | 사용 가능한 IP 주소 수 | `< 10` 시 경고, `< 3` 시 긴급 |
| `cilium_ipam_used` | Gauge | 사용 중인 IP 주소 수 | 전체 대비 90% 초과 시 |
| `cilium_agent_bootstrap_seconds` | Histogram | Agent 부트스트랩 소요 시간 | `> 60s` 시 조사 필요 |
| `cilium_datapath_errors_total` | Counter | 데이터패스 에러 수 | `> 0` 지속 시 |
| `cilium_policy_import_errors_total` | Counter | 정책 임포트 에러 수 | `> 0` 발생 즉시 |
| `cilium_identity_count` | Gauge | 현재 Identity 수 | `> 60000` 시 경고 (기본 한도: 65535) |
| `cilium_unreachable_health_endpoints` | Gauge | 헬스체크 실패 엔드포인트 수 | `> 0` 지속 시 |
| `cilium_controllers_failing` | Gauge | 실패 상태 컨트롤러 수 | `> 0` 지속 시 |
| `cilium_k8s_client_api_calls_total` | Counter | K8s API 호출 수 (method, return_code 레이블) | `5xx` 비율 증가 시 |

**드롭 원인 (Drop Reason) 주요 레이블 값:**

| reason 레이블 값 | 의미 |
|------------------|------|
| `Policy denied` | 네트워크 정책에 의한 차단 |
| `Invalid source mac` | 잘못된 소스 MAC |
| `Invalid destination mac` | 잘못된 목적지 MAC |
| `Unknown L3 target address` | 알 수 없는 L3 대상 |
| `Authentication required` | 인증 미완료 |
| `No tunnel/encapsulation endpoint` | 터널 엔드포인트 없음 |
| `CT: Map insertion failed` | 연결 추적 테이블 삽입 실패 |

#### 핵심 Hubble 메트릭

| 메트릭 | 타입 | 설명 | 알람 기준 |
|--------|------|------|-----------|
| `hubble_flows_processed_total` | Counter | Hubble이 처리한 총 플로우 수 (type, verdict 레이블) | 처리량 갑작스런 저하 |
| `hubble_dns_queries_total` | Counter | DNS 쿼리 수 (rcode, qtypes 레이블) | `NXDOMAIN` 급증 |
| `hubble_dns_responses_total` | Counter | DNS 응답 수 (rcode 레이블) | `SERVFAIL` 지속 |
| `hubble_drop_total` | Counter | 드롭 플로우 수 (reason, protocol 레이블) | 분당 급증 |
| `hubble_tcp_flags_total` | Counter | TCP 플래그별 패킷 수 (flag, family 레이블) | `RST` 급증 시 연결 문제 |
| `hubble_http_requests_total` | Counter | HTTP 요청 수 (method, status, reporter 레이블) | `5xx` 비율 > 5% |
| `hubble_http_request_duration_seconds` | Histogram | HTTP 요청 지연 시간 | p99 > 5s |
| `hubble_icmp_total` | Counter | ICMP 메시지 수 (type, family 레이블) | 비정상 패턴 감지 |
| `hubble_port_distribution_total` | Counter | 포트별 트래픽 분포 | 비인가 포트 사용 감지 |

#### PromQL 알람 규칙 (PrometheusRule CRD)

```yaml
# manifests/cilium-prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cilium-alerts
  namespace: kube-system
  labels:
    release: <PROMETHEUS_RELEASE_NAME>
spec:
  groups:
    # ── 높은 드롭률 (High Drop Rate) ──
    - name: cilium-drop-alerts
      rules:
        - alert: CiliumHighDropRate
          expr: |
            sum(rate(cilium_drop_count_total[5m])) by (reason)
            / on() sum(rate(cilium_forward_count_total[5m])) > 0.01
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Cilium 드롭률이 1%를 초과"
            description: "드롭 원인: {{ $labels.reason }}, 현재 비율: {{ $value | humanizePercentage }}"
            runbook_url: "https://docs.cilium.io/en/stable/observability/metrics/"

        - alert: CiliumCriticalDropRate
          expr: |
            sum(rate(cilium_drop_count_total[5m])) by (reason)
            / on() sum(rate(cilium_forward_count_total[5m])) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Cilium 드롭률이 5%를 초과 — 즉시 조사 필요"
            description: "드롭 원인: {{ $labels.reason }}, 현재 비율: {{ $value | humanizePercentage }}"

    # ── 엔드포인트 미준비 (Endpoint Not Ready) ──
    - name: cilium-endpoint-alerts
      rules:
        - alert: CiliumEndpointNotReady
          expr: |
            cilium_endpoint_state{endpoint_state="not-ready"} > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Cilium 엔드포인트가 not-ready 상태"
            description: "노드 {{ $labels.instance }}에서 {{ $value }}개 엔드포인트가 5분 이상 not-ready"

        - alert: CiliumEndpointRegenerationFailure
          expr: |
            rate(cilium_endpoint_regenerations_total{outcome="fail"}[5m]) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Cilium 엔드포인트 재생성 실패 발생"

    # ── IPAM 고갈 (IPAM Exhaustion) ──
    - name: cilium-ipam-alerts
      rules:
        - alert: CiliumIPAMExhaustionWarning
          expr: |
            cilium_ipam_available < 10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "IPAM 가용 IP 부족 (10개 미만)"
            description: "노드 {{ $labels.instance }}에서 가용 IP {{ $value }}개"

        - alert: CiliumIPAMExhaustionCritical
          expr: |
            cilium_ipam_available < 3
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "IPAM 가용 IP 거의 고갈 (3개 미만) — 새 Pod 스케줄링 불가 위험"

    # ── Agent 재시작 (Agent Restart) ──
    - name: cilium-agent-alerts
      rules:
        - alert: CiliumAgentRestart
          expr: |
            changes(process_start_time_seconds{k8s_app="cilium"}[15m]) > 1
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: "Cilium Agent가 15분 내 2회 이상 재시작"
            description: "노드 {{ $labels.instance }}에서 Agent 반복 재시작 — CrashLoopBackOff 가능성"

        - alert: CiliumAgentBootstrapSlow
          expr: |
            histogram_quantile(0.99,
              rate(cilium_agent_bootstrap_seconds_bucket[10m])
            ) > 60
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Cilium Agent 부트스트랩이 60초 이상 소요"

    # ── BPF 맵 압력 (BPF Map Pressure) ──
    - name: cilium-bpf-alerts
      rules:
        - alert: CiliumBPFMapPressure
          expr: |
            cilium_bpf_map_pressure > 0.9
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "BPF 맵 사용률이 90%를 초과"
            description: "맵: {{ $labels.map_name }}, 사용률: {{ $value | humanizePercentage }}"

    # ── 정책 임포트 에러 ──
    - name: cilium-policy-alerts
      rules:
        - alert: CiliumPolicyImportError
          expr: |
            rate(cilium_policy_import_errors_total[5m]) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Cilium 정책 임포트 에러 발생"

    # ── Hubble DNS 에러 ──
    - name: hubble-alerts
      rules:
        - alert: HubbleDNSErrorRate
          expr: |
            sum(rate(hubble_dns_responses_total{rcode!="No Error"}[5m]))
            / sum(rate(hubble_dns_responses_total[5m])) > 0.05
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "DNS 에러 비율이 5%를 초과"

        - alert: HubbleHighHTTP5xxRate
          expr: |
            sum(rate(hubble_http_requests_total{status=~"5.."}[5m]))
            / sum(rate(hubble_http_requests_total[5m])) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "HTTP 5xx 에러 비율이 5%를 초과"
```

```bash
kubectl apply -f manifests/cilium-prometheus-rules.yaml
```

---

### 2.3 보안/성능 Best Practice

#### 카디널리티 (Cardinality) 관리

Hubble 메트릭은 `labelsContext`에 따라 카디널리티가 폭발적으로 증가할 수 있음. 반드시 필요한 메트릭만 활성화할 것.

```yaml
# 권장: 필요한 메트릭만 선별 활성화
hubble:
  metrics:
    enabled:
      - dns:query;ignoreAAAA                # AAAA 쿼리 제외로 카디널리티 절감
      - drop
      - tcp
      - flow
      # - httpV2:exemplars=true             # HTTP 메트릭은 필요 시에만 활성화
```

**비권장 구성 (카디널리티 폭발 위험):**

```yaml
# 주의: 아래 구성은 카디널리티가 매우 높아짐
hubble:
  metrics:
    enabled:
      - httpV2:exemplars=true;labelsContext=source_ip,destination_ip,source_pod,destination_pod
      # source_ip × destination_ip 조합으로 시계열 수 기하급수 증가
```

#### 릴레이블링 (Relabeling)으로 메트릭 볼륨 제어

```yaml
prometheus:
  enabled: true
  serviceMonitor:
    enabled: true
    metricRelabelings:
      # 불필요한 메트릭 제거
      - sourceLabels: [__name__]
        regex: "cilium_(.+)_bucket"
        action: drop
      # 특정 메트릭만 유지
      - sourceLabels: [__name__]
        regex: "cilium_(endpoint_state|drop_count_total|forward_count_total|policy_verdict|ipam_available|bpf_map_pressure)"
        action: keep
```

#### 저장 및 리텐션 (Retention) 고려사항

| 항목 | 권장 설정 | 비고 |
|------|-----------|------|
| 스크레이프 간격 (Scrape Interval) | 15~30초 | 10초 미만은 Agent 부하 유발 가능 |
| 메트릭 리텐션 (Retention) | 15~30일 | 장기 보관은 Thanos/Cortex 활용 |
| 디스크 여유 공간 | Hubble 메트릭 전체 활성화 시 노드당 ~50MB/day 증가 추정 | 클러스터 규모에 따라 차이 |

---

## 3. 트러블슈팅

### 3.1 주요 이슈

#### 이슈 1: 메트릭 엔드포인트 응답 없음 (Metrics Endpoint Not Responding)

##### 증상
- Prometheus 타겟이 `DOWN` 상태
- `curl` 요청 시 연결 거부 또는 타임아웃

##### 원인
- Cilium Agent의 `prometheus.enabled`가 `false`
- 방화벽 또는 네트워크 정책이 메트릭 포트 차단
- Cilium Agent Pod가 비정상 상태

##### 해결 방법
```bash
# 1. Cilium Agent 상태 확인
kubectl -n kube-system get pods -l k8s-app=cilium

# 2. Cilium 설정에서 Prometheus 활성화 여부 확인
kubectl -n kube-system get cm cilium-config -o yaml | grep prometheus

# 3. Agent Pod에서 직접 메트릭 엔드포인트 테스트
kubectl -n kube-system exec ds/cilium -- curl -s http://localhost:9962/metrics | head -20

# 4. prometheus.enabled=true가 아닌 경우 Helm 업그레이드
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set prometheus.enabled=true
```

---

#### 이슈 2: Hubble 메트릭 누락 (Missing Hubble Metrics)

##### 증상
- Cilium Agent 메트릭은 정상이지만 Hubble 메트릭 (`hubble_*`)이 조회되지 않음
- Prometheus에서 `hubble_flows_processed_total` 등이 없음

##### 원인
- Hubble이 비활성화 상태
- `hubble.metrics.enabled` 설정이 비어 있음
- Hubble 메트릭 전용 Service가 생성되지 않음

##### 해결 방법
```bash
# 1. Hubble 활성화 상태 확인
cilium status | grep Hubble

# 2. Hubble 메트릭 설정 확인
kubectl -n kube-system get cm cilium-config -o yaml | grep -A 10 "hubble"

# 3. Hubble 메트릭 Service 존재 확인
kubectl -n kube-system get svc hubble-metrics

# 4. Hubble 메트릭 직접 확인
kubectl -n kube-system exec ds/cilium -- curl -s http://localhost:9965/metrics | head -20

# 5. Hubble 메트릭 활성화 (없는 경우)
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set hubble.enabled=true \
  --set hubble.metrics.enabled="{dns:query,drop,tcp,flow,icmp}"
```

---

#### 이슈 3: 높은 카디널리티로 Prometheus OOM (High Cardinality Causing Prometheus OOM)

##### 증상
- Prometheus Pod가 `OOMKilled`로 반복 재시작
- Prometheus 메모리 사용량이 지속적으로 증가
- `prometheus_tsdb_symbol_table_size_bytes`가 비정상적으로 큼

##### 원인
- Hubble `httpV2` 메트릭에 `labelsContext`로 고카디널리티 레이블 추가
- `source_ip`, `destination_ip` 등 Pod IP 기반 레이블 사용
- 대규모 클러스터에서 메트릭 시계열 (Time Series) 수 폭발

##### 해결 방법
```bash
# 1. 현재 시계열 수 확인
kubectl -n <PROMETHEUS_NAMESPACE> exec <PROMETHEUS_POD> -- \
  promtool tsdb analyze /prometheus

# 2. 카디널리티가 높은 메트릭 확인 (Prometheus UI에서)
# PromQL: topk(10, count by (__name__)({__name__=~"hubble_.*"}))

# 3. Hubble 메트릭에서 고카디널리티 레이블 제거
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set hubble.metrics.enabled="{dns:query;ignoreAAAA,drop,tcp,flow}"

# 4. metricRelabelings으로 불필요 레이블 제거 (ServiceMonitor)
kubectl -n kube-system edit servicemonitor hubble-metrics
# metricRelabelings 섹션에 불필요 레이블 drop 추가

# 5. Prometheus 메모리 제한 조정 (임시 대응)
kubectl -n <PROMETHEUS_NAMESPACE> edit statefulset prometheus-<NAME>
# resources.limits.memory 증가
```

---

#### 이슈 4: ServiceMonitor가 Prometheus에 인식되지 않음

##### 증상
- ServiceMonitor를 생성했지만 Prometheus 타겟 목록에 나타나지 않음

##### 원인
- ServiceMonitor의 `labels`가 Prometheus Operator의 `serviceMonitorSelector`와 불일치
- ServiceMonitor의 `namespaceSelector`가 올바르지 않음

##### 해결 방법
```bash
# 1. Prometheus Operator가 기대하는 label 확인
kubectl -n <PROMETHEUS_NAMESPACE> get prometheus -o yaml | grep -A 5 serviceMonitorSelector

# 2. ServiceMonitor label 확인 및 수정
kubectl -n kube-system get servicemonitor cilium-agent -o yaml | grep -A 3 labels

# 3. label 일치시키기
kubectl -n kube-system label servicemonitor cilium-agent release=<PROMETHEUS_RELEASE_NAME>
```

---

### 3.2 자주 발생하는 문제 (Q&A)

**Q1: Cilium Agent 메트릭 포트(9962)를 변경할 수 있는가?**

A: Helm values에서 `prometheus.port` 값을 변경하면 됨.

```yaml
prometheus:
  enabled: true
  port: 9090    # 기본값 9962에서 변경
```

---

**Q2: 특정 네임스페이스의 Hubble 메트릭만 수집하고 싶다면?**

A: Hubble 메트릭은 네임스페이스 필터링을 직접 지원하지 않음. `metricRelabelings`에서 `source_namespace` 레이블 기반으로 필터링 가능.

```yaml
hubble:
  metrics:
    enabled:
      - httpV2:labelsContext=source_namespace,destination_namespace
    serviceMonitor:
      metricRelabelings:
        - sourceLabels: [source_namespace]
          regex: "(default|production)"
          action: keep
```

---

**Q3: Grafana 대시보드는 어디서 얻을 수 있는가?**

A: Cilium 공식 Grafana 대시보드를 사용:

```bash
# Grafana 대시보드 ID
# - Cilium Agent: 16611
# - Hubble: 16612
# - Cilium Operator: 16613

# 또는 Helm으로 자동 ConfigMap 생성
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set prometheus.enabled=true \
  --set dashboards.enabled=true
```

---

**Q4: ENI 모드에서 IPAM 메트릭이 중요한 이유는?**

A: ENI 모드 (AWS VPC 네이티브)에서는 각 노드에 할당된 ENI의 IP 슬롯이 유한함. `cilium_ipam_available`이 0에 도달하면 새 Pod 스케줄링이 불가하므로 사전 알람 설정이 필수.

```bash
# 노드별 IPAM 상태 확인
kubectl -n kube-system exec ds/cilium -- cilium status | grep IPAM
```

---

## 4. 모니터링 및 확인

### 메트릭 엔드포인트 직접 확인

```bash
# Cilium Agent 메트릭 엔드포인트 확인 (port-forward)
kubectl -n kube-system port-forward svc/cilium-agent 9962:9962 &
curl -s http://localhost:9962/metrics | grep cilium_endpoint_state

# Hubble 메트릭 엔드포인트 확인
kubectl -n kube-system port-forward svc/hubble-metrics 9965:9965 &
curl -s http://localhost:9965/metrics | grep hubble_flows_processed_total

# Cilium Agent Pod에서 직접 확인 (port-forward 없이)
kubectl -n kube-system exec ds/cilium -- curl -s http://localhost:9962/metrics | grep -c "^cilium_"
kubectl -n kube-system exec ds/cilium -- curl -s http://localhost:9965/metrics | grep -c "^hubble_"
```

### 특정 메트릭 조회

```bash
# 드롭 카운트 (원인별)
kubectl -n kube-system exec ds/cilium -- \
  curl -s http://localhost:9962/metrics | grep "cilium_drop_count_total"

# IPAM 가용 IP 확인
kubectl -n kube-system exec ds/cilium -- \
  curl -s http://localhost:9962/metrics | grep "cilium_ipam_available"

# BPF 맵 압력 확인
kubectl -n kube-system exec ds/cilium -- \
  curl -s http://localhost:9962/metrics | grep "cilium_bpf_map_pressure"

# 엔드포인트 상태
kubectl -n kube-system exec ds/cilium -- \
  curl -s http://localhost:9962/metrics | grep "cilium_endpoint_state"
```

### Prometheus 타겟 상태 확인

```bash
# Prometheus 타겟 상태 확인 (Prometheus UI port-forward)
kubectl -n <PROMETHEUS_NAMESPACE> port-forward svc/prometheus-operated 9090:9090 &
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -A 5 "cilium"

# ServiceMonitor 상태 확인
kubectl -n kube-system get servicemonitor
kubectl -n kube-system describe servicemonitor cilium-agent
```

### Cilium CLI 기반 진단

```bash
# Cilium 전체 상태 확인
cilium status --wait

# 엔드포인트 목록 확인
cilium endpoint list

# Hubble 상태 확인
cilium hubble port-forward &
hubble status
```

---

## 5. TIP

### 공식 참고 자료

- Cilium Metrics 공식 문서: <https://docs.cilium.io/en/v1.16/observability/metrics/>
- Hubble Metrics 공식 문서: <https://docs.cilium.io/en/v1.16/observability/hubble/configuration/metrics/>
- Cilium Grafana 대시보드: <https://grafana.com/grafana/dashboards/16611-cilium-metrics/>
- Prometheus Operator ServiceMonitor 스펙: <https://prometheus-operator.dev/docs/api-reference/api/#monitoring.coreos.com/v1.ServiceMonitor>

### 프로덕션 권장 알람 목록

| 알람 | 심각도 | 설명 |
|------|--------|------|
| `CiliumHighDropRate` | Warning | 드롭률 1% 초과 시 |
| `CiliumCriticalDropRate` | Critical | 드롭률 5% 초과 시 |
| `CiliumEndpointNotReady` | Warning | 엔드포인트 5분 이상 not-ready |
| `CiliumIPAMExhaustionWarning` | Warning | 가용 IP 10개 미만 |
| `CiliumIPAMExhaustionCritical` | Critical | 가용 IP 3개 미만 |
| `CiliumAgentRestart` | Critical | 15분 내 2회 이상 재시작 |
| `CiliumBPFMapPressure` | Critical | BPF 맵 사용률 90% 초과 |
| `CiliumPolicyImportError` | Warning | 정책 임포트 에러 발생 |
| `HubbleDNSErrorRate` | Warning | DNS 에러 비율 5% 초과 |
| `HubbleHighHTTP5xxRate` | Warning | HTTP 5xx 비율 5% 초과 |

### 유용한 PromQL 쿼리 모음

```promql
# 노드별 드롭률 (Top 5)
topk(5,
  sum(rate(cilium_drop_count_total[5m])) by (instance)
)

# 정책 차단 트래픽 추이
sum(rate(cilium_policy_verdict{forwarded="false"}[5m])) by (namespace)

# 노드별 IPAM 사용률
cilium_ipam_used / (cilium_ipam_used + cilium_ipam_available) * 100

# Hubble 프로토콜별 플로우 분포
sum(rate(hubble_flows_processed_total[5m])) by (protocol)

# DNS NXDOMAIN 비율
sum(rate(hubble_dns_responses_total{rcode="Non-Existent Domain"}[5m]))
/ sum(rate(hubble_dns_responses_total[5m]))
```
