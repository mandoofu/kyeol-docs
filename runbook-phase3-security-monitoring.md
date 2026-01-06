# Phase 3 보안 & 모니터링 런북 v3.0

> **버전**: 3.0  
> **작성일**: 2026-01-05  
> **상태**: 코드 정리 완료

---

## 아키텍처 (최종)

```
Users → Route53 → WAF (Global) → CloudFront → origin-{env}.domain → ALB (각 VPC)
                     │
              ┌──────┴──────┐
              │    MGMT     │  ← WAF + CloudFront 생성 위치
              └─────────────┘
```

---

## 1. 실행 순서

| 순서 | 환경 | 작업 |
|:----:|:----:|------|
| 0 | 사전 준비 | us-east-1 ACM 인증서 생성 (CloudFront용) |
| 1 | MGMT | Global WAF + CloudFront 생성 |
| 2 | DEV | VPC Endpoints + S3 + Fluent Bit IRSA |
| 3 | STAGE | VPC Endpoints + S3 + Fluent Bit IRSA |
| 4 | PROD | VPC Endpoints + S3 + Fluent Bit IRSA + CloudTrail |

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

## 2. MGMT 환경 (Global WAF + CloudFront)

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
```

---

## 3. DEV 환경 (VPC Endpoints + S3 + Fluent Bit)

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
```

### Step 3: Terraform 실행

```powershell
terraform init
terraform plan
```

**예상 결과**:
```
Plan: X to add, 0 to change, 0 to destroy.
  + aws_vpc_endpoint.s3
  + module.s3_phase3...
  + aws_iam_role.fluent_bit
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
```

---

## 4. STAGE 환경

DEV와 동일 (경로만 `envs/stage`로 변경)

---

## 5. PROD 환경

DEV와 동일 + CloudTrail 활성화

```hcl
# terraform.tfvars에 추가
enable_cloudtrail = true
```

**CloudTrail 검증**:
```powershell
terraform output cloudtrail_arn
aws cloudtrail describe-trails --region ap-southeast-2
```

---

## 6. 검증 체크리스트

| 항목 | 검증 명령 | 예상 결과 |
|------|----------|----------|
| Global WAF | `terraform output -state=../mgmt/terraform.tfstate global_waf_arn` | WAF ARN |
| CloudFront | `aws cloudfront list-distributions` | 배포 목록 |
| S3 Endpoint | `aws ec2 describe-vpc-endpoints` | available |
| Fluent Bit Role | `terraform output fluent_bit_role_arn` | IAM Role ARN |
| CloudTrail | `aws cloudtrail describe-trails` | Trail 정보 |

---

## 7. 롤백

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

terraform apply -auto-approve
```

---

## 파일 경로

| 환경 | 경로 | Phase 3 리소스 |
|------|------|---------------|
| MGMT | `envs/mgmt/` | Global WAF + CloudFront |
| DEV | `envs/dev/` | VPC Endpoints, S3, Fluent Bit IRSA |
| STAGE | `envs/stage/` | VPC Endpoints, S3, Fluent Bit IRSA |
| PROD | `envs/prod/` | VPC Endpoints, S3, Fluent Bit IRSA, CloudTrail |

---

**운영 반영 전 테스트 환경에서 먼저 검증하세요.**
