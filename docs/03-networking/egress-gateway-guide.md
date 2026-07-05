# Egress Gateway

Egress Gateway는 특정 파드의 외부 트래픽을 **고정된 게이트웨이 노드의 IP**를 통해 내보내는 기능입니다.
외부 방화벽에서 출발지 IP 기반으로 허용 목록을 관리할 때 유용합니다.

---

## 사용 시나리오

```
문제 상황:
  파드 IP는 재시작마다 변경됨
  → 외부 SaaS API의 IP 화이트리스트에 등록 불가
  → DB 방화벽 규칙 관리 어려움

해결책 (Egress Gateway):
  파드 → Egress Gateway 노드 → 고정 Elastic IP → 외부 서비스
         (NAT SNAT를 게이트웨이 노드에서 수행)
```

```
파드 A (IP: 10.0.1.5)  ─┐
파드 B (IP: 10.0.2.7)  ─┤  → Gateway 노드 (EIP: 52.x.x.x) → 외부 API
파드 C (IP: 10.0.3.9)  ─┘
```

---

## 활성화

```yaml
# helm/values.yaml
egressGateway:
  enabled: true
```

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --values my-values.yaml \
  --reuse-values
```

---

## EIP 및 게이트웨이 노드 준비 (AWS)

### 1. Elastic IP 생성

```bash
# EIP 할당
aws ec2 allocate-address --domain vpc

# 출력 예시:
# {
#   "PublicIp": "52.x.x.x",
#   "AllocationId": "eipalloc-XXXXXXXX"
# }
```

### 2. 게이트웨이 노드 지정

특정 노드를 게이트웨이 전용으로 지정합니다.

```bash
# 노드에 레이블 추가
kubectl label node <NODE_NAME> \
  egress-gateway=true \
  egress-gateway/eip=52.x.x.x

# 노드에 EIP 연결 (해당 노드의 기본 ENI)
INSTANCE_ID=$(kubectl get node <NODE_NAME> \
  -o jsonpath='{.spec.providerID}' | cut -d'/' -f5)
INTERFACE_ID=$(aws ec2 describe-instances \
  --instance-ids ${INSTANCE_ID} \
  --query "Reservations[].Instances[].NetworkInterfaces[0].NetworkInterfaceId" \
  --output text)
aws ec2 associate-address \
  --allocation-id eipalloc-XXXXXXXX \
  --network-interface-id ${INTERFACE_ID}
```

---

## CiliumEgressGatewayPolicy 설정

```yaml
# examples/egress-gateway.yaml
apiVersion: cilium.io/v2
kind: CiliumEgressGatewayPolicy
metadata:
  name: production-egress
spec:
  selectors:
    - podSelector:
        matchLabels:
          app: backend           # 이 레이블의 파드에 적용
        namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: production

  destinationCIDRs:
    - "203.0.113.0/24"          # 외부 API 대역 (또는 0.0.0.0/0 전체)

  egressGateway:
    nodeSelector:
      matchLabels:
        egress-gateway: "true"   # 게이트웨이 노드 선택
    egressIP: "52.x.x.x"        # 사용할 EIP
```

```bash
kubectl apply -f examples/egress-gateway.yaml

# 정책 확인
kubectl get ciliumnodeconfigs,ciliumegressgatewaypolicies -A
```

---

## 동작 흐름

```
[production 네임스페이스의 backend 파드]
    │
    │ 외부 203.0.113.x 로 요청 발생
    ▼
Cilium Agent: EgressGatewayPolicy 매칭 확인
    │
    ▼
트래픽을 게이트웨이 노드로 터널링
    │
    ▼
게이트웨이 노드에서 SNAT (소스 IP → 52.x.x.x)
    │
    ▼
외부 서비스 수신 (소스 IP: 52.x.x.x)
```

---

## 고가용성 — 다중 게이트웨이 노드

게이트웨이 노드 장애에 대비하여 복수의 노드를 지정할 수 있습니다.

```bash
# 두 번째 게이트웨이 노드 지정
kubectl label node <NODE_NAME_2> \
  egress-gateway=true

# 두 번째 EIP 연결 (별도 진행)
```

```yaml
# 동일 nodeSelector로 여러 노드 자동 선택
egressGateway:
  nodeSelector:
    matchLabels:
      egress-gateway: "true"    # 레이블을 가진 노드 중 하나 선택
```

> 현재 Cilium Egress Gateway는 Active/Passive 방식으로 동작합니다.
> 활성 게이트웨이 노드 장애 시 다른 노드로 페일오버됩니다.

---

## 검증

```bash
# 게이트웨이를 통한 외부 트래픽 확인 (Hubble)
hubble observe \
  --from-namespace production \
  --verdict FORWARDED \
  --follow

# 파드에서 외부 IP 확인
kubectl exec -n production deploy/backend -- \
  curl -s https://api.ipify.org
# → 52.x.x.x (EIP 주소가 반환되어야 함)

# BPF 맵에서 Egress 정책 확인
kubectl -n kube-system exec ds/cilium -- \
  cilium bpf egress list
```

---

## 주의사항

| 항목 | 내용 |
|---|---|
| EIP 비용 | 연결되지 않은 EIP는 AWS에서 별도 과금 |
| 게이트웨이 노드 용량 | 모든 파드 트래픽이 집중되므로 충분한 인스턴스 타입 필요 |
| ENI 제한 | 노드당 ENI/IP 제한 내에서 운영 |
| 정책 범위 | `destinationCIDRs: 0.0.0.0/0` 사용 시 내부 트래픽도 영향받을 수 있음 |

---

## 참고 링크

- [Cilium Egress Gateway 공식 문서](https://docs.cilium.io/en/stable/network/egress-gateway/)
- [AWS EIP + Egress 가이드](https://docs.cilium.io/en/stable/network/egress-gateway/#aws-elastic-ip)
