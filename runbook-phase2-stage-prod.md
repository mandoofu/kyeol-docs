# Phase-2 STAGE/PROD 환경 배포 런북

> **작성일**: 2026-01-03  
> **버전**: 2.0  
> **대상 환경**: STAGE, PROD (DEV 포함)  
> **목표**: Saleor Storefront + Dashboard를 DEV/STAGE/PROD 환경에서 HTTPS로 정상 접속

---

## 목차

1. [사전 요구사항](#1-사전-요구사항)
2. [레포지토리 클론](#2-레포지토리-클론)
3. [Terraform 인프라 배포](#3-terraform-인프라-배포)
4. [Kubernetes 컨텍스트 설정](#4-kubernetes-컨텍스트-설정)
5. [Helm Addons 설치](#5-helm-addons-설치)
6. [GitHub Actions CI/CD 설정](#6-github-actions-cicd-설정)
7. [GitOps 앱 배포](#7-gitops-앱-배포)
8. [URL 접속 검증](#8-url-접속-검증)
9. [트러블슈팅](#9-트러블슈팅)
10. [운영 주의사항](#10-운영-주의사항)

---

## 1. 사전 요구사항

### 1.1. 필수 도구 설치

| 도구 | 버전 | 설치 확인 명령 |
|------|------|----------------|
| AWS CLI | v2.x | `aws --version` |
| Terraform | >= 1.5.0 | `terraform --version` |
| kubectl | >= 1.28 | `kubectl version --client` |
| Helm | >= 3.12 | `helm version` |
| Git | >= 2.40 | `git --version` |
| PowerShell | 7.x | `$PSVersionTable.PSVersion` |

### 1.2. AWS 자격 증명

```powershell
# AWS CLI 설정 확인
aws sts get-caller-identity

# 기대 출력: Account ID, ARN 표시
```

### 1.3. 필수 AWS 리소스 (사전 생성 필요)

| 리소스 | 설명 |
|--------|------|
| Route53 Hosted Zone | `msp-g1.click` (Hosted Zone ID 필요) |
| ACM 인증서 | `*.msp-g1.click` 와일드카드 인증서 (ISSUED 상태) |
| GitHub OIDC Provider | IAM에 등록된 GitHub Actions OIDC 공급자 |

---

## 2. 레포지토리 클론

```powershell
# 워크스페이스 생성
mkdir D:\WORKSPACE\saleor-demo
cd D:\WORKSPACE\saleor-demo

# 1. Terraform 인프라
git clone https://github.com/mandoofu/kyeol-infra-terraform.git

# 2. Platform GitOps (Helm Addons)
git clone https://github.com/mandoofu/kyeol-platform-gitops.git

# 3. App GitOps (Storefront/Dashboard)
git clone https://github.com/mandoofu/kyeol-app-gitops.git

# 4. Storefront 소스
git clone https://github.com/mandoofu/kyeol-storefront.git

# 5. Dashboard 소스
git clone https://github.com/mandoofu/kyeol-saleor-dashboard.git
```

---

## 3. Terraform 인프라 배포

### 3.1. Bootstrap (최초 1회)

```powershell
cd D:\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\bootstrap

# tfvars 생성
Copy-Item terraform.tfvars.example terraform.tfvars
# terraform.tfvars 편집: aws_account_id, hosted_zone_id 입력

terraform init
terraform plan
terraform apply -auto-approve
```

### 3.2. MGMT 환경

```powershell
cd D:\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\mgmt

Copy-Item terraform.tfvars.example terraform.tfvars
# terraform.tfvars 편집

terraform init
terraform plan
terraform apply -auto-approve
```

### 3.3. DEV 환경

```powershell
cd D:\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\dev

Copy-Item terraform.tfvars.example terraform.tfvars
# terraform.tfvars 편집

terraform init
terraform plan
terraform apply -auto-approve
```

### 3.4. STAGE 환경

```powershell
cd D:\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\stage

Copy-Item terraform.tfvars.example terraform.tfvars
# terraform.tfvars 편집 (cache_node_type = "cache.t3.small" 권장)

terraform init
terraform plan
terraform apply -auto-approve
```

### 3.5. PROD 환경

```powershell
cd D:\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\prod

Copy-Item terraform.tfvars.example terraform.tfvars
# terraform.tfvars 편집 (cache_node_type = "cache.r6g.large" 필수, r6g.medium 불가)

terraform init
terraform plan
terraform apply -auto-approve
```

### 3.6. Terraform Output 확인

```powershell
# 각 환경에서 실행
terraform output eks_cluster_name
terraform output alb_controller_role_arn
terraform output external_dns_role_arn

# 기대 출력: 리소스 이름/ARN
```

---

## 4. Kubernetes 컨텍스트 설정

```powershell
# MGMT
aws eks update-kubeconfig --region ap-southeast-2 --name min-kyeol-mgmt-eks --alias mgmt

# DEV
aws eks update-kubeconfig --region ap-southeast-2 --name min-kyeol-dev-eks --alias dev

# STAGE
aws eks update-kubeconfig --region ap-southeast-2 --name min-kyeol-stage-eks --alias stage

# PROD
aws eks update-kubeconfig --region ap-southeast-2 --name min-kyeol-prod-eks --alias prod

# 컨텍스트 확인
kubectl config get-contexts

# 기대 출력: dev, mgmt, stage, prod 컨텍스트 목록
```

---

## 5. Helm Addons 설치

### 5.1. Helm 레포 추가

```powershell
helm repo add eks https://aws.github.io/eks-charts
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server
helm repo update
```

### 5.2. STAGE Addons 설치

> ⚠️ **중요**: ALB Controller → ExternalDNS → Metrics Server 순서 필수

```powershell
kubectl config use-context stage
cd D:\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\stage

# 1. ALB Controller
$ALB_ROLE_ARN = terraform output -raw alb_controller_role_arn

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller `
  -n kube-system `
  --set clusterName=min-kyeol-stage-eks `
  --set serviceAccount.create=true `
  --set serviceAccount.name=aws-load-balancer-controller `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$ALB_ROLE_ARN" `
  --set replicaCount=2

# ALB Controller Ready 확인 (필수!)
kubectl -n kube-system get pods -l app.kubernetes.io/name=aws-load-balancer-controller
# 기대 출력: 2개 pods Running

# 2. ExternalDNS
$EDNS_ROLE_ARN = terraform output -raw external_dns_role_arn

helm upgrade --install external-dns external-dns/external-dns `
  -n kube-system `
  --set provider=aws `
  --set aws.region=ap-southeast-2 `
  --set domainFilters[0]=msp-g1.click `
  --set txtOwnerId=min-kyeol-stage `
  --set policy=sync `
  --set serviceAccount.create=true `
  --set serviceAccount.name=external-dns `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$EDNS_ROLE_ARN"

# 3. Metrics Server
helm upgrade --install metrics-server metrics-server/metrics-server `
  -n kube-system --set replicas=2
```

### 5.3. PROD Addons 설치

```powershell
kubectl config use-context prod
cd D:\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\prod

# 1. ALB Controller
$ALB_ROLE_ARN = terraform output -raw alb_controller_role_arn

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller `
  -n kube-system `
  --set clusterName=min-kyeol-prod-eks `
  --set serviceAccount.create=true `
  --set serviceAccount.name=aws-load-balancer-controller `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$ALB_ROLE_ARN" `
  --set replicaCount=2

# 2. ExternalDNS
$EDNS_ROLE_ARN = terraform output -raw external_dns_role_arn

helm upgrade --install external-dns external-dns/external-dns `
  -n kube-system `
  --set provider=aws `
  --set aws.region=ap-southeast-2 `
  --set domainFilters[0]=msp-g1.click `
  --set txtOwnerId=min-kyeol-prod `
  --set policy=sync `
  --set serviceAccount.create=true `
  --set serviceAccount.name=external-dns `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$EDNS_ROLE_ARN"

# 3. Metrics Server
helm upgrade --install metrics-server metrics-server/metrics-server `
  -n kube-system --set replicas=2
```

---

## 6. GitHub Actions CI/CD 설정

### 6.1. GitHub Secrets 설정

각 레포지토리에 다음 Secret 추가:

| Secret | 값 |
|--------|-----|
| `AWS_ROLE_ARN` | `arn:aws:iam::827913617839:role/github-actions-ecr-push` |

### 6.2. GitHub OIDC Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::827913617839:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": [
            "repo:mandoofu/kyeol-storefront:*",
            "repo:mandoofu/kyeol-saleor-dashboard:*"
          ]
        }
      }
    }
  ]
}
```

### 6.3. ECR 이미지 태그 규칙

| ECR Repository | 태그 규칙 |
|----------------|----------|
| `min-kyeol-storefront` | `dev-latest`, `stage-latest`, `prod-latest` |
| `min-kyeol-dashboard` | `dev-latest`, `stage-latest`, `prod-latest` |

---

## 7. GitOps 앱 배포

### 7.1. ACM 인증서 ARN 확인

```powershell
# ACM 인증서 목록 조회
aws acm list-certificates --region ap-southeast-2 --query "CertificateSummaryList[*].[DomainName,CertificateArn]" --output table

# msp-g1.click 와일드카드 인증서 ARN 복사
```

### 7.2. Ingress Patch 파일 수정

> ⚠️ **중요**: 아래 4개 파일에서 `STAGE_CERT_ID`, `PROD_CERT_ID`를 실제 ACM ARN으로 교체

**수정 대상 파일**:

| 환경 | 파일 경로 | 교체 대상 |
|------|----------|----------|
| STAGE Storefront | `apps/saleor/overlays/stage/patches/ingress-patch.yaml` | `STAGE_CERT_ID` |
| PROD Storefront | `apps/saleor/overlays/prod/patches/ingress-patch.yaml` | `PROD_CERT_ID` |
| STAGE Dashboard | `apps/saleor-dashboard/overlays/stage/patches/ingress-patch.yaml` | `STAGE_CERT_ID` |
| PROD Dashboard | `apps/saleor-dashboard/overlays/prod/patches/ingress-patch.yaml` | `PROD_CERT_ID` |

### 7.3. STAGE 배포

```powershell
kubectl config use-context stage

# Storefront
kubectl apply -k D:\WORKSPACE\saleor-demo\kyeol-app-gitops\apps\saleor\overlays\stage

# Dashboard
kubectl apply -k D:\WORKSPACE\saleor-demo\kyeol-app-gitops\apps\saleor-dashboard\overlays\stage

# 배포 확인
kubectl -n kyeol get pods,svc,ingress
```

### 7.4. PROD 배포

```powershell
kubectl config use-context prod

# Storefront
kubectl apply -k D:\WORKSPACE\saleor-demo\kyeol-app-gitops\apps\saleor\overlays\prod

# Dashboard
kubectl apply -k D:\WORKSPACE\saleor-demo\kyeol-app-gitops\apps\saleor-dashboard\overlays\prod

# 배포 확인
kubectl -n kyeol get pods,svc,ingress
```

---

## 8. URL 접속 검증

### 8.1. Ingress ALB 주소 확인 (2-3분 대기)

```powershell
# STAGE
kubectl --context stage -n kyeol get ingress -o wide

# PROD
kubectl --context prod -n kyeol get ingress -o wide

# ADDRESS 컬럼에 ALB DNS가 표시되면 정상
```

### 8.2. DNS 레코드 확인

```powershell
nslookup origin-stage-kyeol.msp-g1.click
nslookup origin-prod-kyeol.msp-g1.click
nslookup stage-dashboard-kyeol.msp-g1.click
nslookup dashboard-kyeol.msp-g1.click

# ALB 주소가 반환되면 정상
```

### 8.3. URL 접속 테스트

```powershell
curl -I https://origin-dev-kyeol.msp-g1.click
curl -I https://origin-stage-kyeol.msp-g1.click
curl -I https://origin-prod-kyeol.msp-g1.click
curl -I https://stage-dashboard-kyeol.msp-g1.click
curl -I https://dashboard-kyeol.msp-g1.click

# HTTP/2 200 또는 HTTP/2 301/302 → 정상
```

### 8.4. 환경별 URL 표

| 환경 | 앱 | URL |
|:----:|:--:|-----|
| DEV | Storefront | https://origin-dev-kyeol.msp-g1.click |
| DEV | Dashboard | https://dev-dashboard-kyeol.msp-g1.click |
| STAGE | Storefront | https://origin-stage-kyeol.msp-g1.click |
| STAGE | Dashboard | https://stage-dashboard-kyeol.msp-g1.click |
| PROD | Storefront | https://origin-prod-kyeol.msp-g1.click |
| PROD | Dashboard | https://dashboard-kyeol.msp-g1.click |

---

## 9. 트러블슈팅

### 9.1. ExternalDNS 레코드 미생성

**증상**: Route53에 A 레코드가 없음

**확인**:
```powershell
kubectl -n kube-system logs -l app.kubernetes.io/name=external-dns --tail=50
```

**원인/해결**:
- IRSA 권한 부족 → IAM Role 정책 확인
- domainFilters 불일치 → ExternalDNS values 확인
- Ingress annotation 누락 → `external-dns.alpha.kubernetes.io/hostname` 확인

### 9.2. ALB 미생성 (Ingress ADDRESS 없음)

**증상**: Ingress ADDRESS가 빈 상태, `FailedDeployModel` 이벤트 발생

**확인**:
```powershell
# Ingress 이벤트 확인
kubectl describe ingress -n kyeol

# ALB Controller 로그 확인
kubectl -n kube-system logs -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50

# AccessDenied 검색
kubectl -n kube-system logs -l app.kubernetes.io/name=aws-load-balancer-controller | Select-String "AccessDenied|error"
```

**원인/해결**:

| 원인 | 해결 방법 |
|------|----------|
| ACM ARN 오류 | 실제 ACM ARN으로 교체 |
| 서브넷 태그 누락 | `kubernetes.io/role/elb=1` 태그 확인 |
| **DescribeListenerAttributes 권한 누락** | 아래 코드 수정 참조 |

**DescribeListenerAttributes 권한 이슈 해결 (2026-01 발생)**:

> ⚠️ **중요**: AWS Load Balancer Controller v2.7+ 이후 `DescribeListenerAttributes` API 권한 필요

**증상 로그**:
```
AccessDenied: User is not authorized to perform: elasticloadbalancing:DescribeListenerAttributes
```

**해결 (코드 수정)**:

파일: `kyeol-infra-terraform/modules/eks/iam_irsa.tf`

Describe 권한 블록에 `DescribeListenerAttributes` 추가:
```hcl
"elasticloadbalancing:DescribeListeners",
"elasticloadbalancing:DescribeListenerCertificates",
"elasticloadbalancing:DescribeListenerAttributes",  # <-- 추가
"elasticloadbalancing:DescribeSSLPolicies",
```

**적용 방법**:
```powershell
# STAGE
cd kyeol-infra-terraform/envs/stage
terraform apply -auto-approve

# PROD
cd ../prod
terraform apply -auto-approve

# ALB Controller 재시작
kubectl --context stage -n kube-system rollout restart deploy/aws-load-balancer-controller
kubectl --context prod -n kube-system rollout restart deploy/aws-load-balancer-controller

# 1분 후 Ingress ADDRESS 확인
kubectl --context stage -n kyeol get ingress -o wide
kubectl --context prod -n kyeol get ingress -o wide
```

**정상 결과 예시**:
```
NAME            CLASS   HOSTS                           ADDRESS                           PORTS   AGE
kyeol-ingress   alb     origin-stage-kyeol.msp-g1.click k8s-kyeol-kyeoling-xxx.elb.com   80      5m
```

### 9.3. 502 Bad Gateway

**증상**: URL 접속 시 502 오류

**확인**:
```powershell
kubectl -n kyeol get pods
kubectl -n kyeol describe pods
```

**원인/해결**:
- Pod CrashLoopBackOff → 로그 확인, 이미지 태그 확인
- 헬스체크 실패 → Pod readinessProbe 경로 확인
- 서비스 포트 불일치 → Service port와 Pod containerPort 확인

### 9.4. Valkey r6g.medium 오류

**증상**: `InvalidParameterValue: Invalid Cache Node Type: cache.r6g.medium`

**해결**: PROD에서 `cache_node_type = "cache.r6g.large"` 사용 (r6g.medium 미지원)

---

## 10. 운영 주의사항

### 10.1. 비용 관리

| 환경 | 예상 비용 요소 |
|------|---------------|
| DEV | 단일 NAT, 소규모 노드 |
| STAGE | 단일 NAT, 중규모 노드 |
| PROD | 다중 NAT, Multi-AZ RDS, 대규모 노드 |

### 10.2. 보안 체크리스트

- [ ] `terraform.tfvars` 절대 Git 커밋 금지
- [ ] PROD `rds_deletion_protection = true` 유지
- [ ] GitHub Secrets에 민감 정보 저장
- [ ] IAM Role 최소 권한 원칙 준수

### 10.3. 배포 순서 필수

1. ALB Controller Ready 확인 후 ExternalDNS 설치
2. Ingress 배포 전 ACM ARN 확인
3. STAGE 검증 후 PROD 배포

### 10.4. 롤백 가이드

```powershell
# Git 이전 버전으로 롤백
cd D:\WORKSPACE\saleor-demo\kyeol-app-gitops
git log --oneline -5
git checkout <commit-hash> -- apps/saleor/overlays/stage/

# 재배포
kubectl apply -k apps/saleor/overlays/stage --context stage
```

---

## 완료 체크리스트

- [ ] Bootstrap Terraform 완료
- [ ] MGMT/DEV/STAGE/PROD Terraform 완료
- [ ] 모든 kubeconfig 컨텍스트 설정
- [ ] STAGE/PROD Addons 설치 완료
- [ ] GitHub Actions ECR Push 성공
- [ ] Ingress ACM ARN 실제 값 설정
- [ ] STAGE/PROD Storefront/Dashboard 배포
- [ ] 모든 URL HTTPS 접속 확인
- [ ] HPA 동작 확인

---

> **문서 끝**
