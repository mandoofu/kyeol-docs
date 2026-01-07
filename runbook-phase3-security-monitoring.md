# Phase 3 보안 & 모니터링 런북 v4.0

> **버전**: 4.0  
> **작성일**: 2026-01-07  
> **상태**: VPC Peering + Private Endpoint 추가

---

## 아키텍처 (최종)

```
Users → Route53 → WAF (Global) → CloudFront → origin-{env}.domain → ALB (각 VPC)
                     │
              ┌──────┴──────┐
              │    MGMT     │  ← WAF + CloudFront + ArgoCD 생성 위치
              └──────┬──────┘
                     │ VPC Peering (사설망)
     ┌───────────────┼───────────────┐
     │               │               │
┌────┴────┐    ┌────┴────┐    ┌────┴────┐
│   DEV   │    │  STAGE  │    │  PROD   │
│  EKS    │    │  EKS    │    │  EKS    │
│(Private)│    │(Private)│    │(Private)│
└─────────┘    └─────────┘    └─────────┘
```

---

## 1. 실행 순서

| 순서 | 환경 | 작업 |
|:----:|:----:|------|
| 0 | 사전 준비 | us-east-1 ACM 인증서 생성 (CloudFront용) |
| 1 | MGMT | Global WAF + CloudFront + **VPC Peering Outputs** 생성 |
| 2 | DEV | VPC Endpoints + S3 + Fluent Bit IRSA + **VPC Peering + Private Endpoint** |
| 3 | STAGE | VPC Endpoints + S3 + Fluent Bit IRSA + **VPC Peering + Private Endpoint** |
| 4 | PROD | VPC Endpoints + S3 + Fluent Bit IRSA + CloudTrail + **VPC Peering + Private Endpoint** |

---

## 1.5. 사전 준비: us-east-1 ACM 인증서 생성

> ⚠️ **CloudFront는 us-east-1 리전의 ACM 인증서만 사용 가능**

### 현재 인증서 확인

```powershell
# 시드니 ACM 확인 (현재 있는 인증서)
aws acm list-certificates --region ap-southeast-2

# 버지니아 ACM 확인 (CloudFront용 필요)
aws acm list-certificates --region us-east-1
```

### 버지니아 ACM 생성 (도메인: msp-g1.click)

```powershell
# 와일드카드 인증서 요청
aws acm request-certificate `
  --region us-east-1 `
  --domain-name "msp-g1.click" `
  --subject-alternative-names "*.msp-g1.click" `
  --validation-method DNS
```

**예상 결과**:
```json
{
    "CertificateArn": "arn:aws:acm:us-east-1:827913617839:certificate/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### DNS 검증 레코드 조회

```powershell
# 인증서 ARN으로 검증 정보 조회
aws acm describe-certificate `
  --region us-east-1 `
  --certificate-arn "위에서_받은_ARN"
```

### Route53에 DNS 검증 레코드 추가

```powershell
# 도메인 검증용 CNAME 레코드 자동 생성 (Route53 사용 시)
aws acm describe-certificate `
  --region us-east-1 `
  --certificate-arn "arn:aws:acm:us-east-1:827913617839:certificate/xxx" `
  --query "Certificate.DomainValidationOptions[0].ResourceRecord"
```

**출력된 CNAME Name/Value를 Route53에 추가하거나, 아래 명령으로 자동 추가**:

```powershell
# Hosted Zone ID 확인
aws route53 list-hosted-zones --query "HostedZones[?Name=='msp-g1.click.'].Id" --output text

# CNAME 레코드 생성 (수동) - Route53 콘솔에서 추가 권장
```

### 인증서 발급 대기 (5~30분)

```powershell
# 상태 확인 (ISSUED 될 때까지 대기)
aws acm describe-certificate `
  --region us-east-1 `
  --certificate-arn "arn:aws:acm:us-east-1:827913617839:certificate/xxx" `
  --query "Certificate.Status"
```

**ISSUED 확인 후** 다음 단계로 진행

---

## 2. MGMT 환경 (Global WAF + CloudFront + VPC Peering Outputs)

### Step 1: 디렉토리 이동

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\mgmt
```

### Step 2: tfvars 준비

```powershell
cp terraform.tfvars.example terraform.tfvars
```

**실제 값 입력**:
```hcl
aws_account_id     = "실제_계정_ID"
hosted_zone_id     = "실제_Zone_ID"
cloudfront_acm_arn = "arn:aws:acm:us-east-1:계정ID:certificate/..."
```

### Step 3: Terraform 실행

```powershell
terraform init
terraform plan
```

**예상 결과**:
```
Plan: 2 to add, 0 to change, 0 to destroy.
  + module.waf_global.aws_wafv2_web_acl.global
  + module.cloudfront.aws_cloudfront_distribution.main
```

```powershell
terraform apply -auto-approve
```

### Step 4: 검증

```powershell
terraform output global_waf_arn
terraform output cloudfront_domain_name

# VPC Peering용 outputs 확인 (DEV/STAGE/PROD에서 참조)
terraform output vpc_id
terraform output vpc_cidr
terraform output all_route_table_ids
```

---

## 3. DEV 환경 (VPC Endpoints + S3 + Fluent Bit + VPC Peering)

### Step 1: 디렉토리 이동

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\dev
```

### Step 2: tfvars 준비

```powershell
cp terraform.tfvars.example terraform.tfvars
```

**실제 값 입력**:
```hcl
aws_account_id = "실제_계정_ID"
hosted_zone_id = "실제_Zone_ID"

# =============================================================================
# VPC Peering 설정 (MGMT ↔ DEV)
# ArgoCD(MGMT)가 DEV EKS Private Endpoint로 접근하기 위해 활성화
# =============================================================================
enable_vpc_peering     = true
terraform_state_bucket = "your-terraform-state-bucket"  # 예: "min-kyeol-terraform-state"

# =============================================================================
# EKS Endpoint 설정 (Private Endpoint 전환)
# =============================================================================
endpoint_private_access = true
endpoint_public_access  = false  # Private만 사용 (보안 강화)

# 또는 Public을 관리자 IP만 허용:
# endpoint_public_access = true
# public_access_cidrs    = ["YOUR_ADMIN_IP/32"]

# MGMT VPC CIDR (ArgoCD에서 EKS API 접근 허용)
mgmt_vpc_cidrs = ["10.40.0.0/16"]
```

### Step 3: Terraform 실행

```powershell
terraform init -upgrade
terraform plan
```

**예상 결과**:
```
Plan: X to add, Y to change, 0 to destroy.
  + aws_vpc_endpoint.s3
  + module.s3_phase3...
  + aws_iam_role.fluent_bit
  + module.vpc_peering_mgmt.aws_vpc_peering_connection.main
  + module.vpc_peering_mgmt.aws_route.requester_to_accepter
  + module.vpc_peering_mgmt.aws_route.accepter_to_requester
  ~ module.eks.aws_eks_cluster.main (resourcesVpcConfig.endpointPublicAccess)
```

```powershell
terraform apply -auto-approve
```

### Step 4: 검증

```powershell
# S3 버킷 확인
aws s3 ls | Select-String "kyeol-dev"

# VPC Endpoint 확인
aws ec2 describe-vpc-endpoints --region ap-southeast-2 `
  --filters "Name=tag:Name,Values=*dev*" `
  --query "VpcEndpoints[*].{Name:Tags[?Key=='Name']|[0].Value,State:State}" `
  --output table

# Fluent Bit Role 확인
terraform output fluent_bit_role_arn

# VPC Peering 확인
aws ec2 describe-vpc-peering-connections `
  --region ap-southeast-2 `
  --filters "Name=status-code,Values=active" `
  --query "VpcPeeringConnections[*].[VpcPeeringConnectionId,AccepterVpcInfo.CidrBlock,RequesterVpcInfo.CidrBlock]" `
  --output table

# EKS Endpoint 설정 확인
aws eks describe-cluster --name min-kyeol-dev-eks `
  --query "cluster.resourcesVpcConfig.[endpointPrivateAccess,endpointPublicAccess]" `
  --output table
```

---

## 4. STAGE 환경

DEV와 동일 (경로만 `envs/stage`로 변경)

**terraform.tfvars 추가 설정**:
```hcl
enable_vpc_peering      = true
terraform_state_bucket  = "your-terraform-state-bucket"
endpoint_private_access = true
endpoint_public_access  = false
mgmt_vpc_cidrs          = ["10.40.0.0/16"]
```

---

## 5. PROD 환경

DEV와 동일 + CloudTrail 활성화

```hcl
# terraform.tfvars에 추가
enable_cloudtrail = true

# VPC Peering + Private Endpoint
enable_vpc_peering      = true
terraform_state_bucket  = "your-terraform-state-bucket"
endpoint_private_access = true
endpoint_public_access  = false
mgmt_vpc_cidrs          = ["10.40.0.0/16"]
```

**CloudTrail 검증**:
```powershell
terraform output cloudtrail_arn
aws cloudtrail describe-trails --region ap-southeast-2
```

---

## 6. ArgoCD 클러스터 등록 (Private Endpoint)

> ℹ️ VPC Peering 적용 후 ArgoCD에서 각 클러스터를 Private Endpoint로 등록

### 등록 절차

```bash
# 1. MGMT 클러스터에서 kubectl 접근 확인
kubectl config use-context mgmt

# 2. 대상 클러스터 kubeconfig 가져오기
aws eks update-kubeconfig \
  --region ap-southeast-2 \
  --name min-kyeol-dev-eks \
  --alias dev-private

# 3. ArgoCD CLI로 클러스터 등록
argocd cluster add dev-private --name kyeol-dev

# 4. 등록 확인
argocd cluster list
```

### Sync 테스트

```bash
# 애플리케이션 Sync 테스트
argocd app sync saleor-dev --prune

# 상태 확인
argocd app get saleor-dev
```

---

## 7. 검증 체크리스트

| 항목 | 검증 명령 | 예상 결과 |
|------|----------|----------|
| Global WAF | `terraform output -state=../mgmt/terraform.tfstate global_waf_arn` | WAF ARN |
| CloudFront | `aws cloudfront list-distributions` | 배포 목록 |
| S3 Endpoint | `aws ec2 describe-vpc-endpoints` | available |
| Fluent Bit Role | `terraform output fluent_bit_role_arn` | IAM Role ARN |
| CloudTrail | `aws cloudtrail describe-trails` | Trail 정보 |
| **VPC Peering** | `aws ec2 describe-vpc-peering-connections` | active |
| **EKS Private** | `aws eks describe-cluster --name xxx --query "cluster.resourcesVpcConfig"` | endpointPrivateAccess: true |

---

## 8. 롤백

### MGMT 롤백

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\mgmt

# terraform.tfvars 수정
enable_global_waf = false
enable_cloudfront = false

terraform apply -auto-approve
```

### DEV/STAGE/PROD 롤백

```powershell
# terraform.tfvars 수정
enable_s3_endpoint     = false
enable_phase3_s3       = false
enable_fluent_bit_irsa = false
enable_cloudtrail      = false
enable_vpc_peering     = false

# EKS Public Endpoint 복원
endpoint_public_access = true

terraform apply -auto-approve
```

### EKS Endpoint 즉시 롤백 (AWS CLI)

```powershell
# Public Endpoint 즉시 재활성화 (5-10분 소요)
aws eks update-cluster-config `
  --region ap-southeast-2 `
  --name min-kyeol-dev-eks `
  --resources-vpc-config endpointPublicAccess=true,endpointPrivateAccess=true

# 상태 확인
aws eks describe-cluster --name min-kyeol-dev-eks `
  --query "cluster.resourcesVpcConfig"
```

---

## 9. 비용 산출 (VPC Peering)

| 항목 | 월 비용 |
|------|--------|
| **VPC Peering 트래픽** (Cross-AZ) | ~$2.60 |
| Transit Gateway 대비 절감 | ~$140 |

> 💡 **비용 최적화 팁**
> - ArgoCD Sync 주기 조정: 기본 3분 → 5분 (`ARGOCD_RECONCILIATION_TIMEOUT`)
> - Webhook 활성화로 Poll 대신 Push 기반 Sync

---

## 파일 경로

| 환경 | 경로 | Phase 3 리소스 |
|------|------|---------------|
| MGMT | `envs/mgmt/` | Global WAF + CloudFront + **VPC Peering Outputs** |
| DEV | `envs/dev/` | VPC Endpoints, S3, Fluent Bit IRSA + **VPC Peering + Private Endpoint** |
| STAGE | `envs/stage/` | VPC Endpoints, S3, Fluent Bit IRSA + **VPC Peering + Private Endpoint** |
| PROD | `envs/prod/` | VPC Endpoints, S3, Fluent Bit IRSA, CloudTrail + **VPC Peering + Private Endpoint** |

---

**운영 반영 전 테스트 환경에서 먼저 검증하세요.**

---

## 다음 단계

Phase 3 완료 후 **Phase 4: 로그 분석 자동화 파이프라인**을 진행하세요.

📋 [Phase 4 런북: 로그 분석 자동화](./runbook-phase4-log-analytics.md)
- EventBridge 스케줄링
- Athena 로그 쿼리
- Bedrock AI 분석
- Slack 알림 자동화


