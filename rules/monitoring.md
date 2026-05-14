# 모니터링 작성 기준 (Monitoring Guidelines)

Cilium 관련 문서에서 모니터링/확인 섹션 작성 시 따라야 할 기준입니다.

---

## 1. 모니터링 섹션 필수 포함 항목

모든 문서의 **4. 모니터링 및 확인** 섹션에는 아래 중 해당하는 항목을 반드시 포함:

| 항목 | 포함 조건 |
|------|----------|
| `cilium` CLI 진단 명령어 | 모든 문서 |
| Hubble 확인 방법 | 트래픽 관리 / 관측성 문서 |
| Prometheus 지표명 | 메트릭 관련 문서 |
| Grafana 대시보드 | 운영 관련 문서 |
| BPF 맵 확인 | 로드밸런싱 / 정책 문서 |

## 2. 핵심 cilium CLI 진단 명령어

```bash
# Cilium 전체 상태 확인
cilium status --wait

# 연결성 테스트 (Pod 간 통신 검증)
cilium connectivity test

# 엔드포인트 상태 확인
cilium endpoint list

# 정책 적용 상태 확인
cilium policy get -n <NAMESPACE>

# BPF 로드밸런서 상태
cilium bpf lb list

# 암호화 상태 확인
cilium encrypt status
```

## 3. 핵심 Hubble 명령어

```bash
# 네임스페이스 전체 트래픽 관측
hubble observe --namespace <NAMESPACE>

# 차단된 트래픽만 필터
hubble observe --verdict DROPPED

# 특정 Pod 간 통신 관측
hubble observe --from-pod <NAMESPACE>/<POD_NAME> --to-pod <NAMESPACE>/<POD_NAME>

# DNS 요청 관측
hubble observe --protocol DNS

# L7 HTTP 흐름 관측
hubble observe --protocol HTTP --namespace <NAMESPACE>
```

## 4. 핵심 Prometheus 지표

| 지표 | 설명 | 알람 기준 예시 |
|------|------|--------------|
| `cilium_endpoint_state` | 엔드포인트 상태별 수 | not-ready > 0 지속 |
| `cilium_policy_verdict` | 정책 판정 결과 (허용/차단) | dropped 급증 |
| `cilium_forward_count_total` | 전달된 패킷 수 | 급격한 변화 |
| `cilium_drop_count_total` | 드롭된 패킷 수 | 지속적 증가 |
| `hubble_flows_processed_total` | Hubble 처리 플로우 수 | 처리량 저하 |

## 5. Grafana 대시보드 확인 포인트

- **Cilium Agent**: 엔드포인트 수, 정책 적용률, BPF 맵 사용량
- **Hubble**: 트래픽 흐름, 프로토콜 분포, 차단 트래픽
- **네트워크 정책**: 정책별 허용/차단 비율, L7 요청 분포

## 6. BPF 맵 확인

```bash
# 로드밸런서 서비스 맵
cilium bpf lb list

# 연결 추적 테이블
cilium bpf ct list global

# 정책 맵 확인
cilium bpf policy get --all

# NAT 테이블
cilium bpf nat list
```
