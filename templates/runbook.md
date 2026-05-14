# Runbook: {작업명}

> **분류**: {네트워크 정책 변경 | Egress Gateway 설정 | 암호화 전환 | 업그레이드}
> **대상 서비스**: {서비스명}
> **작성일**: {YYYY-MM-DD}
> **예상 소요 시간**: {N분}
> **영향 범위**: {무중단 | 순단 N초 | 서비스 중단}

---

## 사전 체크리스트

- [ ] 변경 대상 YAML 백업 완료
- [ ] 관련 팀 슬랙 채널 공지
- [ ] 롤백 YAML 준비 완료
- [ ] Hubble UI / Grafana 대시보드 준비

---

## 환경 변수 설정

```bash
export NAMESPACE=<NAMESPACE>
export APP_NAME=<APP_NAME>
export CILIUM_VERSION=1.16.x

# 작업 대상 (시작 전 반드시 확인)
export TARGET_POLICY=<POLICY_NAME>
```

---

## Step 1: 사전 상태 확인

```bash
# 현재 정책 상태 백업
kubectl get cnp ${TARGET_POLICY} -n ${NAMESPACE} -o yaml > backup-cnp-$(date +%Y%m%d%H%M%S).yaml

# Cilium 전체 상태 확인
cilium status --wait

# 엔드포인트 상태 확인
cilium endpoint list

# 연결성 테스트
cilium connectivity test
```

**예상 출력:**
```
# cilium status: OK
# endpoint list: ready 상태인지 확인
# connectivity test: All tests passed
```

---

## Step 2: {작업 내용}

```yaml
# 변경할 YAML
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
...
```

```bash
kubectl apply -f <CHANGE_YAML> -n ${NAMESPACE}
```

**확인 포인트:**
- 정책이 올바르게 적용됐는지 확인 (`kubectl get cnp -n ${NAMESPACE}`)

---

## Step 3: 완료 확인

```bash
# 변경 후 정책 상태 확인
kubectl get cnp -n ${NAMESPACE}
cilium policy get -n ${NAMESPACE}

# 트래픽 정상 흐름 확인 (Hubble)
hubble observe --namespace ${NAMESPACE} --verdict DROPPED
```

**성공 기준:**
- [ ] `cilium status` 에러 없음
- [ ] 드롭 패킷 없음 (5분간 모니터링)
- [ ] 대상 서비스 정상 응답

---

## 롤백 절차

> 아래 상황에서 롤백 수행: {드롭 패킷 지속 / 서비스 불통 / 엔드포인트 not-ready}

```bash
# 백업 YAML로 롤백
kubectl apply -f backup-cnp-<TIMESTAMP>.yaml -n ${NAMESPACE}

# 롤백 확인
cilium status --wait
hubble observe --namespace ${NAMESPACE} --verdict DROPPED
```

---

## 모니터링 포인트

작업 완료 후 **30분간** 아래 지표 모니터링:

| 지표 | 정상 범위 | 이상 기준 |
|------|----------|---------|
| 드롭 패킷 | 0 | 지속 발생 |
| 엔드포인트 상태 | 전체 ready | not-ready 존재 |
| 정책 verdict | 예상 허용/차단 비율 | 예상 외 차단 급증 |

---

## 완료 보고 템플릿

```
[작업 완료 보고]
- 작업명: {작업명}
- 수행 시간: {시작} ~ {종료}
- 영향: {실제 영향}
- 결과: 정상 완료 / 롤백 / 이슈 발생
- 특이사항: {없음 | 내용}
```
