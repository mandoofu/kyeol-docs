# Phase 3 보안 & 모니터링 런북

> **버전**: 1.5 (절차 중심)  
> **작성일**: 2026-01-04  
> **대상 환경**: DEV / STAGE / PROD

---

## 0. 현재 상태 요약 (Phase 1/2 완료)

### 이미 존재하는 것

| 구분 | 리소스 | 비고 |
|------|--------|------|
| VPC | 3개 (DEV/STAGE/PROD) | NAT Gateway, 서브넷 |
| EKS | 3개 클러스터 | Node Group, IRSA |
| RDS | 3개 PostgreSQL | STAGE/PROD Multi-AZ |
| Valkey | STAGE/PROD | ElastiCache |
| ECR | 환경별 리포지토리 | api, storefront, dashboard |
| ALB | Ingress가 생성 | Terraform 관리 X |
| ArgoCD | MGMT 클러스터 | GitOps 배포 |

### Phase 3에서 새로 생성 예정

| 구분 | 생성 리소스 |
|------|------------|
| VPC Endpoint | S3, ECR, Logs, STS |
| S3 | Media, Logs, WAF Logs, Audit |
| WAF | Regional (ALB용) |
| CloudFront | CDN (STAGE/PROD) |
| CloudTrail | 감사 로그 (PROD 권장) |
| Fluent Bit | IRSA (Terraform) + Helm (GitOps) |

---

## 1. Phase 3 생성 리소스 한눈에 보기

### 1.1. 환경별 생성 체크표

| 리소스 | 변수 | DEV | STAGE | PROD | 리전 |
|--------|------|:---:|:-----:|:----:|:----:|
| S3 VPC Endpoint | `enable_s3_endpoint` | ✅ | ✅ | ✅ | ap-southeast-2 |
| ECR VPC Endpoints | `enable_ecr_endpoints` | ❌ | ✅ | ✅ | ap-southeast-2 |
| Logs VPC Endpoint | `enable_logs_endpoint` | ❌ | ✅ | ✅ | ap-southeast-2 |
| STS VPC Endpoint | `enable_sts_endpoint` | ❌ | ❌ | ✅ | ap-southeast-2 |
| S3 Media/Logs 버킷 | `enable_phase3_s3` | ✅ | ✅ | ✅ | ap-southeast-2 |
| WAF Web ACL | `enable_waf` | ✅ | ✅ | ✅ | ap-southeast-2 |
| Fluent Bit IRSA | `enable_fluent_bit_irsa` | ✅ | ✅ | ✅ | ap-southeast-2 |
| CloudFront | `enable_cloudfront` | ❌ | ✅ | ✅ | us-east-1 |
| CloudTrail | `enable_cloudtrail` | ❌ | ❌ | ✅ | ap-southeast-2 |

### 1.2. 변수 → 리소스 매핑

| 변수 | 생성되는 리소스 |
|------|----------------|
| `enable_s3_endpoint` | aws_vpc_endpoint.s3 (Gateway) |
| `enable_ecr_endpoints` | aws_vpc_endpoint.ecr_api, ecr_dkr + SG |
| `enable_logs_endpoint` | aws_vpc_endpoint.logs |
| `enable_sts_endpoint` | aws_vpc_endpoint.sts |
| `enable_phase3_s3` | aws_s3_bucket.media, logs, waf_logs + 정책 |
| `enable_waf` | aws_wafv2_web_acl + 규칙 4개 |
| `enable_fluent_bit_irsa` | aws_iam_role, aws_iam_role_policy |
| `enable_cloudfront` | aws_cloudfront_distribution + Cache Policy |
| `enable_cloudtrail` | aws_cloudtrail + aws_s3_bucket.audit + 정책 |

---

## 2. 사전 준비 (입력값 수집)

### 2.1. ALB 정보 수집 (WAF 연결용)

```powershell
# DEV ALB 확인
kubectl --context dev -n kyeol get ingress

# ALB 상세 정보 조회
aws elbv2 describe-load-balancers \
  --region ap-southeast-2 \
  --query "LoadBalancers[*].{Name:LoadBalancerName,ARN:LoadBalancerArn,DNS:DNSName,ZoneID:CanonicalHostedZoneId}" \
  --output table
```

**예상 결과**:
```
Name    | min-kyeol-dev-...
ARN     | arn:aws:elasticloadbalancing:ap-southeast-2:ACCOUNT:loadbalancer/app/...
DNS     | min-kyeol-dev-xxx.ap-southeast-2.elb.amazonaws.com
ZoneID  | Z1GM3OXH4ZPM65
```

**수집값 기록**:
- `waf_alb_arn`: ARN 값
- `cloudfront_origin_alb_dns`: DNS 값
- `cloudfront_origin_alb_zone_id`: ZoneID 값

### 2.2. CloudFront ACM 확인 (us-east-1)

```powershell
aws acm list-certificates \
  --region us-east-1 \
  --query "CertificateSummaryList[?contains(DomainName, 'msp-g1.click')]"
```

**예상 결과**:
```json
[{"DomainName": "*.msp-g1.click", "CertificateArn": "arn:aws:acm:us-east-1:..."}]
```

### 2.3. AWS 계정 ID 확인

```powershell
aws sts get-caller-identity --query "Account" --output text
```

---

## 3. 실행 순서 (충돌 방지)

> ⚠️ **순서 필수**: DEV → STAGE → PROD 순차 적용

---

### 3.1. DEV 환경 적용

#### (a) 디렉토리 이동

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\dev
```

#### (b) terraform.tfvars 수정

파일: `terraform.tfvars`

```hcl
# Phase 3 - VPC Endpoints
enable_s3_endpoint   = true
enable_ecr_endpoints = false
enable_logs_endpoint = false

# Phase 3 - S3
enable_phase3_s3    = true
logs_retention_days = 14
enforce_tls         = true

# Phase 3 - WAF (Count 모드로 시작)
enable_waf               = true
waf_alb_arn              = "arn:aws:elasticloadbalancing:ap-southeast-2:ACCOUNT:loadbalancer/app/..."
waf_rule_action_override = "count"

# Phase 3 - Fluent Bit IRSA
enable_fluent_bit_irsa = true

# Phase 3 - CloudFront/CloudTrail (DEV에서 비활성화)
enable_cloudfront = false
enable_cloudtrail = false
```

#### (c) Terraform 실행

```powershell
terraform init
terraform plan -out=phase3.tfplan
```

**예상 plan 결과 (Phase 3 ON)**:
```
Plan: 12 to add, 0 to change, 0 to destroy.
  + aws_vpc_endpoint.s3
  + aws_s3_bucket.media
  + aws_s3_bucket.logs
  + aws_s3_bucket.waf_logs
  + aws_wafv2_web_acl.main
  + aws_iam_role.fluent_bit
  + aws_iam_role_policy.fluent_bit
  ...
```

```powershell
terraform apply phase3.tfplan
```

#### (d) 검증

```powershell
# VPC Endpoint 확인
aws ec2 describe-vpc-endpoints \
  --region ap-southeast-2 \
  --filters "Name=tag:Name,Values=min-kyeol-dev-*-endpoint" \
  --query "VpcEndpoints[*].{Name:Tags[?Key=='Name']|[0].Value,State:State}" \
  --output table

# 예상: State = available

# S3 버킷 확인
aws s3 ls | grep "min-kyeol-dev"

# 예상: min-kyeol-dev-media, min-kyeol-dev-logs, aws-waf-logs-min-kyeol-dev

# WAF 확인
aws wafv2 list-web-acls --scope REGIONAL --region ap-southeast-2

# Fluent Bit Role 확인
terraform output fluent_bit_role_arn
```

#### 롤백 (문제 발생 시)

```powershell
# terraform.tfvars에서 enable 변수를 false로 변경
enable_phase3_s3 = false
enable_waf       = false
terraform apply
```

---

### 3.2. STAGE 환경 적용

#### (a) 디렉토리 이동

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\stage
```

#### (b) terraform.tfvars 수정

```hcl
# Phase 3 - VPC Endpoints (확장)
enable_s3_endpoint   = true
enable_ecr_endpoints = true
enable_logs_endpoint = true

# Phase 3 - S3
enable_phase3_s3    = true
logs_retention_days = 30
enforce_tls         = true

# Phase 3 - WAF
enable_waf               = true
waf_alb_arn              = "arn:aws:elasticloadbalancing:..."
waf_rule_action_override = "count"

# Phase 3 - Fluent Bit IRSA
enable_fluent_bit_irsa = true

# Phase 3 - CloudFront (활성화)
enable_cloudfront             = true
cloudfront_acm_arn            = "arn:aws:acm:us-east-1:..."
cloudfront_origin_alb_dns     = "min-kyeol-stage-xxx.ap-southeast-2.elb.amazonaws.com"
cloudfront_origin_alb_zone_id = "Z1GM3OXH4ZPM65"

# Phase 3 - CloudTrail (STAGE에서는 비활성화)
enable_cloudtrail = false
```

#### (c) Terraform 실행

```powershell
terraform init
terraform plan -out=phase3.tfplan
terraform apply phase3.tfplan
```

#### (d) 검증

```powershell
# ECR Endpoint 확인
aws ec2 describe-vpc-endpoints \
  --region ap-southeast-2 \
  --filters "Name=tag:Name,Values=min-kyeol-stage-ecr*" \
  --query "VpcEndpoints[*].{Name:Tags[?Key=='Name']|[0].Value,State:State}" \
  --output table

# CloudFront 확인
aws cloudfront list-distributions \
  --query "DistributionList.Items[?contains(Comment, 'stage')].{Id:Id,Domain:DomainName,Status:Status}" \
  --output table
```

---

### 3.3. PROD 환경 적용

#### (a) 디렉토리 이동

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\prod
```

#### (b) terraform.tfvars 수정

```hcl
# Phase 3 - VPC Endpoints (전체)
enable_s3_endpoint   = true
enable_ecr_endpoints = true
enable_logs_endpoint = true
enable_sts_endpoint  = true

# Phase 3 - S3 (KMS 권장)
enable_phase3_s3          = true
logs_retention_days       = 90
enforce_tls               = true
enable_kms_encryption     = true
kms_key_arn               = "arn:aws:kms:ap-southeast-2:..."
enable_glacier_transition = true
glacier_transition_days   = 90

# Phase 3 - WAF (운영 안정화 후 Block 전환)
enable_waf               = true
waf_alb_arn              = "arn:aws:elasticloadbalancing:..."
waf_rule_action_override = "count"  # 2주 후 "none"으로 변경
waf_rate_limit           = 5000

# Phase 3 - Fluent Bit IRSA
enable_fluent_bit_irsa = true

# Phase 3 - CloudFront
enable_cloudfront             = true
cloudfront_acm_arn            = "arn:aws:acm:us-east-1:..."
cloudfront_origin_alb_dns     = "..."
cloudfront_origin_alb_zone_id = "..."

# Phase 3 - CloudTrail (계정당 1개 - PROD에서 활성화)
enable_cloudtrail             = true
enable_cloudtrail_data_events = true
enable_cloudtrail_cloudwatch  = false  # 비용 주의
enable_cloudtrail_kms         = true
cloudtrail_kms_key_arn        = "arn:aws:kms:ap-southeast-2:..."
```

#### (c) Terraform 실행

```powershell
terraform init
terraform plan -out=phase3.tfplan
terraform apply phase3.tfplan
```

#### (d) 검증

```powershell
# CloudTrail 확인
aws cloudtrail describe-trails \
  --region ap-southeast-2 \
  --query "trailList[?contains(Name, 'kyeol')].{Name:Name,S3Bucket:S3BucketName,IsLogging:IsLogging}"

# Audit 버킷 확인
terraform output audit_bucket_id
```

---

### 3.4. Fluent Bit 배포 (GitOps)

> ⚠️ **Terraform은 IRSA만 생성** - Fluent Bit 배포는 ArgoCD/Helm으로

#### (a) IRSA Role ARN 확인

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\dev
terraform output fluent_bit_role_arn
```

**예상**: `arn:aws:iam::ACCOUNT:role/min-kyeol-dev-fluent-bit-role`

#### (b) GitOps values 파일 수정

파일: `kyeol-platform-gitops/clusters/{env}/values/fluent-bit.values.yaml`

```yaml
serviceAccount:
  create: true
  name: fluent-bit
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::ACCOUNT:role/min-kyeol-dev-fluent-bit-role"

config:
  outputs: |
    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.*
        region              ap-southeast-2
        log_group_name      /aws/eks/min-kyeol-dev-eks/containers
        log_stream_prefix   fluentbit-
        auto_create_group   true
```

#### (c) ArgoCD 배포 또는 Helm 수동 설치

```powershell
# Helm 수동 설치 (ArgoCD 미사용 시)
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

helm upgrade --install fluent-bit fluent/fluent-bit \
  --namespace amazon-cloudwatch \
  --create-namespace \
  -f clusters/dev/values/fluent-bit.values.yaml \
  --kube-context dev
```

#### (d) 검증

```powershell
# Pod 상태
kubectl --context dev -n amazon-cloudwatch get pods

# CloudWatch 로그 그룹 확인
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/eks/min-kyeol-dev" \
  --region ap-southeast-2
```

**예상**: Log Group 존재, Pod Running

---

## 4. 운영 전환 절차

### 4.1. WAF Count → Block 전환

> ⚠️ **2주 이상 Count 모드 운영 후 전환**

#### Step 1: WAF 로그 분석

```powershell
# WAF 로그 확인 (S3)
aws s3 ls s3://aws-waf-logs-min-kyeol-prod/ --recursive | head
```

#### Step 2: 오탐 확인

- Block 대상 요청 확인
- IP 화이트리스트 필요 여부 판단

#### Step 3: Block 전환

```hcl
# terraform.tfvars
waf_rule_action_override = "none"  # Block 모드
```

```powershell
terraform plan -out=waf-block.tfplan
terraform apply waf-block.tfplan
```

### 4.2. CloudFront 캐시 검증

```powershell
# API 경로 캐시 비활성화 확인
curl -I https://stage.msp-g1.click/graphql

# X-Cache: Miss from cloudfront (항상)
```

> ⛔ **API/GraphQL 경로에 "Hit from cloudfront" 표시 시 즉시 롤백**

---

## 5. 검증 체크리스트

### 최종 확인표

| 항목 | 명령어 | 예상 결과 |
|------|--------|----------|
| VPC Endpoint | `aws ec2 describe-vpc-endpoints` | State: available |
| S3 버킷 | `aws s3 ls \| grep kyeol` | 버킷 3개 표시 |
| WAF | `aws wafv2 list-web-acls --scope REGIONAL` | ACL 존재 |
| WAF 연결 | `aws wafv2 list-resources-for-web-acl` | ALB ARN 표시 |
| CloudFront | `aws cloudfront list-distributions` | Status: Deployed |
| CloudTrail | `aws cloudtrail describe-trails` | IsLogging: true |
| Fluent Bit | `kubectl get pods -n amazon-cloudwatch` | Running |

---

## 6. 단계별 롤백

### 6.1. WAF 즉시 분리

```powershell
aws wafv2 disassociate-web-acl \
  --resource-arn "arn:aws:elasticloadbalancing:..." \
  --region ap-southeast-2
```

### 6.2. CloudFront 비활성화

```hcl
# terraform.tfvars
enable_cloudfront = false
```

```powershell
terraform apply
```

### 6.3. CloudTrail 비활성화

```hcl
# terraform.tfvars
enable_cloudtrail = false
```

```powershell
terraform apply
```

### 6.4. Phase 3 전체 비활성화

```hcl
# terraform.tfvars - 모든 Phase 3 변수 false
enable_s3_endpoint     = false
enable_phase3_s3       = false
enable_waf             = false
enable_fluent_bit_irsa = false
enable_cloudfront      = false
enable_cloudtrail      = false
```

```powershell
terraform apply
```

---

## 파일 경로 참조표

| 구분 | 경로 |
|------|------|
| CloudTrail 모듈 | `modules/cloudtrail/{main.tf, variables.tf, outputs.tf}` |
| S3 모듈 | `modules/s3/{main.tf, variables.tf, outputs.tf}` |
| WAF 모듈 | `modules/waf/{main.tf, variables.tf, outputs.tf}` |
| CloudFront 모듈 | `modules/cloudfront/{main.tf, variables.tf, outputs.tf}` |
| VPC Endpoints | `modules/vpc/endpoints.tf` |
| Fluent Bit IRSA | `modules/eks/fluent_bit_irsa.tf` |
| DEV 환경 | `envs/dev/{main.tf, variables.tf, outputs.tf, terraform.tfvars}` |
| STAGE 환경 | `envs/stage/{main.tf, variables.tf, outputs.tf, terraform.tfvars}` |
| PROD 환경 | `envs/prod/{main.tf, variables.tf, outputs.tf, terraform.tfvars}` |
| Fluent Bit Values | `kyeol-platform-gitops/clusters/{env}/values/fluent-bit.values.yaml` |

---

**운영 반영 전 테스트 환경에서 먼저 검증하세요.**
