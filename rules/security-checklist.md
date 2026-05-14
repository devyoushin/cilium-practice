# 보안 체크리스트 (Security Checklist)

Cilium 관련 문서 작성 및 YAML 작성 시 보안 검토 기준입니다.

---

## 1. 네트워크 정책 체크리스트

- [ ] 네임스페이스에 Default-deny 정책이 존재하는지 확인
- [ ] L7 정책 사용 시 필요한 HTTP 메서드/경로만 허용
- [ ] DNS 기반 Egress 정책에서 FQDN을 명시적으로 지정

```yaml
# Default-deny 예시 (반드시 네임스페이스에 적용)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny
  namespace: <NAMESPACE>
spec:
  endpointSelector: {}
  ingress:
    - {}
  egress:
    - {}
```

## 2. 암호화 체크리스트

- [ ] WireGuard 투명 암호화 활성화 여부 확인 (운영 환경 권장)
- [ ] 암호화 미적용 시 반드시 이유 주석 추가 및 임시 적용 명시
- [ ] 노드 간 통신 암호화 상태 확인

```bash
# WireGuard 상태 확인
cilium encrypt status
# 노드별 암호화 인터페이스 확인
kubectl -n kube-system exec ds/cilium -- cilium encrypt status
```

## 3. YAML 보안 체크리스트

- [ ] 하드코딩된 비밀번호, 토큰, 인증서 내용 없는지 확인
- [ ] Secret은 YAML에 직접 포함하지 않고 `secretRef` 사용
- [ ] `hostNetwork: true` / `privileged: true` Pod 사용 금지 (Cilium Agent 제외)
- [ ] `0.0.0.0/0` CIDR 범위 사용 시 주의 문구 추가

## 4. Egress 제어 체크리스트

- [ ] 외부 통신이 필요한 Pod에만 Egress 정책 허용
- [ ] DNS 기반 FQDN 정책으로 외부 접근 대상 명시적 정의
- [ ] Egress Gateway를 통해 외부 트래픽 제어 권장 (고정 IP 필요 시)

## 5. 문서 내 보안 표현 규칙

- 보안 취약 설정 예시 작성 시 반드시 **주의** 또는 **운영 환경 금지** 표시
- 예시:
  ```yaml
  # ⚠️ 주의: 아래 설정은 개발 환경 전용. 운영 환경에서 사용 금지
  encryption:
    enabled: false
  ```
