# Grafana 대시보드 가이드

## 1. 개요

Grafana(그라파나)는 Cilium과 Hubble(허블)이 수집하는 메트릭(Metrics)을 시각화하기 위한 오픈소스 대시보드 도구임. Cilium 프로젝트는 공식 Grafana 대시보드를 제공하며, 이를 통해 네트워크 정책 적용 현황, 엔드포인트(Endpoint) 상태, DNS 쿼리, HTTP 트래픽 등을 한눈에 파악할 수 있음.

### 공식 Cilium Grafana 대시보드 구성

| 대시보드 | Grafana.com ID | 설명 |
|----------|----------------|------|
| Cilium Agent | `16611` | Agent 핵심 지표: 엔드포인트, BPF, 정책 등 |
| Cilium Operator | `16612` | Operator 상태: IPAM, CRD 처리 등 |
| Hubble Overview | `16613` | 네트워크 플로우(Flow) 전체 현황 |
| Hubble DNS | `16614` | DNS 쿼리 성능 및 오류 |
| Hubble HTTP | `16615` | L7 HTTP 요청/응답 메트릭 |

### 데이터 흐름

```
Cilium Agent / Hubble ──(메트릭 노출)──▶ Prometheus ──(쿼리)──▶ Grafana
       :9962 / :9965                     scrape & store         시각화 & 알림
```

---

## 2. 설명

### 2.1 핵심 개념

#### 대시보드 카테고리별 커버 범위

**Cilium Agent 대시보드 (ID: 16611)**
- 엔드포인트 생성/삭제/재생성 비율
- BPF 맵(Map) 사용률 및 오류
- 네트워크 정책(Network Policy) 적용 횟수와 드롭(Drop) 비율
- API 요청 처리 레이턴시(Latency)
- Datapath(데이터패스) 커넥티비티(Connectivity) 상태

**Cilium Operator 대시보드 (ID: 16612)**
- IPAM(IP Address Management) 할당 현황
- CRD(Custom Resource Definition) 처리 큐 길이
- Operator 리더 선출(Leader Election) 상태
- AWS ENI 할당 및 가용 IP 추이

**Hubble Overview 대시보드 (ID: 16613)**
- 전체 네트워크 플로우 처리량(Throughput)
- 포워드(Forward)/드롭(Drop) 비율
- 소스/목적지별 트래픽 분포

**Hubble DNS 대시보드 (ID: 16614)**
- DNS 쿼리 레이턴시 분포
- DNS 응답 코드(RCODE)별 비율
- 쿼리 타입(A, AAAA, CNAME 등) 분포

**Hubble HTTP 대시보드 (ID: 16615)**
- HTTP 요청 비율(Request Rate)
- 응답 코드(Status Code)별 분포
- 요청 지속 시간(Duration) 히스토그램

#### 데이터 흐름 상세

Cilium Agent는 `:9962` 포트에서 Prometheus 형식의 메트릭을 노출함. Hubble은 `:9965` 포트를 사용함. Prometheus가 이 엔드포인트를 스크레이프(Scrape)하여 시계열 데이터(Time Series Data)를 저장하고, Grafana가 PromQL(Prometheus Query Language)로 질의하여 시각화하는 구조임.

---

### 2.2 실무 적용 코드

#### Grafana Helm 설치

```bash
# Grafana Helm 리포지터리 추가
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Grafana 설치 (Prometheus 데이터 소스 자동 설정 포함)
helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set adminUser=admin \
  --set adminPassword=<GRAFANA_ADMIN_PASSWORD> \
  --set "datasources.datasources\\.yaml.apiVersion=1" \
  --set "datasources.datasources\\.yaml.datasources[0].name=Prometheus" \
  --set "datasources.datasources\\.yaml.datasources[0].type=prometheus" \
  --set "datasources.datasources\\.yaml.datasources[0].url=http://prometheus-server.monitoring.svc.cluster.local" \
  --set "datasources.datasources\\.yaml.datasources[0].access=proxy" \
  --set "datasources.datasources\\.yaml.datasources[0].isDefault=true"
```

#### Prometheus 데이터 소스 설정 (values.yaml 방식)

```yaml
# grafana-values.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-server.monitoring.svc.cluster.local
        access: proxy
        isDefault: true
        jsonData:
          timeInterval: "15s"
          httpMethod: POST
```

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  -f grafana-values.yaml
```

#### Prometheus에서 Cilium 메트릭 스크레이프 설정

```yaml
# prometheus-values.yaml (prometheus-community/prometheus Helm 차트 기준)
serverFiles:
  prometheus.yml:
    scrape_configs:
      # Cilium Agent 메트릭
      - job_name: "cilium-agent"
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - kube-system
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_k8s_app]
            action: keep
            regex: cilium
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: (.+)
            replacement: "${1}:9962"
        metric_relabel_configs:
          - source_labels: [__name__]
            regex: "cilium_.*"
            action: keep

      # Hubble 메트릭
      - job_name: "hubble"
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - kube-system
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_k8s_app]
            action: keep
            regex: cilium
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: (.+)
            replacement: "${1}:9965"

      # Cilium Operator 메트릭
      - job_name: "cilium-operator"
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - kube-system
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_name]
            action: keep
            regex: cilium-operator
          - source_labels: [__address__]
            action: replace
            target_label: __address__
            regex: (.+)
            replacement: "${1}:9963"
```

#### Cilium Helm 차트에서 메트릭 활성화

```bash
# Cilium 설치 시 Prometheus 메트릭과 Hubble 메트릭 활성화
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --set hubble.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"
```

#### 공식 대시보드 임포트 (CLI 방식)

```bash
# Grafana 포트포워드
kubectl port-forward svc/grafana 3000:80 -n monitoring &

# Grafana API를 사용한 대시보드 임포트
GRAFANA_URL="http://localhost:3000"
GRAFANA_AUTH="admin:<GRAFANA_ADMIN_PASSWORD>"

# 대시보드 ID 목록
DASHBOARD_IDS=("16611" "16612" "16613" "16614" "16615")

for DASHBOARD_ID in "${DASHBOARD_IDS[@]}"; do
  # Grafana.com에서 대시보드 JSON 다운로드
  DASHBOARD_JSON=$(curl -s "https://grafana.com/api/dashboards/${DASHBOARD_ID}/revisions/latest/download")

  # Grafana에 임포트
  curl -s -X POST "${GRAFANA_URL}/api/dashboards/import" \
    -u "${GRAFANA_AUTH}" \
    -H "Content-Type: application/json" \
    -d "{
      \"dashboard\": ${DASHBOARD_JSON},
      \"overwrite\": true,
      \"inputs\": [{
        \"name\": \"DS_PROMETHEUS\",
        \"type\": \"datasource\",
        \"pluginId\": \"prometheus\",
        \"value\": \"Prometheus\"
      }],
      \"folderId\": 0
    }"
  echo " → Dashboard ${DASHBOARD_ID} imported"
done
```

#### 커스텀 대시보드 패널: 유용한 PromQL 쿼리

**네트워크 정책 드롭 비율(Network Policy Drop Rate) 패널**

```promql
# 초당 네트워크 정책에 의한 드롭 비율
sum(rate(cilium_drop_count_total{reason="Policy denied"}[5m])) by (direction)
```

```json
{
  "title": "Network Policy Drop Rate",
  "type": "timeseries",
  "targets": [
    {
      "expr": "sum(rate(cilium_drop_count_total{reason=\"Policy denied\"}[5m])) by (direction)",
      "legendFormat": "{{direction}}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "pps",
      "thresholds": {
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 10 },
          { "color": "red", "value": 100 }
        ]
      }
    }
  }
}
```

**엔드포인트 헬스 개요(Endpoint Health Overview) 패널**

```promql
# 엔드포인트 상태별 개수
sum(cilium_endpoint_state) by (endpoint_state)

# 비정상 엔드포인트 비율
sum(cilium_endpoint_state{endpoint_state!="ready"}) /
sum(cilium_endpoint_state) * 100
```

```json
{
  "title": "Endpoint Health Overview",
  "type": "piechart",
  "targets": [
    {
      "expr": "sum(cilium_endpoint_state) by (endpoint_state)",
      "legendFormat": "{{endpoint_state}}"
    }
  ]
}
```

**DNS 쿼리 레이턴시(DNS Query Latency) 패널**

```promql
# DNS 쿼리 레이턴시 p50, p95, p99
histogram_quantile(0.50, sum(rate(hubble_dns_response_types_total[5m])) by (le))
histogram_quantile(0.95, sum(rate(hubble_dns_responses_total_bucket[5m])) by (le))
histogram_quantile(0.99, sum(rate(hubble_dns_responses_total_bucket[5m])) by (le))

# 평균 DNS 레이턴시
sum(rate(hubble_dns_response_types_total[5m])) by (qtypes, rcode)
```

```json
{
  "title": "DNS Query Latency",
  "type": "timeseries",
  "targets": [
    {
      "expr": "histogram_quantile(0.95, sum(rate(hubble_dns_responses_total_bucket[5m])) by (le))",
      "legendFormat": "p95"
    },
    {
      "expr": "histogram_quantile(0.99, sum(rate(hubble_dns_responses_total_bucket[5m])) by (le))",
      "legendFormat": "p99"
    }
  ],
  "fieldConfig": {
    "defaults": { "unit": "s" }
  }
}
```

**HTTP 요청 비율 및 오류 비율(HTTP Request Rate and Error Rate) 패널**

```promql
# HTTP 요청 비율 (Hubble L7 메트릭)
sum(rate(hubble_http_requests_total[5m])) by (method, protocol)

# HTTP 오류 비율 (4xx + 5xx)
sum(rate(hubble_http_requests_total{status=~"4..|5.."}[5m]))
/
sum(rate(hubble_http_requests_total[5m])) * 100

# HTTP 레이턴시 p95
histogram_quantile(0.95,
  sum(rate(hubble_http_request_duration_seconds_bucket[5m])) by (le, method)
)
```

```json
{
  "title": "HTTP Error Rate (%)",
  "type": "stat",
  "targets": [
    {
      "expr": "sum(rate(hubble_http_requests_total{status=~\"4..|5..\"}[5m])) / sum(rate(hubble_http_requests_total[5m])) * 100",
      "legendFormat": "Error Rate"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "thresholds": {
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 1 },
          { "color": "red", "value": 5 }
        ]
      }
    }
  }
}
```

**BPF 맵 사용률(BPF Map Utilization) 패널**

```promql
# BPF 맵 사용률
cilium_bpf_map_pressure{}

# 임계치 초과 BPF 맵 감지
cilium_bpf_map_pressure{} > 0.9
```

```json
{
  "title": "BPF Map Utilization",
  "type": "gauge",
  "targets": [
    {
      "expr": "cilium_bpf_map_pressure{}",
      "legendFormat": "{{map_name}}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percentunit",
      "min": 0,
      "max": 1,
      "thresholds": {
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 0.7 },
          { "color": "red", "value": 0.9 }
        ]
      }
    }
  }
}
```

**IPAM 가용 IP 추이(IPAM Available IPs Trend) 패널**

```promql
# 노드별 가용 IP 수
cilium_operator_ipam_available_ips

# 노드별 사용 중인 IP 수
cilium_operator_ipam_used_ips

# 가용 IP 비율
cilium_operator_ipam_available_ips /
(cilium_operator_ipam_available_ips + cilium_operator_ipam_used_ips) * 100
```

```json
{
  "title": "IPAM Available IPs Trend",
  "type": "timeseries",
  "targets": [
    {
      "expr": "cilium_operator_ipam_available_ips",
      "legendFormat": "Available - {{node}}"
    },
    {
      "expr": "cilium_operator_ipam_used_ips",
      "legendFormat": "Used - {{node}}"
    }
  ],
  "fieldConfig": {
    "defaults": { "unit": "short" }
  }
}
```

#### GrafanaDashboard CRD를 활용한 GitOps 관리

Grafana Operator(그라파나 오퍼레이터)를 사용하면 대시보드를 Kubernetes CRD로 선언적 관리가 가능함.

```bash
# Grafana Operator 설치
helm repo add grafana-operator https://grafana.github.io/grafana-operator
helm install grafana-operator grafana-operator/grafana-operator \
  --namespace monitoring
```

```yaml
# cilium-dashboards.yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: cilium-agent-dashboard
  namespace: monitoring
  labels:
    app: grafana
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  grafanaCom:
    id: 16611
    revision: 1
  datasources:
    - inputName: "DS_PROMETHEUS"
      datasourceName: "Prometheus"
---
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: cilium-operator-dashboard
  namespace: monitoring
  labels:
    app: grafana
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  grafanaCom:
    id: 16612
    revision: 1
  datasources:
    - inputName: "DS_PROMETHEUS"
      datasourceName: "Prometheus"
---
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: hubble-overview-dashboard
  namespace: monitoring
  labels:
    app: grafana
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  grafanaCom:
    id: 16613
    revision: 1
  datasources:
    - inputName: "DS_PROMETHEUS"
      datasourceName: "Prometheus"
---
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: hubble-dns-dashboard
  namespace: monitoring
  labels:
    app: grafana
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  grafanaCom:
    id: 16614
    revision: 1
  datasources:
    - inputName: "DS_PROMETHEUS"
      datasourceName: "Prometheus"
---
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: hubble-http-dashboard
  namespace: monitoring
  labels:
    app: grafana
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  grafanaCom:
    id: 16615
    revision: 1
  datasources:
    - inputName: "DS_PROMETHEUS"
      datasourceName: "Prometheus"
```

```bash
kubectl apply -f cilium-dashboards.yaml
```

커스텀 대시보드도 JSON을 직접 임베딩하여 CRD로 관리 가능함.

```yaml
# custom-cilium-dashboard.yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: cilium-custom-overview
  namespace: monitoring
spec:
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  json: |
    {
      "title": "Cilium Custom Overview",
      "tags": ["cilium", "custom"],
      "panels": [
        {
          "title": "Policy Drop Rate",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
          "targets": [{
            "expr": "sum(rate(cilium_drop_count_total{reason=\"Policy denied\"}[5m])) by (direction)",
            "legendFormat": "{{direction}}"
          }]
        },
        {
          "title": "Endpoint States",
          "type": "piechart",
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
          "targets": [{
            "expr": "sum(cilium_endpoint_state) by (endpoint_state)",
            "legendFormat": "{{endpoint_state}}"
          }]
        }
      ]
    }
```

---

### 2.3 보안/성능 Best Practice

#### 대시보드 폴더 구조

대시보드를 체계적으로 관리하기 위한 권장 폴더 구조:

| 폴더명 | 포함 대시보드 | 대상 사용자 |
|--------|-------------|------------|
| `Cilium` | Agent, Operator 대시보드 | 플랫폼 엔지니어 |
| `Hubble` | Overview, DNS, HTTP 대시보드 | 네트워크/SRE 엔지니어 |
| `Alerts` | 알림 전용 대시보드 | 온콜(On-Call) 담당자 |
| `Custom` | 팀별 커스텀 대시보드 | 개발팀 |

```bash
# Grafana API로 폴더 생성
curl -s -X POST "${GRAFANA_URL}/api/folders" \
  -u "${GRAFANA_AUTH}" \
  -H "Content-Type: application/json" \
  -d '{"title": "Cilium"}'

curl -s -X POST "${GRAFANA_URL}/api/folders" \
  -u "${GRAFANA_AUTH}" \
  -H "Content-Type: application/json" \
  -d '{"title": "Hubble"}'

curl -s -X POST "${GRAFANA_URL}/api/folders" \
  -u "${GRAFANA_AUTH}" \
  -H "Content-Type: application/json" \
  -d '{"title": "Alerts"}'
```

#### 리프레시 인터벌(Refresh Interval) 권장값

| 환경 | 권장 리프레시 인터벌 | 근거 |
|------|---------------------|------|
| 개발/테스트 | 30s | 빠른 피드백 필요 |
| 스테이징 | 1m | 적절한 균형 |
| 프로덕션 (일반) | 1m ~ 5m | Prometheus 부하 절감 |
| 프로덕션 (인시던트 대응) | 10s ~ 15s | 실시간 모니터링 필요 시 수동 변경 |

> 기본 리프레시 인터벌을 너무 짧게 설정하면 Prometheus 쿼리 부하가 증가하여 전체 모니터링 시스템 성능에 영향을 줄 수 있음. 일반적으로 1분 이상을 권장함.

#### 변수 템플릿(Variable Templates) 설정

네임스페이스(Namespace)와 파드(Pod) 기준 필터링을 위한 대시보드 변수 설정:

```json
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(cilium_endpoint_state, namespace)",
        "refresh": 2,
        "sort": 1,
        "includeAll": true,
        "multi": true
      },
      {
        "name": "pod",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(cilium_endpoint_state{namespace=~\"$namespace\"}, pod)",
        "refresh": 2,
        "sort": 1,
        "includeAll": true,
        "multi": true
      },
      {
        "name": "node",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(cilium_endpoint_state, instance)",
        "refresh": 2,
        "sort": 1,
        "includeAll": true,
        "multi": true
      }
    ]
  }
}
```

패널에서 변수를 사용하는 PromQL 예시:

```promql
# 선택한 네임스페이스의 드롭 비율
sum(rate(cilium_drop_count_total{namespace=~"$namespace"}[5m])) by (reason)

# 선택한 노드의 엔드포인트 상태
sum(cilium_endpoint_state{instance=~"$node"}) by (endpoint_state)
```

#### Grafana 알림 규칙(Alert Rules) 설정

```yaml
# grafana-alerts.yaml (Grafana Provisioning 방식)
apiVersion: 1
groups:
  - orgId: 1
    name: Cilium Alerts
    folder: Alerts
    interval: 1m
    rules:
      - uid: cilium-policy-drop-high
        title: "High Network Policy Drop Rate"
        condition: C
        data:
          - refId: A
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: <PROMETHEUS_DATASOURCE_UID>
            model:
              expr: "sum(rate(cilium_drop_count_total{reason=\"Policy denied\"}[5m]))"
              intervalMs: 1000
              maxDataPoints: 43200
          - refId: C
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: "__expr__"
            model:
              type: threshold
              conditions:
                - evaluator:
                    type: gt
                    params: [50]
        for: 5m
        annotations:
          summary: "네트워크 정책 드롭 비율이 높음 (> 50/s)"
          description: "최근 5분간 Policy denied 드롭이 초당 50건을 초과함."

      - uid: cilium-endpoint-unhealthy
        title: "Unhealthy Endpoints Detected"
        condition: C
        data:
          - refId: A
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: <PROMETHEUS_DATASOURCE_UID>
            model:
              expr: "sum(cilium_endpoint_state{endpoint_state!=\"ready\"}) / sum(cilium_endpoint_state) * 100"
          - refId: C
            datasourceUid: "__expr__"
            model:
              type: threshold
              conditions:
                - evaluator:
                    type: gt
                    params: [10]
        for: 5m
        annotations:
          summary: "비정상 엔드포인트 비율 10% 초과"

      - uid: cilium-bpf-map-pressure
        title: "BPF Map Pressure High"
        condition: C
        data:
          - refId: A
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: <PROMETHEUS_DATASOURCE_UID>
            model:
              expr: "max(cilium_bpf_map_pressure{}) by (map_name)"
          - refId: C
            datasourceUid: "__expr__"
            model:
              type: threshold
              conditions:
                - evaluator:
                    type: gt
                    params: [0.9]
        for: 10m
        annotations:
          summary: "BPF 맵 사용률 90% 초과"
          description: "맵 이름: {{ $labels.map_name }}"

      - uid: cilium-ipam-exhaustion
        title: "IPAM IP Exhaustion Warning"
        condition: C
        data:
          - refId: A
            relativeTimeRange:
              from: 300
              to: 0
            datasourceUid: <PROMETHEUS_DATASOURCE_UID>
            model:
              expr: "cilium_operator_ipam_available_ips < 5"
          - refId: C
            datasourceUid: "__expr__"
            model:
              type: threshold
              conditions:
                - evaluator:
                    type: gt
                    params: [0]
        for: 5m
        annotations:
          summary: "가용 IP가 5개 미만으로 감소"
```

---

## 3. 트러블슈팅

### 3.1 주요 이슈

#### 이슈 1: 대시보드에 "No data" 표시

**증상**: 대시보드 임포트 후 모든 패널에 "No data" 메시지가 표시됨.

**원인**: Prometheus 데이터 소스가 올바르게 설정되지 않았거나, 데이터 소스 이름이 대시보드에서 기대하는 것과 다름.

**해결 방법**:

```bash
# 1. Prometheus 데이터 소스 연결 확인
kubectl port-forward svc/grafana 3000:80 -n monitoring
# Grafana UI → Configuration → Data Sources → Prometheus → "Save & Test" 클릭

# 2. Prometheus가 Cilium 메트릭을 수집하고 있는지 직접 확인
kubectl port-forward svc/prometheus-server 9090:80 -n monitoring
# http://localhost:9090/targets 에서 cilium-agent 타겟 상태 확인

# 3. Cilium Agent에서 메트릭이 노출되는지 확인
kubectl exec -n kube-system <CILIUM_POD_NAME> -- curl -s http://localhost:9962/metrics | head -20

# 4. 대시보드 임포트 시 데이터 소스 매핑 재확인
# Grafana UI → Dashboard → Import → 데이터 소스 드롭다운에서 올바른 Prometheus 선택
```

#### 이슈 2: Hubble 패널이 비어 있음

**증상**: Cilium Agent 대시보드는 정상이지만, Hubble 관련 대시보드(DNS, HTTP, Overview)의 패널이 비어 있음.

**원인**: Hubble 메트릭이 활성화되지 않았거나, 필요한 메트릭 종류가 설정에 포함되지 않음.

**해결 방법**:

```bash
# 1. Hubble 활성화 상태 확인
cilium hubble port-forward &
cilium hubble status

# 2. Hubble 메트릭 설정 확인
kubectl get configmap cilium-config -n kube-system -o yaml | grep -A 5 "hubble"

# 3. Hubble 메트릭 엔드포인트 직접 확인
kubectl exec -n kube-system <CILIUM_POD_NAME> -- curl -s http://localhost:9965/metrics | head -30

# 4. Hubble 메트릭이 비어 있다면 Cilium Helm 값 업데이트
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set hubble.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"

# 5. Cilium Agent 재시작 (메트릭 설정 변경 적용)
kubectl rollout restart daemonset/cilium -n kube-system
```

#### 이슈 3: 대시보드 로드 시간이 매우 느림

**증상**: 대시보드 열기에 10초 이상 소요되며, 패널이 순차적으로 느리게 로드됨.

**원인**: 패널 수가 과도하거나, 리프레시 인터벌이 너무 짧거나, 쿼리의 시간 범위가 너무 넓음.

**해결 방법**:

```bash
# 1. Prometheus 쿼리 성능 확인
# Grafana UI → Panel → Inspect → Query → 각 쿼리 실행 시간 확인

# 2. 대시보드 설정 최적화
# - 기본 시간 범위를 "Last 1h"로 제한 (너무 넓은 범위 방지)
# - 리프레시 인터벌을 1m 이상으로 설정
# - 사용하지 않는 패널은 접어두거나 삭제

# 3. Prometheus Recording Rule 추가로 자주 사용하는 쿼리 사전 계산
```

```yaml
# prometheus-recording-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cilium-recording-rules
  namespace: monitoring
spec:
  groups:
    - name: cilium.rules
      interval: 30s
      rules:
        - record: cilium:drop_rate:5m
          expr: sum(rate(cilium_drop_count_total[5m])) by (reason, direction)

        - record: cilium:endpoint_state:total
          expr: sum(cilium_endpoint_state) by (endpoint_state)

        - record: hubble:http_error_rate:5m
          expr: |
            sum(rate(hubble_http_requests_total{status=~"4..|5.."}[5m]))
            / sum(rate(hubble_http_requests_total[5m])) * 100

        - record: cilium:ipam_available_ratio
          expr: |
            cilium_operator_ipam_available_ips
            / (cilium_operator_ipam_available_ips + cilium_operator_ipam_used_ips) * 100
```

Recording Rule 적용 후 대시보드 패널의 PromQL을 사전 계산된 메트릭으로 교체:

```promql
# 변경 전
sum(rate(cilium_drop_count_total[5m])) by (reason, direction)

# 변경 후 (Recording Rule 사용)
cilium:drop_rate:5m
```

#### 이슈 4: 대시보드 임포트 시 "Datasource not found" 오류

**증상**: Grafana.com에서 대시보드를 임포트할 때 데이터 소스를 찾을 수 없다는 오류 발생.

**원인**: 대시보드가 특정 이름의 데이터 소스를 요구하지만 실제 설정된 이름이 다름.

**해결 방법**:

```bash
# 1. 현재 설정된 데이터 소스 목록 확인
curl -s "${GRAFANA_URL}/api/datasources" -u "${GRAFANA_AUTH}" | jq '.[].name'

# 2. 임포트 시 inputs 매핑을 명시적으로 지정
curl -s -X POST "${GRAFANA_URL}/api/dashboards/import" \
  -u "${GRAFANA_AUTH}" \
  -H "Content-Type: application/json" \
  -d "{
    \"dashboard\": $(curl -s https://grafana.com/api/dashboards/16611/revisions/latest/download),
    \"overwrite\": true,
    \"inputs\": [{
      \"name\": \"DS_PROMETHEUS\",
      \"type\": \"datasource\",
      \"pluginId\": \"prometheus\",
      \"value\": \"<ACTUAL_DATASOURCE_NAME>\"
    }]
  }"
```

---

### 3.2 자주 발생하는 문제 (Q&A)

**Q1: Cilium 대시보드에서 일부 메트릭만 표시되고 나머지는 "No data"인 이유는?**

A: Cilium 버전에 따라 지원하는 메트릭이 다를 수 있음. `cilium version` 명령으로 Agent 버전을 확인하고, 해당 버전에 맞는 대시보드 리비전(Revision)을 사용해야 함. 또한 일부 메트릭은 특정 기능이 활성화되어야만 노출됨(예: Hubble L7 메트릭은 L7 프록시가 활성화되어야 함).

```bash
# Cilium 버전 확인
cilium version

# 사용 가능한 메트릭 목록 확인
kubectl exec -n kube-system <CILIUM_POD_NAME> -- curl -s http://localhost:9962/metrics | grep "^cilium_" | cut -d'{' -f1 | sort -u
```

**Q2: Grafana에서 Prometheus 데이터 소스 테스트 시 "Bad Gateway" 오류가 발생하는 이유는?**

A: Grafana 파드에서 Prometheus 서비스로의 네트워크 통신이 차단되었을 가능성이 있음. Cilium 네트워크 정책이 monitoring 네임스페이스 간 통신을 허용하는지 확인 필요.

```bash
# Grafana 파드에서 Prometheus 엔드포인트 접근 테스트
kubectl exec -n monitoring <GRAFANA_POD_NAME> -- \
  wget -qO- http://prometheus-server.monitoring.svc.cluster.local/api/v1/status/config

# 필요 시 네트워크 정책 추가
```

```yaml
# allow-grafana-to-prometheus.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-grafana-to-prometheus
  namespace: monitoring
spec:
  endpointSelector:
    matchLabels:
      app.kubernetes.io/name: grafana
  egress:
    - toEndpoints:
        - matchLabels:
            app.kubernetes.io/name: prometheus
      toPorts:
        - ports:
            - port: "9090"
              protocol: TCP
```

**Q3: 대시보드를 팀원과 공유하려면 어떻게 해야 하는가?**

A: 세 가지 방법이 있음:

1. **Grafana UI**: Dashboard → Share → 링크 또는 스냅샷(Snapshot) 공유
2. **JSON Export/Import**: Dashboard → Settings → JSON Model → Export → 팀원이 Import
3. **GitOps (권장)**: GrafanaDashboard CRD로 Git 리포지터리에서 관리하여 모든 팀원이 동일한 대시보드를 자동으로 프로비저닝

**Q4: EKS 환경에서 ENI 모드 사용 시 IPAM 관련 대시보드 패널에 데이터가 없는 이유는?**

A: Cilium Operator의 Prometheus 메트릭 노출이 활성화되어 있는지 확인 필요.

```bash
# Cilium Operator 메트릭 활성화 확인
helm get values cilium -n kube-system | grep -A 3 "operator"

# Operator 메트릭이 비활성화되어 있다면 활성화
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set operator.prometheus.enabled=true
```

---

## 4. 모니터링 및 확인

### Grafana 헬스 체크(Health Check)

```bash
# Grafana 파드 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# Grafana API 헬스 체크
kubectl port-forward svc/grafana 3000:80 -n monitoring &
curl -s http://localhost:3000/api/health | jq .
# 기대 출력: {"commit":"...","database":"ok","version":"..."}

# Grafana 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana --tail=50
```

### Prometheus 타겟(Target) 검증

```bash
# Prometheus 포트포워드
kubectl port-forward svc/prometheus-server 9090:80 -n monitoring &

# Prometheus 타겟 상태 API 확인
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.labels.job | startswith("cilium")) | {job: .labels.job, health: .health, lastError: .lastError}'
```

기대 출력 예시:

```json
{
  "job": "cilium-agent",
  "health": "up",
  "lastError": ""
}
{
  "job": "cilium-operator",
  "health": "up",
  "lastError": ""
}
{
  "job": "hubble",
  "health": "up",
  "lastError": ""
}
```

타겟이 `down` 상태인 경우:

```bash
# Cilium Agent 메트릭 포트 직접 확인
kubectl exec -n kube-system <CILIUM_POD_NAME> -- curl -s http://localhost:9962/metrics | wc -l

# Prometheus ServiceMonitor 또는 PodMonitor 확인 (kube-prometheus-stack 사용 시)
kubectl get servicemonitor -n monitoring
kubectl get podmonitor -n monitoring
```

### 대시보드 프로비저닝(Provisioning) 상태 확인

```bash
# Grafana Provisioning 설정 확인
kubectl exec -n monitoring <GRAFANA_POD_NAME> -- ls /etc/grafana/provisioning/dashboards/

# 프로비저닝된 대시보드 목록 API 확인
curl -s "${GRAFANA_URL}/api/search?type=dash-db" -u "${GRAFANA_AUTH}" | \
  jq '.[] | {title: .title, uid: .uid, folderTitle: .folderTitle}'

# GrafanaDashboard CRD 상태 확인 (Grafana Operator 사용 시)
kubectl get grafanadashboards -n monitoring
kubectl describe grafanadashboard cilium-agent-dashboard -n monitoring
```

### 전체 모니터링 스택 정상 동작 통합 점검 스크립트

```bash
#!/bin/bash
# check-monitoring-stack.sh

echo "=== 1. Grafana 상태 확인 ==="
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -o wide

echo ""
echo "=== 2. Prometheus 상태 확인 ==="
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus -o wide

echo ""
echo "=== 3. Cilium 메트릭 포트 확인 ==="
CILIUM_POD=$(kubectl get pods -n kube-system -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}')
echo "Cilium Agent 메트릭 (9962):"
kubectl exec -n kube-system "${CILIUM_POD}" -- curl -s -o /dev/null -w "%{http_code}" http://localhost:9962/metrics
echo ""
echo "Hubble 메트릭 (9965):"
kubectl exec -n kube-system "${CILIUM_POD}" -- curl -s -o /dev/null -w "%{http_code}" http://localhost:9965/metrics
echo ""

echo ""
echo "=== 4. Cilium Operator 메트릭 포트 확인 ==="
OPERATOR_POD=$(kubectl get pods -n kube-system -l name=cilium-operator -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n kube-system "${OPERATOR_POD}" -- curl -s -o /dev/null -w "%{http_code}" http://localhost:9963/metrics
echo ""

echo ""
echo "=== 5. Prometheus 타겟 상태 요약 ==="
kubectl port-forward svc/prometheus-server 9090:80 -n monitoring &
PF_PID=$!
sleep 2
curl -s http://localhost:9090/api/v1/targets | jq '[.data.activeTargets[] | select(.labels.job | startswith("cilium") or startswith("hubble"))] | group_by(.health) | map({health: .[0].health, count: length})'
kill $PF_PID 2>/dev/null
```

---

## 5. TIP

### 공식 Cilium Grafana 대시보드 링크

- **Cilium Agent**: https://grafana.com/grafana/dashboards/16611
- **Cilium Operator**: https://grafana.com/grafana/dashboards/16612
- **Hubble Overview**: https://grafana.com/grafana/dashboards/16613
- **Hubble DNS**: https://grafana.com/grafana/dashboards/16614
- **Hubble HTTP**: https://grafana.com/grafana/dashboards/16615

### 프로덕션 환경 권장 대시보드 세트

| 우선순위 | 대시보드 | 용도 | 대상 |
|---------|---------|------|------|
| 필수 | Cilium Agent (16611) | Agent 핵심 상태 모니터링 | 플랫폼팀 |
| 필수 | Cilium Operator (16612) | IPAM 및 Operator 상태 (특히 ENI 모드) | 플랫폼팀 |
| 필수 | Hubble Overview (16613) | 네트워크 플로우 전체 현황 | SRE/네트워크팀 |
| 권장 | Hubble DNS (16614) | DNS 문제 진단 | SRE/개발팀 |
| 권장 | Hubble HTTP (16615) | L7 HTTP 트래픽 분석 | 개발팀/SRE |
| 권장 | Custom: IPAM Trend | EKS ENI 모드 IP 가용성 추이 | 플랫폼팀 |
| 선택 | Custom: Policy Drop | 네트워크 정책 위반 실시간 감지 | 보안팀 |

### 추가 팁

- **대시보드 버전 관리**: Grafana 자체 버전 히스토리 기능(Dashboard → Settings → Versions)을 활용하여 변경 이력 추적이 가능함. 단, 프로덕션에서는 GitOps 방식 병행을 권장함.
- **익스플로러(Explorer) 활용**: 새로운 PromQL 쿼리를 작성할 때는 대시보드 패널이 아닌 Grafana Explorer에서 먼저 테스트 후 패널에 추가하는 것이 효율적임.
- **어노테이션(Annotation) 활용**: Helm 업그레이드, 네트워크 정책 변경 등의 이벤트를 Grafana 어노테이션으로 기록하면, 메트릭 변화와 변경 사항의 상관관계를 파악하기 쉬움.

```bash
# Grafana 어노테이션 추가 예시 (Cilium 업그레이드 시)
curl -s -X POST "${GRAFANA_URL}/api/annotations" \
  -u "${GRAFANA_AUTH}" \
  -H "Content-Type: application/json" \
  -d "{
    \"text\": \"Cilium upgraded to v1.16.x\",
    \"tags\": [\"cilium\", \"upgrade\"]
  }"
```

- **Grafana Loki 연동**: Hubble 플로우 로그를 Loki(로키)로 수집하면, Grafana에서 메트릭과 로그를 동시에 조회 가능함. 특정 시점의 드롭 급증 원인을 로그로 추적할 때 유용함.
