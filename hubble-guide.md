# Hubble — 네트워크 가시성

Hubble은 Cilium에 내장된 **eBPF 기반 네트워크 관찰 플랫폼**입니다.
사이드카 없이 L3/L4/L7 수준의 모든 네트워크 흐름을 실시간으로 캡처합니다.

---

## 아키텍처

```
┌──────────────────────────────────────────────────────┐
│  Node A                                              │
│  Pod → eBPF hook → Cilium Agent                     │
│                        │                            │
│                  Hubble Server (gRPC :4244)          │
└──────────────────────────────────┬───────────────────┘
                                   │ gRPC
                           Hubble Relay
                           (클러스터 전체 흐름 집계)
                           ┌───────┴──────────┐
                      Hubble CLI          Hubble UI
                      (hubble observe)   (Web 시각화)
                                   │
                            Prometheus
                            (/metrics)
                                   │
                            Grafana Dashboard
```

---

## Hubble 활성화

```yaml
# helm/values.yaml
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
  metrics:
    enabled:
      - dns:query;ignoreAAAA
      - drop
      - tcp
      - flow
      - icmp
      - http
```

적용:

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --values my-values.yaml \
  --reuse-values

# Hubble 파드 확인
kubectl get pods -n kube-system -l k8s-app=hubble-relay
kubectl get pods -n kube-system -l k8s-app=hubble-ui
```

---

## Hubble CLI 사용법

### 설치

```bash
# Hubble CLI 설치 (macOS)
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all \
  https://github.com/cilium/hubble/releases/download/${HUBBLE_VERSION}/hubble-darwin-amd64.tar.gz
tar xzvf hubble-darwin-amd64.tar.gz
sudo mv hubble /usr/local/bin
```

### 포트 포워드

```bash
# Hubble Relay에 접속
kubectl port-forward svc/hubble-relay -n kube-system 4245:80 &

# 상태 확인
hubble status
```

### 기본 명령어

```bash
# 모든 흐름 실시간 관찰
hubble observe --follow

# 특정 네임스페이스의 흐름
hubble observe --namespace default --follow

# 특정 파드의 송수신 흐름
hubble observe --pod default/frontend --follow

# 드롭된 패킷만 확인
hubble observe --verdict DROPPED --follow

# 특정 두 파드 간 트래픽
hubble observe \
  --from-pod default/frontend \
  --to-pod default/backend \
  --follow

# L7 HTTP 흐름 확인
hubble observe --protocol http --follow

# DNS 쿼리 확인
hubble observe --protocol dns --follow

# 최근 1000개 흐름 조회 (follow 없이)
hubble observe --last 1000
```

---

## 흐름 필터 옵션

| 옵션 | 설명 | 예시 |
|---|---|---|
| `--namespace` | 네임스페이스 필터 | `--namespace production` |
| `--pod` | 특정 파드 | `--pod default/nginx` |
| `--from-pod` | 출발 파드 | `--from-pod default/frontend` |
| `--to-pod` | 도착 파드 | `--to-pod default/backend` |
| `--verdict` | 처리 결과 | `--verdict DROPPED` |
| `--protocol` | 프로토콜 | `--protocol http` |
| `--port` | 포트 | `--port 443` |
| `--type` | 이벤트 타입 | `--type drop` |
| `--label` | 레이블 필터 | `--label app=frontend` |

---

## Hubble UI

```bash
# 포트 포워드
kubectl port-forward svc/hubble-ui -n kube-system 8081:80 &

# 브라우저에서 열기
open http://localhost:8081
```

UI에서 확인 가능한 항목:
- **서비스 맵**: 서비스 간 통신 흐름 시각화
- **네임스페이스 필터**: 네임스페이스별 흐름 분리
- **흐름 상세**: 각 연결의 L3/L4/L7 정보
- **드롭 강조 표시**: 차단된 패킷 실시간 표시

---

## Prometheus 메트릭

Hubble은 Prometheus 메트릭을 기본 포트 `9965`에 노출합니다.

```bash
# 메트릭 확인 (Agent에서)
kubectl -n kube-system exec ds/cilium -- \
  curl -s http://localhost:9965/metrics | head -50
```

### 주요 메트릭

| 메트릭 | 설명 |
|---|---|
| `hubble_flows_processed_total` | 처리된 총 흐름 수 |
| `hubble_drop_total` | 드롭된 패킷 수 (reason 레이블 포함) |
| `hubble_http_requests_total` | HTTP 요청 수 (method, status 레이블) |
| `hubble_dns_queries_total` | DNS 쿼리 수 |
| `hubble_dns_responses_total` | DNS 응답 수 |
| `hubble_tcp_flags_total` | TCP 플래그 통계 |

### Prometheus ServiceMonitor 예시

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: hubble
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - kube-system
  selector:
    matchLabels:
      k8s-app: cilium
  endpoints:
    - port: hubble-metrics
      interval: 30s
      path: /metrics
```

---

## Grafana 대시보드

Cilium 공식 Grafana 대시보드 ID:

| 대시보드 | ID |
|---|---|
| Cilium Agent Metrics | 16611 |
| Cilium Operator Metrics | 16612 |
| Hubble L7 HTTP Metrics | 16613 |
| Hubble DNS Metrics | 16614 |

```bash
# Grafana에서 Import 방법
# Grafana → Dashboards → Import → ID 입력
```

---

## 흐름 분석 실전 예시

### 차단된 트래픽 원인 파악

```bash
# 드롭된 패킷과 원인 확인
hubble observe --verdict DROPPED --follow \
  --output json | jq '{
    from: .flow.source.pod_name,
    to: .flow.destination.pod_name,
    reason: .flow.drop_reason_desc,
    port: .flow.l4.TCP.destination_port
  }'
```

### 특정 서비스의 오류율 확인

```bash
# HTTP 5xx 오류 흐름
hubble observe --protocol http --follow \
  --output json | jq 'select(.flow.l7.http.code >= 500)'
```

### DNS 실패 추적

```bash
# DNS NXDOMAIN 응답 확인
hubble observe --protocol dns --follow \
  --output json | jq 'select(.flow.l7.dns.rcode == "NXDomain")'
```

---

## 참고 링크

- [Hubble 공식 문서](https://docs.cilium.io/en/stable/observability/hubble/)
- [Hubble CLI GitHub](https://github.com/cilium/hubble)
- [Hubble 메트릭 레퍼런스](https://docs.cilium.io/en/stable/observability/metrics/)
- [Grafana 대시보드 목록](https://grafana.com/orgs/cilium/dashboards)
