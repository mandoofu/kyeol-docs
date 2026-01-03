# 📚 KYEOL Saleor 프로젝트 문서

> **Saleor e-Commerce 플랫폼의 Multi-Environment(DEV/STAGE/PROD) 배포를 위한 기술 문서 저장소**

## 🏗️ 전체 아키텍처

![KYEOL Saleor Architecture](architecture.png)

---

## 👋 처음 오신 분들을 위한 안내

이 프로젝트가 처음이시라면, **본인의 역할/목적에 따라** 아래 가이드를 참고하세요.

---

### 🧱 "처음부터 인프라를 구축해야 합니다"

> AWS 계정에 아무것도 없는 상태에서 시작하는 경우

| 순서 | 문서 | 설명 |
|:----:|------|------|
| 1 | [runbook-phase1-dev-mgmt.md](runbook-phase1-dev-mgmt.md) | Bootstrap, MGMT, DEV 환경 구축 |
| 2 | [runbook-phase2-stage-prod.md](runbook-phase2-stage-prod.md) | STAGE, PROD 환경 구축 및 앱 배포 |

**예상 소요 시간**: 2-3시간 (처음인 경우)

---

### 🚀 "인프라는 있고, 앱만 배포하려고 합니다"

> EKS, RDS, Valkey 등 인프라가 이미 있고, Storefront/Dashboard만 배포하려는 경우

| 문서 | 설명 |
|------|------|
| [runbook-phase2-stage-prod.md](runbook-phase2-stage-prod.md) **7. GitOps 앱 배포** 섹션 | Kustomize로 앱 배포 |

```powershell
# 빠른 배포 (STAGE 예시)
kubectl apply -k apps/saleor/overlays/stage --context stage
kubectl apply -k apps/saleor-dashboard/overlays/stage --context stage
```

---

### 🛠 "Ingress / ALB / DNS 문제를 해결 중입니다"

> 배포는 됐는데 URL 접속이 안 되거나, Ingress ADDRESS가 없는 경우

| 문서 | 섹션 |
|------|------|
| [runbook-phase2-stage-prod.md](runbook-phase2-stage-prod.md) | **9. 트러블슈팅** |
| [runbook-phase2-stage-prod-troubleshooting.md](runbook-phase2-stage-prod-troubleshooting.md) | 상세 이슈별 해결 방법 |

**자주 발생하는 문제**:
- ❌ Ingress ADDRESS 미생성 → ALB Controller IRSA 권한 확인 (9.2 섹션)
- ❌ 502 Bad Gateway → Pod 상태 및 헬스체크 확인 (9.3 섹션)
- ❌ ImagePullBackOff → ECR 이미지 경로/태그 확인

---

### 📊 "운영 / 확장 / 비용 관점에서 검토 중입니다"

> 아키텍처, 비용 구조, 스케일링 정책을 이해하려는 경우

| 문서 | 설명 |
|------|------|
| [spec.md](spec.md) | 프로젝트 요구사항 및 아키텍처 스펙 |
| [decision-log.md](decision-log.md) | 주요 기술 결정 사항 및 근거 |
| [variables-required.md](variables-required.md) | 환경별 필수 변수 목록 |

---

## 📁 문서 목록

| 파일명 | 용도 | 대상 |
|--------|------|------|
| `runbook-phase1-dev-mgmt.md` | Bootstrap/MGMT/DEV 구축 가이드 | 인프라 담당자 |
| `runbook-phase2-stage-prod.md` | STAGE/PROD 구축 및 앱 배포 가이드 | 인프라/배포 담당자 |
| `runbook-phase2-stage-prod-troubleshooting.md` | 상세 트러블슈팅 | 운영 담당자 |
| `spec.md` | 요구사항 및 아키텍처 | 전체 팀 |
| `decision-log.md` | 기술 결정 로그 | 아키텍트 |
| `variables-required.md` | 필수 변수 참조 | 모든 담당자 |

---

## 🔗 관련 레포지토리

| 레포지토리 | 역할 | 언제 사용 |
|-----------|------|----------|
| [kyeol-infra-terraform](https://github.com/mandoofu/kyeol-infra-terraform) | AWS 인프라 IaC | 인프라 생성/수정 시 |
| [kyeol-platform-gitops](https://github.com/mandoofu/kyeol-platform-gitops) | Helm Addons (ALB Controller, ExternalDNS) | 클러스터 애드온 설치 시 |
| [kyeol-app-gitops](https://github.com/mandoofu/kyeol-app-gitops) | Storefront/Dashboard Kustomize | 앱 배포 시 |
| [kyeol-storefront](https://github.com/mandoofu/kyeol-storefront) | Storefront 소스 코드 | 프론트엔드 개발 시 |
| [kyeol-saleor-dashboard](https://github.com/mandoofu/kyeol-saleor-dashboard) | Dashboard 소스 코드 | 대시보드 개발 시 |

---

## ⚠️ 주의사항

1. **terraform.tfvars는 절대 Git에 커밋하지 마세요** - 민감 정보 포함
2. **PROD 배포 전 반드시 STAGE에서 검증하세요**
3. **IAM 권한 변경 시 반드시 코드로 관리하세요** - 수동 콘솔 수정 금지

---

> **마지막 업데이트**: 2026-01-03
