# CLAUDE.md — cilium-practice 지식 베이스

EKS + Cilium 운영 경험 기반의 개인 지식 베이스입니다. 문서 추가/수정 시 아래 가이드를 따릅니다.

## 프로젝트 설정

- **환경**: EKS
- **Cilium 버전**: 1.16.x (cilium Helm chart)
- **Cilium 네임스페이스**: `kube-system`
- **Application 네임스페이스**: `default`
- **CNI 모드**: ENI 모드 (AWS VPC 네이티브)
- **kube-proxy 대체**: BPF kube-proxy replacement 사용
- **앱 이름 컨벤션**: `my-app`

---

## 프로젝트 구조

```
cilium-practice/
├── docs/                              # 지식 문서 (카테고리별 분류)
│   ├── install/            (1개)     # Cilium 설치, Helm 설정
│   ├── architecture/       (1개)     # eBPF, Agent, Operator, Hubble 아키텍처
│   ├── security/           (1개)     # NetworkPolicy, CiliumNetworkPolicy, Zero Trust
│   ├── observability/      (1개)     # Hubble CLI/UI/Relay, Prometheus 메트릭
│   ├── networking/         (3개)     # 로드밸런싱, Egress Gateway, BGP
│   ├── service-mesh/       (1개)     # Cilium Service Mesh, mTLS, L7 정책
│   └── operations/         (2개)     # 트러블슈팅, E2E 실습
│
├── examples/                          # Cilium/K8s 리소스 YAML 예제
│   ├── network-policy.yaml
│   ├── cilium-network-policy.yaml
│   └── egress-gateway.yaml
│
├── helm/                              # Helm values
│   ├── values.yaml                    # 개발/테스트용
│   └── values-prod.yaml              # 운영용
│
├── templates/                         # 재사용 문서 템플릿
│   ├── service-doc.md                 # 서비스 Cilium 구성 문서
│   ├── runbook.md                     # 운영 Runbook
│   └── incident-report.md            # 장애 보고서
│
├── rules/                             # Claude 작성 규칙
│   ├── doc-writing.md                 # 문서 스타일 가이드
│   ├── cilium-conventions.md          # YAML/명령어 코드 규칙
│   ├── security-checklist.md          # 보안 검토 체크리스트
│   └── monitoring.md                  # 모니터링/확인 작성 기준
│
├── agents/                            # Claude 전문 에이전트
│   ├── doc-writer.md                  # 문서 작성 에이전트
│   ├── network-advisor.md             # 네트워크 아키텍처 설계/검토 에이전트
│   ├── traffic-analyzer.md            # 트래픽 분석 에이전트
│   └── security-reviewer.md           # 보안 정책 검토 에이전트
│
└── .claude/
    ├── settings.json                  # 프로젝트 공유 설정
    └── commands/                      # 커스텀 슬래시 커맨드
        ├── new-doc.md                 # /new-doc
        ├── new-runbook.md             # /new-runbook
        ├── review-doc.md              # /review-doc
        ├── add-troubleshooting.md     # /add-troubleshooting
        └── search-kb.md              # /search-kb
```

---

## 커스텀 슬래시 커맨드

| 커맨드 | 사용법 | 설명 |
|--------|--------|------|
| `/new-doc` | `/new-doc networking wireguard-encryption` | 신규 문서 스캐폴딩 |
| `/new-runbook` | `/new-runbook operations cilium-upgrade` | 운영 Runbook 생성 |
| `/review-doc` | `/review-doc docs/security/network-policy-guide.md` | 문서 품질 검토 |
| `/add-troubleshooting` | `/add-troubleshooting docs/networking/load-balancing-guide.md <증상>` | 트러블슈팅 추가 |
| `/search-kb` | `/search-kb WireGuard 암호화` | 지식 베이스 키워드 검색 |

---

## 파일 네이밍 규칙

```
docs/{카테고리}/{주제}.md
```

- 카테고리: `install`, `architecture`, `security`, `observability`, `networking`, `service-mesh`, `operations`
- 주제: 소문자 영어, 하이픈 구분
- 예시: `docs/networking/wireguard-encryption-guide.md`, `docs/security/fqdn-egress-policy.md`

---

## 문서 작성 원칙

1. **실제 경험 기반** — 운영 중 실제로 겪은 이슈와 해결 방법 위주
2. **재현 가능한 코드** — YAML, kubectl/cilium/hubble 명령어 복붙 즉시 적용 가능
3. **원인 중심 트러블슈팅** — 증상만 나열하지 말고 근본 원인 설명
4. **한국어 기술 문서** — 주요 개념은 영어 원문 병기
5. **모니터링 필수** — 모든 문서에 `cilium` CLI / Hubble 진단 명령어 포함

세부 규칙은 `rules/` 디렉토리를 참조합니다.

---

## 카테고리별 문서 목록

### docs/install/
| 파일 | 주제 |
|------|------|
| `install.md` | EKS에 Cilium 설치 (Helm, ENI 모드, kube-proxy 대체) |

### docs/architecture/
| 파일 | 주제 |
|------|------|
| `architecture-guide.md` | Cilium 아키텍처 — eBPF, Agent, Operator, Hubble, ENI 모드 |

### docs/security/
| 파일 | 주제 |
|------|------|
| `network-policy-guide.md` | 네트워크 정책 — K8s NetworkPolicy, CiliumNetworkPolicy, Zero Trust |

### docs/observability/
| 파일 | 주제 |
|------|------|
| `hubble-guide.md` | Hubble — 실시간 네트워크 흐름 관찰, UI, Prometheus 메트릭 |

### docs/networking/
| 파일 | 주제 |
|------|------|
| `load-balancing-guide.md` | BPF 로드밸런싱 — kube-proxy 대체, DSR, Maglev 해싱 |
| `egress-gateway-guide.md` | Egress Gateway — 파드 트래픽을 고정 IP로 외부 라우팅 |
| `bgp-guide.md` | BGP Control Plane — LoadBalancer IP를 BGP로 광고 |

### docs/service-mesh/
| 파일 | 주제 |
|------|------|
| `service-mesh-guide.md` | Cilium Service Mesh — 사이드카 없는 mTLS, L7 정책, 트래픽 관리 |

### docs/operations/
| 파일 | 주제 |
|------|------|
| `troubleshooting-guide.md` | 자주 발생하는 문제와 진단 방법 |
| `e2e-practice.md` | End-to-End 실습 (설치 → 정책 → Hubble 관찰) |
