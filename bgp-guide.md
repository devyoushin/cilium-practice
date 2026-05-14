# BGP Control Plane

Cilium BGP Control Plane은 Kubernetes Service의 `LoadBalancer` IP를
**BGP 프로토콜로 온프레미스 라우터에 광고**하는 기능입니다.
AWS에서는 Transit Gateway 또는 Direct Connect 환경에서 유용합니다.

---

## 사용 시나리오

```
온프레미스 / Direct Connect 환경:
  EKS 클러스터 → BGP → 온프레미스 라우터 → 온프레미스 서버
  (LoadBalancer IP를 별도 NLB 없이 BGP로 광고)

또는 자체 관리형 Kubernetes:
  클러스터 → BGP → ToR 스위치 → 데이터센터 네트워크
```

> **참고**: 일반 AWS EKS 환경(인터넷 NLB 사용)에서는 AWS Load Balancer Controller를
> 사용하는 것이 더 적합합니다. BGP는 주로 온프레미스/하이브리드 환경에서 활용됩니다.

---

## 활성화

```yaml
# helm/values.yaml
bgpControlPlane:
  enabled: true
```

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --values my-values.yaml \
  --reuse-values

# BGP 상태 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium bgp peers
```

---

## CiliumBGPPeeringPolicy 설정

### 1. BGP 피어링 정책 (노드와 라우터 간 BGP 세션 수립)

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: bgp-peering
spec:
  nodeSelector:
    matchLabels:
      bgp-speaker: "true"           # BGP를 실행할 노드 선택
  virtualRouters:
    - localASN: 65001               # 클러스터 AS 번호
      exportPodCIDR: false          # 파드 CIDR 광고 여부
      neighbors:
        - peerAddress: "192.168.1.1/32"   # 온프레미스 라우터 IP
          peerASN: 65000                  # 라우터 AS 번호
          gracefulRestart:
            enabled: true
            restartTimeSeconds: 120
          families:
            - afi: ipv4
              safi: unicast
      serviceSelector:
        matchExpressions:
          - key: io.kubernetes.service.namespace
            operator: NotIn
            values:
              - kube-system         # kube-system 서비스는 광고 제외
```

### 2. 노드에 레이블 추가

```bash
kubectl label node <NODE_NAME> bgp-speaker=true
```

### 3. LoadBalancer IP Pool 설정

BGP로 광고할 IP 대역을 지정합니다.

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: bgp-pool
spec:
  cidrs:
    - cidr: "192.168.100.0/24"     # 광고할 IP 대역
  serviceSelector:
    matchLabels:
      advertise-via-bgp: "true"    # 이 레이블을 가진 서비스에만 할당
```

---

## Service에 LoadBalancer IP 할당

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    advertise-via-bgp: "true"       # BGP 풀에서 IP 할당
spec:
  type: LoadBalancer
  loadBalancerClass: io.cilium/bgp  # Cilium BGP 로드밸런서 클래스
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f my-service.yaml

# IP 할당 확인
kubectl get svc my-service
# EXTERNAL-IP에 192.168.100.x가 할당되어야 함
```

---

## BGP 상태 확인

```bash
# BGP 피어 상태 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium bgp peers

# 광고 중인 경로 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium bgp routes advertised ipv4 unicast

# 수신된 경로 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium bgp routes available ipv4 unicast

# BFD(Bidirectional Forwarding Detection) 세션 상태
kubectl -n kube-system exec ds/cilium -- \
  cilium bgp peers | grep -i state
```

---

## BGP 피어 상태 값

| 상태 | 설명 |
|---|---|
| `Established` | BGP 세션 정상 수립, 경로 교환 중 |
| `OpenSent` | OPEN 메시지 전송 후 응답 대기 |
| `Active` | 연결 시도 중 (피어에 도달 불가) |
| `Idle` | 세션 비활성 (오류 후 대기) |

---

## 트러블슈팅

```bash
# BGP 로그 확인
kubectl -n kube-system logs ds/cilium -c cilium-agent | grep -i bgp

# 피어가 Established가 아닐 때
# 1. 네트워크 연결 확인 (포트 179/TCP)
kubectl -n kube-system exec ds/cilium -- \
  curl -v telnet://192.168.1.1:179

# 2. AS 번호 확인
# 3. 피어 IP 주소 확인
# 4. 보안그룹에서 포트 179 허용 여부 확인
```

---

## 참고 링크

- [Cilium BGP Control Plane 문서](https://docs.cilium.io/en/stable/network/bgp-control-plane/)
- [CiliumBGPPeeringPolicy API](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-v2/)
- [LB IPAM](https://docs.cilium.io/en/stable/network/lb-ipam/)
