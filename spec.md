<!-- docs/spec.md -->

# KYEOL Saleor 멀티환경 클라우드 아키텍처 스펙 (Anti-Gravity Reference)
- 기준일: 2025-12-29
- 목적: VSCode Agent/MCP가 “흔들리지 않도록” 변경 불가 기준(고정 스펙)만 제공

---

## 0) 프로젝트 목표(고정)
- Saleor 기반 이커머스 멀티환경(DEV/STAGE/PROD/MGMT) 아키텍처 구현
- 환경별 동일 UI/서비스를 운영하고, 도메인으로 환경 분리
- MSP 운영 수준(IaC + GitOps + CI/CD + 보안/운영 고려)으로 재현 가능해야 함

---

## 1) 환경/도메인 정책(고정)

### 1.1 환경
- `dev`, `stage`, `prod`, `mgmt` 총 4개 환경
- Phase-1에서는 **DEV + MGMT만 적용 대상**
  - STAGE/PROD는 **코드 템플릿만 생성**, apply/runbook에서는 비활성/미사용 처리

### 1.2 도메인
- 메인 도메인: `msp-g1.click`
- 서비스 도메인(CloudFront 앞단)
  - DEV: `dev-kyeol.msp-g1.click`
  - STAGE: `stage-kyeol.msp-g1.click`
  - PROD: `kyeol.msp-g1.click`
- 오리진 도메인(CloudFront Origin)
  - DEV: `origin-dev-kyeol.msp-g1.click`
  - STAGE: `origin-stage-kyeol.msp-g1.click`
  - PROD: `origin-prod-kyeol.msp-g1.click`
- 규칙: **CloudFront Origin은 항상 origin-* 도메인**을 사용(ALB DNS 직접 사용 금지)

---

## 2) 핵심 원칙(고정)

### 2.1 ALB 생성 원칙
- ALB는 Terraform으로 직접 생성하지 않는다.
- **EKS + AWS Load Balancer Controller + Ingress**로 ALB가 자동 생성되어야 한다.

### 2.2 DNS 자동화 원칙
- Route53 레코드는 **ExternalDNS**가 자동 관리한다.
- 특히 `origin-*.msp-g1.click` 레코드를 ExternalDNS가 자동 생성/갱신해야 한다.

### 2.3 NAT 원칙
- VPC별 NAT는 **VPC당 1개**로 구성
- **Regional NAT Gateway + Elastic IP 수동 지정(manual mode)**

### 2.4 CloudFront 원칙
- CloudFront 배포판은 **DEV/STAGE/PROD 총 3개**
- Phase-1에서는 CloudFront는 선택(계획만 문서화 가능), 우선 목표는 origin-dev 도메인으로 UI 동작 확인

---

## 3) 리전/프로바이더/인증서(고정)

### 3.1 리전
- 기본 리전: **ap-southeast-2 (Sydney)**
- AZ 표기는 `a`, `c` 사용:
  - `ap-southeast-2a`
  - `ap-southeast-2c`

### 3.2 ACM 인증서
- CloudFront 인증서: **us-east-1** (필수)
- ALB(Ingress) 인증서: **ap-southeast-2**
- 권장: `*.msp-g1.click` 와일드카드 + 필요한 SAN 관리

---

## 4) CIDR/서브넷(고정 표)

### 4.1 DEV (컴퓨트 1AZ + Public만 2AZ, cache 없음)
- VPC: `10.10.0.0/16`
- Public: 2AZ (a, c)
- App/Data: 1AZ (a)
- Subnets:
  - public-a: `10.10.0.0/24`
  - public-c: `10.10.1.0/24`
  - app-private-a: `10.10.4.0/22`
  - data-private-a: `10.10.9.0/24`
- DEV에는 cache subnet/리소스를 **생성하지 않는다**.

### 4.2 STAGE (2AZ, 4티어) — 템플릿만
- VPC: `10.20.0.0/16`
- Subnets:
  - public-a: `10.20.0.0/24`
  - public-c: `10.20.1.0/24`
  - app-private-a: `10.20.4.0/22`
  - app-private-c: `10.20.12.0/22`
  - cache-private-a: `10.20.8.0/26`
  - cache-private-c: `10.20.8.64/26`
  - data-private-a: `10.20.9.0/24`
  - data-private-c: `10.20.10.0/24`

### 4.3 PROD (2AZ, 4티어) — 템플릿만
- VPC: `10.30.0.0/16`
- Subnets:
  - public-a: `10.30.0.0/24`
  - public-c: `10.30.1.0/24`
  - app-private-a: `10.30.4.0/22`
  - app-private-c: `10.30.12.0/22`
  - cache-private-a: `10.30.8.0/26`
  - cache-private-c: `10.30.8.64/26`
  - data-private-a: `10.30.9.0/24`
  - data-private-c: `10.30.10.0/24`

### 4.4 MGMT (2AZ)
- VPC: `10.40.0.0/16`
- Subnets:
  - public-a: `10.40.0.0/24`
  - public-c: `10.40.1.0/24`
  - ops-private-a: `10.40.4.0/22`
  - ops-private-c: `10.40.12.0/22`

---

## 5) 네이밍 컨벤션(고정)

### 5.1 기본 규칙
- 기본 컨벤션: `kyeol-[env]-[resource]-[detail]-[id]`
- 환경(env): `prod`, `stage`, `dev`, `mgmt`
- 소문자 + 하이픈(-)

### 5.2 추가 프리픽스(필수)
- 모든 AWS 리소스(Name tag, 식별자, 물리 이름)에 **`min-`** 프리픽스를 반드시 붙인다.
- 최종 네이밍: `min-kyeol-[env]-...`

### 5.3 예시
- VPC: `min-kyeol-dev-vpc`
- Subnet: `min-kyeol-dev-sub-pub-a`
- NAT GW: `min-kyeol-dev-nat-a`
- EKS: `min-kyeol-dev-eks`
- SG: `min-kyeol-dev-sg-alb`

### 5.4 S3(글로벌 유니크 필수)
- `min-kyeol-[env]-s3-[usage]-<account>-ap-southeast-2`
  - 예: `min-kyeol-dev-s3-tfstate-123456789012-ap-southeast-2`

---

## 6) 권장 사이징(테스트/포트폴리오 기준)
- EKS NodeGroup(예시):
  - DEV: t3.medium, desired 2 (min 1, max 2)
  - MGMT: t3.medium, desired 2 (min 1, max 3)
- RDS(Postgres):
  - DEV: db.t3.small, Multi-AZ ❌
  - STAGE/PROD: db.t3.medium, Multi-AZ ✅ (템플릿)
- Cache(Valkey/Redis):
  - DEV: cache.t3.micro (단일, 선택)
  - STAGE/PROD: cache.t3.small (단일/선택)

---

## 7) EKS 필수 애드온(고정)
- AWS Load Balancer Controller (IRSA)
- ExternalDNS (IRSA)
- (권장) metrics-server

---

## 8) Ingress 표준(고정 템플릿 개념)
- host는 origin-* 도메인 사용
- TLS는 ALB 인증서(ap-southeast-2 ACM ARN)로 설정
- ExternalDNS hostname annotation 사용

---

## 9) 산출물(고정)
- IaC: `kyeol-infra-terraform` (bootstrap/dev/mgmt + modules + global/us-east-1 템플릿)
- Platform GitOps: `kyeol-platform-gitops` (ArgoCD + addons)
- App GitOps: `kyeol-app-gitops` (Saleor base + overlays)
- 문서:
  - `docs/runbook-phase1-dev-mgmt.md`
  - `docs/variables-required.md`
  - `docs/decision-log.md`
  - `INDEX.md`
