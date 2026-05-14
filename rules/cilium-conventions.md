# Cilium 코드 작성 규칙 (Cilium Code Conventions)

이 저장소에서 Cilium YAML 및 kubectl/cilium CLI 예시 코드 작성 시 따라야 할 규칙입니다.

---

## 1. YAML 공통 규칙

### 기본 형식
```yaml
apiVersion: cilium.io/v2                 # Cilium CRD API 버전
kind: CiliumNetworkPolicy
metadata:
  name: <RESOURCE_NAME>
  namespace: <NAMESPACE>               # namespace 항상 명시
spec:
  ...
```

- `apiVersion`은 `cilium.io/v2` 사용
- `namespace` 항상 명시 (default 의존 금지)
- 플레이스홀더: `<NAMESPACE>`, `<SERVICE_NAME>`, `<POD_SELECTOR>` 형식

### 레이블 필수 항목
```yaml
metadata:
  labels:
    app: <APP_NAME>
spec:
  endpointSelector:
    matchLabels:
      app: <APP_NAME>        # 정책 대상 명확히 지정
```

## 2. CiliumNetworkPolicy 규칙

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: <POLICY_NAME>
  namespace: <NAMESPACE>
spec:
  endpointSelector:
    matchLabels:
      app: <APP_NAME>
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: <SOURCE_APP>
      toPorts:
        - ports:
            - port: "<PORT>"
              protocol: TCP
          rules:
            http:                        # L7 정책은 필요 시에만
              - method: "GET"
                path: "/api/.*"
```

## 3. CiliumClusterwideNetworkPolicy 규칙

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: <POLICY_NAME>                   # namespace 없음 (클러스터 전역)
spec:
  endpointSelector: {}                  # 전체 엔드포인트 대상
  ingress:
    - fromEntities:
        - health                        # Health check 허용
```

## 4. CiliumEgressGatewayPolicy 규칙

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumEgressGatewayPolicy
metadata:
  name: <POLICY_NAME>
spec:
  selectors:
    - podSelector:
        matchLabels:
          app: <APP_NAME>
  destinationCIDRs:
    - "0.0.0.0/0"                       # 외부 전체 또는 특정 CIDR
  egressGateway:
    nodeSelector:
      matchLabels:
        cilium.io/egress-gateway: "true"
    egressIP: <EIP>
```

## 5. kubectl / cilium CLI 명령어 규칙

### 기본 형식
```bash
kubectl apply -f <YAML_FILE> -n <NAMESPACE>
kubectl get cnp -n <NAMESPACE>
cilium status --wait
cilium connectivity test
hubble observe --namespace <NAMESPACE>
```

- `-n <NAMESPACE>` 항상 명시 (컨텍스트 의존 금지)
- 진단 시 `cilium status` 먼저 실행
- 트래픽 확인은 `hubble observe` 사용

## 6. 환경 설정

- 예시의 기본 네임스페이스: `default` (앱), `kube-system` (Cilium)
- 앱 이름 컨벤션: `my-app`
- Cilium 버전: `1.16.x` (EKS ENI 모드 기준)
