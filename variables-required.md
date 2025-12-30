# KYEOL 필수 입력값 목록

배포 전 반드시 준비해야 하는 값들입니다.

---

## 공통 (모든 환경)

| 변수 | 설명 | 위치 | 예시 |
|------|------|------|------|
| `aws_account_id` | AWS 계정 ID | terraform.tfvars | `123456789012` |
| `aws_region` | AWS 리전 | terraform.tfvars | `ap-southeast-2` |
| `hosted_zone_id` | Route53 Hosted Zone ID (msp-g1.click) | terraform.tfvars | `Z0XXXXXXXXXXXX` |

---

## ACM 인증서 (수동 발급 권장)

| 인증서 | 리전 | 용도 | 도메인 |
|--------|------|------|--------|
| ALB용 | ap-southeast-2 | Ingress HTTPS | `*.msp-g1.click` 또는 `origin-*.msp-g1.click` |
| CloudFront용 | us-east-1 | CloudFront HTTPS | `*.msp-g1.click` 또는 `dev-kyeol.msp-g1.click` 등 |

### ACM ARN 확인 방법

```powershell
# ap-southeast-2 (ALB용)
aws acm list-certificates --region ap-southeast-2

# us-east-1 (CloudFront용)
aws acm list-certificates --region us-east-1
```

---

## Terraform 환경별 필수값

### Bootstrap

| 변수 | 필수 | 설명 |
|------|:----:|------|
| `aws_account_id` | ✅ | S3 버킷 네이밍에 사용 |
| `aws_region` | ❌ | 기본값: ap-southeast-2 |

### DEV / MGMT

| 변수 | 필수 | 설명 |
|------|:----:|------|
| `aws_account_id` | ✅ | |
| `hosted_zone_id` | ✅ | ExternalDNS용 |
| backend.tf S3 버킷 | ✅ | Bootstrap output 참조 |

---

## GitOps 값 교체 필요

### Platform GitOps (IRSA)

| 파일 | 교체 대상 | 값 출처 |
|------|-----------|---------|
| `clusters/dev/values/aws-load-balancer-controller.values.yaml` | `serviceAccount.annotations.eks.amazonaws.com/role-arn` | Terraform output: `alb_controller_role_arn` |
| `clusters/dev/values/external-dns.values.yaml` | `serviceAccount.annotations.eks.amazonaws.com/role-arn` | Terraform output: `external_dns_role_arn` |

### App GitOps

| 파일 | 교체 대상 | 설명 |
|------|-----------|------|
| `apps/saleor/overlays/dev/patches/ingress-patch.yaml` | `certificate-arn` | ALB용 ACM 인증서 ARN |
| `apps/saleor/base/deployment-storefront.yaml` | 이미지 URL | ECR 리포지토리 URL |

---

## GitHub OIDC (선택, CI/CD용)

| 항목 | 설명 |
|------|------|
| GitHub OIDC Role ARN | GitHub Actions → ECR 푸시용 IAM 역할 |
| GitHub Repo URL | ArgoCD 소스 레포지토리 |

---

## 값 확인 명령어

```powershell
# AWS Account ID
aws sts get-caller-identity --query Account --output text

# Hosted Zone ID
aws route53 list-hosted-zones-by-name --dns-name msp-g1.click --query "HostedZones[0].Id" --output text

# Terraform Output (환경별)
cd kyeol-infra-terraform/envs/dev
terraform output alb_controller_role_arn
terraform output external_dns_role_arn
terraform output ecr_repository_urls
```
