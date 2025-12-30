# KYEOL Phase-1 런북 (DEV + MGMT)

Phase-1에서 DEV와 MGMT 환경을 배포하기 위한 실행 순서입니다.

---

## 0. 사전 준비

### 0.1. 필수 도구

| 도구 | 최소 버전 | 설치 확인 |
|------|----------|----------|
| AWS CLI | 2.x | `aws --version` |
| Terraform | 1.5.0+ | `terraform --version` |
| kubectl | 1.28+ | `kubectl version --client` |
| kustomize | 5.0+ | `kustomize version` |
| helm | 3.12+ | `helm version` |
| git | 2.x | `git --version` |
| docker | 24.x+ | `docker --version` |

### 0.2. 필수 입력값 (Placeholders)

| Placeholder | 설명 | 예시 |
|-------------|------|------|
| `ACCOUNT_ID` | AWS 계정 ID | `827913617839` |
| `HOSTED_ZONE_ID` | Route53 Hosted Zone ID | `Z0XXXXXXXXXXXX` |
| `ACM_ARN` | ALB용 ACM 인증서 ARN | `arn:aws:acm:ap-southeast-2:...` |
| `REPO_OWNER` | GitHub 사용자/조직 | `mandoofu` |
| `AWS_REGION` | AWS 리전 | `ap-southeast-2` |

### 0.3. GitHub 레포지토리 구조 (3개)

> ⚠️ 이 프로젝트는 3개의 GitHub 레포지토리로 관리됩니다.

| # | 레포지토리 이름 | 로컬 폴더 | 역할 |
|:-:|----------------|----------|------|
| 1 | `kyeol-infra-terraform` | `kyeol-infra-terraform/` | Terraform IaC (VPC, EKS, RDS, ECR) |
| 2 | `kyeol-platform-gitops` | `kyeol-platform-gitops/` | ArgoCD, 클러스터 Addons |
| 3 | `kyeol-storefront` | `kyeol-storefront/` | Saleor Storefront 앱 (Fork) |

> **참고**: `kyeol-app-gitops/`는 `kyeol-platform-gitops/`에 통합하거나 별도 4번째 레포로 분리 가능

#### 폴더 → 레포지토리 매핑

```
saleor-demo/                          ← 로컬 작업 루트 (이 폴더 자체는 Git 미관리)
├── kyeol-infra-terraform/            → github.com/REPO_OWNER/kyeol-infra-terraform
├── kyeol-platform-gitops/            → github.com/REPO_OWNER/kyeol-platform-gitops
├── kyeol-app-gitops/                 → (옵션) kyeol-platform-gitops에 병합 또는 별도 레포
├── kyeol-storefront/                 → github.com/REPO_OWNER/kyeol-storefront (Fork)
└── docs/                             → kyeol-infra-terraform 또는 별도 docs 레포
```

### 0.4. Placeholder 검사 (배포 전 필수)

```powershell
# 모든 placeholder가 실제 값으로 교체되었는지 확인
Get-ChildItem -Path "d:\4th_Parkminwook\WORKSPACE\saleor-demo" -Recurse -Include *.yaml,*.tf,*.tfvars,*.json | `
  Select-String -Pattern "ACCOUNT_ID|ARN_HERE|REPO_OWNER|YOUR_ORG" | `
  Select-Object Path, LineNumber, Line
```

---

## 1. GitHub 레포지토리 생성 및 푸시

### 1.1. kyeol-infra-terraform 레포

```powershell
# GitHub에서 레포 생성: kyeol-infra-terraform (Private 권장)

cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform

# Git 초기화 및 푸시
git init
git remote add origin https://github.com/REPO_OWNER/kyeol-infra-terraform.git
git add .
git commit -m "Initial commit: KYEOL Infrastructure Terraform"
git branch -M main
git push -u origin main
```

### 1.2. kyeol-platform-gitops 레포

```powershell
# GitHub에서 레포 생성: kyeol-platform-gitops

cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-platform-gitops

git init
git remote add origin https://github.com/REPO_OWNER/kyeol-platform-gitops.git
git add .
git commit -m "Initial commit: ArgoCD and Platform Addons"
git branch -M main
git push -u origin main
```

### 1.3. kyeol-storefront 레포 (이미 Fork됨)

> ✅ 이미 `mandoofu/kyeol-storefront`로 Fork하여 클론 완료

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-storefront

# 현재 상태 확인
git remote -v
# origin  https://github.com/mandoofu/kyeol-storefront.git (fetch)

# 변경사항 커밋 및 푸시 (GitHub Actions 워크플로 추가 후)
git add .
git commit -m "Add ECR push workflow for KYEOL deployment"
git push origin main
```

---

## 2. Bootstrap: tfstate 저장소 생성

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\bootstrap

# 변수 파일 생성
Copy-Item terraform.tfvars.example terraform.tfvars
# terraform.tfvars 편집: aws_account_id 입력

# 초기화 및 적용
terraform init
terraform plan
terraform apply -auto-approve
```

### 검증

```powershell
aws s3 ls | Select-String "min-kyeol-tfstate"
aws dynamodb list-tables --region ap-southeast-2 | Select-String "min-kyeol-tfstate-lock"
```

---

## 3. MGMT 환경 배포

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\mgmt

# backend.tf의 S3 버킷 이름을 실제 값으로 수정
Copy-Item terraform.tfvars.example terraform.tfvars

terraform init
terraform plan
terraform apply -auto-approve
```

---

## 4. DEV 환경 배포

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\dev

Copy-Item terraform.tfvars.example terraform.tfvars

terraform init
terraform plan
terraform apply -auto-approve
```

### 검증

```powershell
aws eks describe-cluster --name min-kyeol-dev-eks --region ap-southeast-2 --query "cluster.status"
aws ecr describe-repositories --region ap-southeast-2 --query "repositories[].repositoryName"
```

---

## 5. kubeconfig 연결

```powershell
aws eks update-kubeconfig --region ap-southeast-2 --name min-kyeol-mgmt-eks --alias mgmt
aws eks update-kubeconfig --region ap-southeast-2 --name min-kyeol-dev-eks --alias dev
kubectl config get-contexts
```

---

## 6. MGMT에 ArgoCD 설치

```powershell
kubectl config use-context mgmt
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-platform-gitops

kubectl apply -k argocd/bootstrap/
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
```

### ArgoCD 접속

```powershell
# 초기 비밀번호
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# LoadBalancer URL
kubectl -n argocd get svc argocd-server
```

---

## 7. DEV에 Addons 설치

```powershell
kubectl config use-context dev

# IRSA 역할 ARN 확인
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\dev
$ALB_ROLE_ARN = terraform output -raw alb_controller_role_arn
$EDNS_ROLE_ARN = terraform output -raw external_dns_role_arn

# AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller `
  -n kube-system `
  --set clusterName=min-kyeol-dev-eks `
  --set serviceAccount.create=true `
  --set serviceAccount.name=aws-load-balancer-controller `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$ALB_ROLE_ARN"

# ExternalDNS
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns
helm upgrade --install external-dns external-dns/external-dns `
  -n kube-system `
  --set provider=aws `
  --set aws.region=ap-southeast-2 `
  --set domainFilters[0]=msp-g1.click `
  --set txtOwnerId=min-kyeol-dev `
  --set policy=sync `
  --set serviceAccount.create=true `
  --set serviceAccount.name=external-dns `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$EDNS_ROLE_ARN"

# Metrics Server
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server
helm upgrade --install metrics-server metrics-server/metrics-server `
  -n kube-system --set args[0]="--kubelet-insecure-tls"
```

### 검증

```powershell
kubectl get pods -n kube-system | Select-String "aws-load-balancer|external-dns|metrics-server"
```

---

## 8. GitHub Actions → ECR CI/CD 설정

### 8.1. 생성해야 할 파일 목록

| 파일 | 위치 | 설명 |
|------|------|------|
| `build-push-ecr.yml` | `kyeol-storefront/.github/workflows/` | ECR 푸시 워크플로 **(새로 생성)** |
| `trust-policy.json` | 로컬 임시 | GitHub OIDC Trust Policy |
| `ecr-push-policy.json` | 로컬 임시 | ECR Push IAM 정책 |

### 8.2. GitHub OIDC Provider 생성 (AWS 계정당 1회)

```powershell
aws iam create-open-id-connect-provider `
  --url https://token.actions.githubusercontent.com `
  --client-id-list sts.amazonaws.com `
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# 확인
aws iam list-open-id-connect-providers
```

### 8.3. IAM Role 생성

**파일: `trust-policy.json`** (로컬에 저장)

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
          "token.actions.githubusercontent.com:sub": "repo:mandoofu/kyeol-storefront:*"
        }
      }
    }
  ]
}
```

**파일: `ecr-push-policy.json`** (로컬에 저장)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "arn:aws:ecr:ap-southeast-2:827913617839:repository/min-kyeol-*"
    }
  ]
}
```

**Role 생성 명령**:

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo

# Role 생성
aws iam create-role `
  --role-name github-actions-ecr-push `
  --assume-role-policy-document file://trust-policy.json

# 정책 연결
aws iam put-role-policy `
  --role-name github-actions-ecr-push `
  --policy-name ecr-push-policy `
  --policy-document file://ecr-push-policy.json

# Role ARN 확인
aws iam get-role --role-name github-actions-ecr-push --query "Role.Arn" --output text
```

### 8.4. GitHub Repository Secrets 설정

GitHub 웹 UI: `mandoofu/kyeol-storefront` → Settings → Secrets and variables → Actions

| Name | Value |
|------|-------|
| `AWS_ROLE_ARN` | `arn:aws:iam::827913617839:role/github-actions-ecr-push` |
| `AWS_REGION` | `ap-southeast-2` |
| `ECR_REGISTRY` | `827913617839.dkr.ecr.ap-southeast-2.amazonaws.com` |
| `ECR_REPOSITORY` | `min-kyeol-storefront` |

### 8.5. GitHub Actions Workflow 파일 생성

> ⚠️ **새로 생성해야 할 파일**

**파일: `kyeol-storefront/.github/workflows/build-push-ecr.yml`**

```yaml
name: Build and Push to ECR

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  AWS_REGION: ap-southeast-2

permissions:
  id-token: write
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          build-args: |
            NEXT_PUBLIC_SALEOR_API_URL=https://storefront1.saleor.cloud/graphql/
            NEXT_PUBLIC_STOREFRONT_URL=https://origin-dev-kyeol.msp-g1.click
            NEXT_PUBLIC_DEFAULT_CHANNEL=default-channel
          tags: |
            ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:dev-latest
            ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Summary
        run: |
          echo "### Image pushed successfully! :rocket:" >> $GITHUB_STEP_SUMMARY
          echo "- \`dev-latest\`" >> $GITHUB_STEP_SUMMARY
          echo "- \`${{ github.sha }}\`" >> $GITHUB_STEP_SUMMARY
```

### 8.6. 워크플로 파일 생성 및 푸시

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-storefront

# 워크플로 파일 생성 (위 YAML 내용을 복사)
# New-Item -ItemType File -Path ".github\workflows\build-push-ecr.yml" -Force
# 위 YAML 내용을 파일에 작성

# 커밋 및 푸시 → GitHub Actions 자동 실행
git add .github/workflows/build-push-ecr.yml
git commit -m "Add ECR push workflow"
git push origin main
```

### 8.7. 수동 첫 이미지 푸시 (GitHub Actions 전)

```powershell
# ECR 로그인
aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin 827913617839.dkr.ecr.ap-southeast-2.amazonaws.com

# 이미지 빌드
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-storefront
docker build -t min-kyeol-storefront:dev-latest `
  --build-arg NEXT_PUBLIC_SALEOR_API_URL=https://demo.saleor.io/graphql/ `
  --build-arg NEXT_PUBLIC_STOREFRONT_URL=https://origin-dev-kyeol.msp-g1.click `
  .

# 태그 및 푸시
docker tag min-kyeol-storefront:dev-latest 827913617839.dkr.ecr.ap-southeast-2.amazonaws.com/min-kyeol-storefront:dev-latest
docker push 827913617839.dkr.ecr.ap-southeast-2.amazonaws.com/min-kyeol-storefront:dev-latest
```

### 8.8. ECR 이미지 확인

```powershell
aws ecr describe-images `
  --repository-name min-kyeol-storefront `
  --region ap-southeast-2 `
  --query "imageDetails[*].{Tags:imageTags,Pushed:imagePushedAt}" `
  --output table
```

---

## 9. DEV에 Saleor 배포

### 9.1. ImagePullBackOff 방지 체크리스트

- [ ] ECR에 `min-kyeol-storefront:dev-latest` 이미지 존재
- [ ] `kustomization.yaml`의 `newName`/`newTag` 정확
- [ ] EKS Node Role에 ECR ReadOnly 권한 있음

### 9.2. Kustomization 이미지 설정 확인

**파일: `kyeol-app-gitops/apps/saleor/overlays/dev/kustomization.yaml`**

```yaml
images:
  - name: ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com/min-kyeol-storefront
    newName: 827913617839.dkr.ecr.ap-southeast-2.amazonaws.com/min-kyeol-storefront
    newTag: dev-latest
```

### 9.3. 배포 실행

```powershell
kubectl config use-context dev
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-app-gitops

# 빌드 테스트
kustomize build apps/saleor/overlays/dev | Select-String "image:"

# 적용
kubectl apply -k apps/saleor/overlays/dev/
```

---

## 10. 검증

### Pod 상태 확인

```powershell
kubectl get pods -n kyeol
kubectl describe pod -n kyeol -l app=storefront
```

### Ingress/ALB 확인

```powershell
kubectl get ingress -n kyeol
kubectl get ingress kyeol-ingress -n kyeol -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### HTTP 접근 테스트

```powershell
curl -I https://origin-dev-kyeol.msp-g1.click
```

---

## 11. 트러블슈팅

### ImagePullBackOff

```powershell
kubectl describe pod -n kyeol -l app=storefront | Select-String "Error|Failed" -Context 2,2

# 점검사항:
# 1. ECR 이미지 존재 확인 (섹션 8.8)
# 2. kustomization.yaml 이미지 태그 확인
# 3. Node IAM Role에 AmazonEC2ContainerRegistryReadOnly 정책 확인
```

### AWS LBC 오류

```powershell
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
kubectl describe ingress kyeol-ingress -n kyeol
```

---

## 12. Phase-2 계획

1. CloudFront 배포 (global/us-east-1)
2. STAGE/PROD 환경 활성화
3. ArgoCD Image Updater 연동
4. WAF 연동

---

## 부록 A: 레포지토리별 푸시 명령어 요약

```powershell
# 1. kyeol-infra-terraform
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform
git init
git remote add origin https://github.com/mandoofu/kyeol-infra-terraform.git
git add . && git commit -m "Initial commit" && git branch -M main && git push -u origin main

# 2. kyeol-platform-gitops
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-platform-gitops
git init
git remote add origin https://github.com/mandoofu/kyeol-platform-gitops.git
git add . && git commit -m "Initial commit" && git branch -M main && git push -u origin main

# 3. kyeol-storefront (이미 Fork 클론됨)
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-storefront
git add . && git commit -m "Add ECR workflow" && git push origin main
```

## 부록 B: GitHub Actions 워크플로 파일 생성 스크립트

```powershell
$workflowContent = @'
name: Build and Push to ECR

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  AWS_REGION: ap-southeast-2

permissions:
  id-token: write
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          build-args: |
            NEXT_PUBLIC_SALEOR_API_URL=https://demo.saleor.io/graphql/
            NEXT_PUBLIC_STOREFRONT_URL=https://origin-dev-kyeol.msp-g1.click
          tags: |
            ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:dev-latest
            ${{ secrets.ECR_REGISTRY }}/${{ secrets.ECR_REPOSITORY }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
'@

$workflowPath = "d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-storefront\.github\workflows\build-push-ecr.yml"
$workflowContent | Out-File -FilePath $workflowPath -Encoding utf8
Write-Host "Created: $workflowPath"
```
