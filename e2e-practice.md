# End-to-End 실습

설치부터 네트워크 정책 적용, Hubble 관찰까지 전체 흐름을 실습합니다.

---

## 실습 목표

```
1. Cilium 설치 및 상태 확인
2. 테스트 애플리케이션 배포
3. 파드 간 통신 확인 (정책 없음)
4. NetworkPolicy 적용 및 차단 확인
5. CiliumNetworkPolicy (L7)로 HTTP 경로 제어
6. Hubble로 트래픽 흐름 관찰
7. Egress Gateway 실습 (선택)
```

---

## 사전 준비

```bash
# 환경 확인
kubectl get nodes
cilium status --wait

# 실습용 네임스페이스 생성
kubectl create namespace practice
```

---

## Step 1: 테스트 애플리케이션 배포

### frontend 파드 (클라이언트 역할)

```bash
kubectl apply -n practice -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: nicolaka/netshoot
          command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - port: 80
EOF
```

### backend 파드 (HTTP 서버 역할)

```bash
kubectl apply -n practice -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: hashicorp/http-echo
          args:
            - "-text=Hello from backend"
            - "-listen=:8080"
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
EOF
```

### 배포 확인

```bash
kubectl get pods -n practice
kubectl get svc -n practice
```

---

## Step 2: 정책 없이 통신 확인 (허용 상태)

```bash
FRONTEND_POD=$(kubectl get pod -n practice -l app=frontend -o jsonpath='{.items[0].metadata.name}')

# backend로 HTTP 요청
kubectl exec -n practice ${FRONTEND_POD} -- \
  curl -s http://backend:8080
# 예상 결과: "Hello from backend"

# 연결 성공 확인
echo "정책 없음 → 통신 허용 확인 완료"
```

---

## Step 3: K8s NetworkPolicy 적용

backend 파드에 Ingress 정책을 적용하여 frontend에서만 접근 허용합니다.

```bash
kubectl apply -n practice -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
EOF
```

### 정책 효과 확인

```bash
# frontend에서는 여전히 접근 가능
kubectl exec -n practice ${FRONTEND_POD} -- \
  curl -s http://backend:8080
# 예상 결과: "Hello from backend" (성공)

# 다른 파드에서는 차단
kubectl run -n practice temp --image=nicolaka/netshoot --rm -it -- \
  curl -s --max-time 3 http://backend.practice:8080
# 예상 결과: 연결 타임아웃 (차단)
```

---

## Step 4: Hubble로 트래픽 흐름 관찰

```bash
# 포트 포워드 (백그라운드)
kubectl port-forward svc/hubble-relay -n kube-system 4245:80 &

# 실시간 흐름 관찰
hubble observe --namespace practice --follow &
HUBBLE_PID=$!

# 요청 발생
kubectl exec -n practice ${FRONTEND_POD} -- \
  curl -s http://backend:8080

# 차단된 흐름 관찰
kubectl run -n practice temp2 --image=nicolaka/netshoot --rm -it -- \
  curl -s --max-time 3 http://backend.practice:8080 || true

# DROPPED 흐름만 필터링
hubble observe --namespace practice --verdict DROPPED
```

**예상 출력**:
```
TIMESTAMP   SOURCE              DESTINATION        TYPE     VERDICT  SUMMARY
...         practice/frontend   practice/backend   TCP      FORWARDED  ...
...         practice/temp2      practice/backend   Policy   DROPPED    Policy denied
```

---

## Step 5: CiliumNetworkPolicy (L7 HTTP)

HTTP 메서드와 경로를 기반으로 더 세밀한 제어를 적용합니다.

먼저 기존 K8s NetworkPolicy를 제거하고 CiliumNetworkPolicy로 교체합니다.

```bash
kubectl delete networkpolicy backend-ingress-policy -n practice
```

```bash
kubectl apply -n practice -f - <<EOF
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-l7-policy
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
EOF
```

### L7 정책 효과 확인

```bash
# GET 요청 → 허용
kubectl exec -n practice ${FRONTEND_POD} -- \
  curl -s -X GET http://backend:8080
# 예상 결과: "Hello from backend"

# POST 요청 → 차단 (L7 정책으로 GET만 허용)
kubectl exec -n practice ${FRONTEND_POD} -- \
  curl -s -X POST http://backend:8080 -d '{}'
# 예상 결과: 403 Access Denied

# Hubble에서 L7 차단 확인
hubble observe --namespace practice --verdict DROPPED --follow
```

---

## Step 6: DNS 기반 이그레스 정책

외부 API 접근을 도메인명으로 제어합니다.

```bash
kubectl apply -n practice -f - <<EOF
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: frontend-egress-policy
spec:
  endpointSelector:
    matchLabels:
      app: frontend
  egress:
    # DNS 허용 (필수)
    - toEndpoints:
        - matchLabels:
            k8s:k8s-app: kube-dns
            k8s:io.kubernetes.pod.namespace: kube-system
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    # 특정 외부 도메인만 허용
    - toFQDNs:
        - matchName: "example.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    # 클러스터 내부 backend 허용
    - toEndpoints:
        - matchLabels:
            app: backend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
EOF
```

### DNS 정책 확인

```bash
# example.com → 허용
kubectl exec -n practice ${FRONTEND_POD} -- \
  curl -s --max-time 5 https://example.com | head -5

# google.com → 차단
kubectl exec -n practice ${FRONTEND_POD} -- \
  curl -s --max-time 3 https://google.com || echo "차단됨"

# DNS 흐름 확인
hubble observe --namespace practice --protocol dns
```

---

## Step 7: 정리

```bash
# 실습 리소스 정리
kubectl delete namespace practice

# 포트 포워드 종료
kill ${HUBBLE_PID}
pkill -f "port-forward.*hubble-relay"
```

---

## 실습 체크리스트

- [ ] Cilium 설치 및 `cilium status` 정상 확인
- [ ] 정책 없는 상태에서 파드 간 통신 확인
- [ ] K8s NetworkPolicy 적용 후 차단 확인
- [ ] Hubble에서 FORWARDED / DROPPED 흐름 확인
- [ ] CiliumNetworkPolicy L7 (HTTP 메서드 제한) 적용 확인
- [ ] DNS 기반 이그레스 정책 적용 확인
- [ ] (선택) Hubble UI에서 서비스 맵 확인

---

## 심화 실습 (선택)

```bash
# 1. WireGuard 암호화 활성화 후 노드 간 트래픽 암호화 확인
# → service-mesh-guide.md 참고

# 2. Egress Gateway로 고정 IP 출구 구성
# → egress-gateway-guide.md 참고

# 3. cilium connectivity test 전체 실행
cilium connectivity test

# 4. Hubble Grafana 대시보드 설정
# → hubble-guide.md 참고
```
