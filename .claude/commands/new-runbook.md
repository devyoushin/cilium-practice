---
description: 운영 Runbook을 생성합니다. 사용법: /new-runbook <카테고리> <작업명>
---

`$ARGUMENTS`를 파싱하여 운영 Runbook을 생성합니다.
- 첫 번째 인자: 대상 카테고리 (install, security, networking, observability, operations)
- 나머지 인자: 작업명

## 파일 생성 규칙

1. 파일명: `runbook-{작업명}.md`
2. 저장 위치: `docs/{해당 카테고리}/`
3. `templates/runbook.md` 템플릿 기반으로 작성

## 작성 시 필수 포함 사항

- **사전 체크리스트**: 최소 4개 항목
- **환경 변수 설정**: `export` 형식으로 명시
- **단계별 명령어**: 복붙 즉시 실행 가능
- **확인 명령어**: 각 스텝 완료 후 `kubectl` / `cilium` CLI 검증 방법
- **롤백 절차**: 되돌릴 수 있는 YAML 또는 명령어 포함
- **모니터링 포인트**: 작업 후 확인할 Hubble / Prometheus 지표

## 작업 특성별 추가 요소

| 작업 유형 | 추가 항목 |
|----------|---------|
| 네트워크 정책 변경 | CiliumNetworkPolicy 영향 범위, 정책 우선순위 확인 |
| Egress Gateway 설정 | EIP 매핑, 게이트웨이 노드 상태 확인 |
| 업그레이드 | Cilium 버전 호환성, eBPF 프로그램 재로드 계획 |
| 암호화 전환 | WireGuard/IPsec 영향 범위, 순단 여부 확인 |
