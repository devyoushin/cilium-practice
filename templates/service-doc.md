# {서비스명} Cilium 구성 문서

> **네임스페이스**: {namespace}
> **작성일**: {YYYY-MM-DD}
> **Cilium 버전**: 1.16.x

---

## 1. 서비스 개요

| 항목 | 내용 |
|------|------|
| 서비스명 | {service-name} |
| 포트 | {PORT} |
| 프로토콜 | HTTP / gRPC / TCP |
| 네임스페이스 | {namespace} |
| 레이블 | `app: {app-name}` |

---

## 2. 네트워크 정책

### Ingress 정책

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: {service-name}-ingress
  namespace: {namespace}
spec:
  endpointSelector:
    matchLabels:
      app: {service-name}
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: {source-service}
      toPorts:
        - ports:
            - port: "{PORT}"
              protocol: TCP
```

### Egress 정책

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: {service-name}-egress
  namespace: {namespace}
spec:
  endpointSelector:
    matchLabels:
      app: {service-name}
  egress:
    - toEndpoints:
        - matchLabels:
            app: {dest-service}
      toPorts:
        - ports:
            - port: "{PORT}"
              protocol: TCP
    - toFQDNs:
        - matchName: "{external-domain}"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

---

## 3. 외부 연동

| 외부 서비스 | FQDN | 포트 | 프로토콜 | Egress Gateway |
|------------|------|------|---------|---------------|
| {서비스명} | {domain} | {port} | HTTPS | 사용 / 미사용 |

---

## 4. 암호화 설정

```
현재 모드: WireGuard / IPsec / 미설정
```

---

## 5. 모니터링

```bash
# 서비스 관련 엔드포인트 상태 확인
cilium endpoint list | grep {service-name}

# 서비스 트래픽 관측
hubble observe --namespace {namespace} --to-label app={service-name}

# 정책 적용 상태 확인
kubectl get cnp -n {namespace}
```

- Hubble UI: `{서비스명}` 트래픽 흐름에서 드롭 확인
- Prometheus: `cilium_policy_verdict{direction="ingress",match_l4_only="false"}`
