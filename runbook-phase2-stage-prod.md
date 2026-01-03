# KYEOL Phase-2 런북 (STAGE + PROD + Dashboard)

Phase-2에서 STAGE, PROD 환경을 활성화하고 Saleor Dashboard를 배포하기 위한 실행 순서입니다.

---

## 0. Phase-2 개요

### Phase-1과의 차이

| 항목 | Phase-1 | Phase-2 |
|------|---------|---------|
| 환경 | DEV, MGMT | DEV, STAGE, PROD |
| Storefront | DEV만 | DEV, STAGE, PROD |
| Dashboard | 없음 | DEV, STAGE, PROD |
| RDS | Single-AZ | Multi-AZ (STAGE/PROD) |
| EKS Nodes | 최소 | 단계적 증가 |
| 캐시 | 비활성 (DEV) | 활성 (STAGE/PROD) |

### Phase-2 스코프

- STAGE VPC 및 EKS 활성화
- PROD VPC 및 EKS 활성화
- Saleor Dashboard 배포 (DEV/STAGE/PROD)
- 환경별 HPA 적용

---

## 1. 사전 준비

### 1.1. Phase-1 완료 확인

```powershell
# DEV 환경 확인
aws eks describe-cluster --name min-kyeol-dev-eks --region ap-southeast-2 --query "cluster.status"

# MGMT ArgoCD 확인
kubectl --context mgmt -n argocd get pods
```

### 1.2. ACM 인증서 발급 (STAGE/PROD용)

```powershell
# STAGE용 ACM 인증서 요청
aws acm request-certificate `
  --domain-name "origin-stage-kyeol.msp-g1.click" `
  --subject-alternative-names "stage-dashboard-kyeol.msp-g1.click" `
  --validation-method DNS `
  --region ap-southeast-2

# PROD용 ACM 인증서 요청
aws acm request-certificate `
  --domain-name "origin-prod-kyeol.msp-g1.click" `
  --subject-alternative-names "dashboard-kyeol.msp-g1.click" `
  --validation-method DNS `
  --region ap-southeast-2

# 인증서 확인
aws acm list-certificates --region ap-southeast-2
```

> ⚠️ DNS 검증을 위해 Route53에 CNAME 레코드를 추가해야 합니다.
> 해당리전에 위 명령으로 생성된 ACM 목록 들어가서 Route53에 CNAME 레코드 추가 버튼으로 추가 가능

---

## 2. STAGE 환경 배포

### 2.1. Terraform 설정

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\stage

# tfvars 복사 및 편집
Copy-Item terraform.tfvars.example terraform.tfvars
# terraform.tfvars 편집: aws_account_id, hosted_zone_id 설정

# 초기화
terraform init

# 계획 확인
terraform plan

# 적용 (비대화형)
terraform apply -auto-approve
```

### 2.2. kubeconfig 연결

```powershell
aws eks update-kubeconfig --region ap-southeast-2 --name min-kyeol-stage-eks --alias stage
kubectl config use-context stage
kubectl get nodes
```

### 2.3. Addons 설치

```powershell
# IRSA Role ARN 확인
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\stage
$STAGE_ALB_ROLE_ARN = terraform output -raw alb_controller_role_arn
$STAGE_EDNS_ROLE_ARN = terraform output -raw external_dns_role_arn

# AWS Load Balancer Controller
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller `
  -n kube-system `
  --set clusterName=min-kyeol-stage-eks `
  --set serviceAccount.create=true `
  --set serviceAccount.name=aws-load-balancer-controller `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$STAGE_ALB_ROLE_ARN" `
  --set replicaCount=2

# ExternalDNS
helm upgrade --install external-dns external-dns/external-dns `
  -n kube-system `
  --set provider=aws `
  --set aws.region=ap-southeast-2 `
  --set domainFilters[0]=msp-g1.click `
  --set txtOwnerId=min-kyeol-stage `
  --set policy=sync `
  --set serviceAccount.create=true `
  --set serviceAccount.name=external-dns `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$STAGE_EDNS_ROLE_ARN" `
  --set replicaCount=2

# Metrics Server
helm upgrade --install metrics-server metrics-server/metrics-server `
  -n kube-system --set replicas=2
```

### 2.4. 검증

```powershell
kubectl get pods -n kube-system | Select-String "aws-load-balancer|external-dns|metrics-server"
```

---

## 3. PROD 환경 배포

### 3.1. Terraform 설정

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\prod

# tfvars 복사 및 편집
Copy-Item terraform.tfvars.example terraform.tfvars
# terraform.tfvars 편집

# 초기화 및 적용
terraform init
terraform plan
terraform apply -auto-approve
```

### 3.2. kubeconfig 연결

```powershell
aws eks update-kubeconfig --region ap-southeast-2 --name min-kyeol-prod-eks --alias prod
kubectl config use-context prod
kubectl get nodes
```

### 3.3. Addons 설치

```powershell
# IRSA Role ARN 확인
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\prod
$PROD_ALB_ROLE_ARN = terraform output -raw alb_controller_role_arn
$PROD_EDNS_ROLE_ARN = terraform output -raw external_dns_role_arn

# AWS Load Balancer Controller (3 replicas for HA)
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller `
  -n kube-system `
  --set clusterName=min-kyeol-prod-eks `
  --set serviceAccount.create=true `
  --set serviceAccount.name=aws-load-balancer-controller `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$PROD_ALB_ROLE_ARN" `
  --set replicaCount=3

# ExternalDNS (3 replicas for HA)
helm upgrade --install external-dns external-dns/external-dns `
  -n kube-system `
  --set provider=aws `
  --set aws.region=ap-southeast-2 `
  --set domainFilters[0]=msp-g1.click `
  --set txtOwnerId=min-kyeol-prod `
  --set policy=sync `
  --set serviceAccount.create=true `
  --set serviceAccount.name=external-dns `
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$PROD_EDNS_ROLE_ARN" `
  --set replicaCount=3

# Metrics Server
helm upgrade --install metrics-server metrics-server/metrics-server `
  -n kube-system --set replicas=3
```

### 3.4. Helm 설치 주의사항

> ⚠️ **ALB Controller → ExternalDNS 순서 필수**

```powershell
# 1. ALB Controller 먼저 설치
helm upgrade --install aws-load-balancer-controller ...

# 2. ALB Controller Ready 확인 (필수!)
kubectl -n kube-system get pods -l app.kubernetes.io/name=aws-load-balancer-controller
kubectl -n kube-system get endpoints aws-load-balancer-webhook-service

# 3. Ready 확인 후 ExternalDNS 설치
helm upgrade --install external-dns ...
```

> ⚠️ **Webhook 엔드포인트 없음 오류 발생 시**:
> ALB Controller pods가 아직 Ready 상태가 아님. 잠시 대기 후 재시도.

---

## 4. PROD 적용 체크리스트

### 4.1. STAGE→PROD 동기화 수정사항

아래 항목들은 STAGE에서 해결된 후 PROD에 동일 적용되었습니다:

| # | 수정 항목 | 파일 | 상태 |
|:-:|----------|------|:----:|
| 1 | ECR name_prefix에 environment 포함 | `envs/prod/main.tf` | ✅ |
| 2 | EKS 모듈 enable_alb_controller_irsa 사용 | `envs/prod/main.tf` | ✅ |
| 3 | EKS 모듈 external_dns_hosted_zone_id 사용 | `envs/prod/main.tf` | ✅ |
| 4 | VPC 모듈 eks_cluster_name 사용 | `envs/prod/main.tf` | ✅ |
| 5 | Valkey Replication Group 기반 | `envs/prod/main.tf` | ✅ |
| 6 | RDS outputs db_instance_* 사용 | `envs/prod/outputs.tf` | ✅ |
| 7 | Valkey outputs cache_* 사용 | `envs/prod/outputs.tf` | ✅ |

### 4.2. PROD 전용 설정 확인

| 항목 | 기대값 | 확인 방법 |
|------|--------|----------|
| RDS Multi-AZ | `true` | `terraform output rds_multi_az` |
| RDS 삭제 보호 | `true` | `terraform output rds_deletion_protection` |
| EKS 노드 수 | 3+ | `kubectl get nodes` |
| Valkey HA | replicas=1+ | `terraform output valkey_endpoint` |
| NAT Gateway | AZ별 분리 | AWS 콘솔 |

### 4.3. PROD 배포 전 검증 명령

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\prod

# 1. 코드 포맷팅
terraform fmt

# 2. 정적 검증
terraform validate

# 3. Plan 확인 (리소스 변경 사항 검토)
terraform plan -out=tfplan

# 4. Plan 내용 검토 후 적용
terraform apply tfplan
```

### 4.4. PROD 배포 후 검증

```powershell
# kubeconfig 설정
aws eks update-kubeconfig --region ap-southeast-2 --name min-kyeol-prod-eks --alias prod

# 노드 확인
kubectl --context prod get nodes

# Addons 확인
kubectl --context prod -n kube-system get pods | Select-String "aws-load-balancer|external-dns|metrics-server"

# RDS 연결 테스트 (선택)
terraform output -raw rds_endpoint
```

### 4.5. PROD 주의사항

> [!CAUTION]
> **RDS 삭제 보호**: `rds_deletion_protection = true` 설정으로 실수로 삭제 방지
> 
> **terraform destroy 전 확인**:
> ```powershell
> # RDS 삭제 보호 해제 필요 (의도적 삭제 시)
> aws rds modify-db-instance --db-instance-identifier <id> --no-deletion-protection
> ```

> [!WARNING]
> **비용 주의**: PROD는 Multi-AZ RDS, 다중 NAT Gateway, 더 많은 EKS 노드로 인해 DEV/STAGE보다 비용이 높음

### 4.6. terraform.tfvars 커밋 금지

```powershell
# 실제 값이 들어간 terraform.tfvars는 절대 커밋하지 않음
# .gitignore에 추가되어 있는지 확인
Get-Content .gitignore | Select-String "terraform.tfvars"

# terraform.tfvars.example만 유지
```

## 4. Saleor Dashboard ECR 이미지 준비

### 4.1. kyeol-saleor-dashboard 레포 생성

GitHub에서 `saleor/saleor-dashboard`를 Fork하여 `kyeol-saleor-dashboard` 레포 생성

```powershell
# Fork 후 클론
git clone https://github.com/mandoofu/kyeol-saleor-dashboard.git
cd kyeol-saleor-dashboard
```

### 4.2. Saleor Dashboard Image Build & ECR Push (GitHub Actions)

#### 디렉토리 구조

```
kyeol-saleor-dashboard/
├── .github/
│   └── workflows/
│       └── build-push-dashboard-ecr.yml  # ECR 빌드/푸시 워크플로
├── Dockerfile                             # 멀티스테이지 빌드
├── nginx/                                 # nginx 설정
│   └── default.conf
├── src/                                   # 소스 코드
├── package.json
└── pnpm-lock.yaml
```

#### Dockerfile 빌드 흐름

1. **Builder Stage**: Node.js 22-alpine 기반
   - pnpm 설치 및 의존성 설치
   - `API_URL` 환경변수로 GraphQL 엔드포인트 설정
   - `pnpm run generate:main` → `vite build`

2. **Runner Stage**: nginx:stable-alpine 기반
   - 빌드된 static 파일을 nginx로 서빙
   - SPA fallback 설정 포함

#### GitHub Actions 실행 흐름

```yaml
# .github/workflows/build-push-dashboard-ecr.yml
on: push (main), workflow_dispatch

jobs:
  build:
    matrix: [dev, stage, prod]
    steps:
      1. Checkout
      2. OIDC AWS 인증 (AWS_ROLE_ARN)
      3. ECR 로그인
      4. Docker Buildx 설정
      5. Build & Push (환경별 API_URL)
```

#### ECR 태그 규칙

| 환경 | 태그 | API_URL |
|:----:|------|---------|
| DEV | `dev-latest`, `dev-{sha}` | `origin-dev-kyeol.msp-g1.click/graphql/` |
| STAGE | `stage-latest`, `stage-{sha}` | `origin-stage-kyeol.msp-g1.click/graphql/` |
| PROD | `prod-latest`, `prod-{sha}` | `origin-prod-kyeol.msp-g1.click/graphql/` |

#### GitHub Secrets 필수 설정

| Secret | 값 예시 |
|--------|---------|
| `AWS_ROLE_ARN` | `arn:aws:iam::827913617839:role/github-actions-ecr-push` |

> GitHub Actions OIDC를 사용하므로 AWS Access Key 대신 IAM Role을 사용합니다.

#### 트러블슈팅

| 오류 | 원인 | 해결 |
|------|------|------|
| ECR 로그인 실패 | OIDC 설정 오류 | IAM Role Trust Policy에 GitHub OIDC Provider 추가 |
| 빌드 실패 (pnpm) | 의존성 캐시 | `pnpm-lock.yaml` 일관성 확인 |
| API_URL 미적용 | ARG 순서 | Dockerfile에서 ARG 선언 후 ENV 사용 |

#### 수동 빌드 (로컬 테스트)

```powershell
cd D:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-saleor-dashboard

# DEV 환경 빌드
docker build --build-arg API_URL=https://origin-dev-kyeol.msp-g1.click/graphql/ -t dashboard:dev .

# 로컬 실행
docker run -p 8080:80 dashboard:dev
```

### 4.3. GitHub Actions Workflow 파일

---

## 5. ArgoCD Application 동기화

### 5.1. Storefront 배포

```powershell
# MGMT 클러스터에서 ArgoCD Application 적용
kubectl config use-context mgmt

# STAGE Storefront
kubectl apply -f d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-app-gitops\argocd\applications\saleor-stage.yaml

# PROD Storefront
kubectl apply -f d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-app-gitops\argocd\applications\saleor-prod.yaml
```

### 5.2. Dashboard 배포

```powershell
# DEV Dashboard
kubectl apply -f d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-app-gitops\argocd\applications\dashboard-dev.yaml

# STAGE Dashboard
kubectl apply -f d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-app-gitops\argocd\applications\dashboard-stage.yaml

# PROD Dashboard
kubectl apply -f d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-app-gitops\argocd\applications\dashboard-prod.yaml
```

### 5.3. 동기화 상태 확인

```powershell
kubectl -n argocd get applications
```

---

## 6. 검증 절차

### 6.1. Storefront 검증

| 환경 | URL | 예상 결과 |
|------|-----|----------|
| DEV | https://origin-dev-kyeol.msp-g1.click | 200 OK |
| STAGE | https://origin-stage-kyeol.msp-g1.click | 200 OK |
| PROD | https://origin-prod-kyeol.msp-g1.click | 200 OK |

```powershell
curl -I https://origin-dev-kyeol.msp-g1.click
curl -I https://origin-stage-kyeol.msp-g1.click
curl -I https://origin-prod-kyeol.msp-g1.click
```

### 6.2. Dashboard 검증

| 환경 | URL | 예상 결과 |
|------|-----|----------|
| DEV | https://dev-dashboard-kyeol.msp-g1.click | Dashboard UI |
| STAGE | https://stage-dashboard-kyeol.msp-g1.click | Dashboard UI |
| PROD | https://dashboard-kyeol.msp-g1.click | Dashboard UI |

### 6.3. HPA 검증

```powershell
kubectl --context stage -n kyeol get hpa
kubectl --context prod -n kyeol get hpa
```

---

## 7. 장애 발생 시 롤백 전략

### 7.1. ArgoCD를 통한 롤백

```powershell
# 이전 버전으로 롤백 (ArgoCD UI 또는 CLI)
argocd app rollback saleor-storefront-prod
```

### 7.2. 이미지 태그 롤백

```yaml
# kustomization.yaml 수정
images:
  - name: ...
    newTag: <previous-sha>
```

### 7.3. Terraform 롤백

```powershell
# 이전 상태로 롤백
terraform apply -target=module.eks -var-file=terraform.tfvars.backup
```

---

## 8. 환경별 리소스 정책

| 리소스 | DEV | STAGE | PROD |
|--------|-----|-------|------|
| Storefront Replicas | 1-2 | 3-6 | 5-20 |
| Dashboard Replicas | 1 | 2 | 3 |
| EKS Nodes | 2 | 2-5 | 3-10 |
| RDS | Single-AZ | Multi-AZ | Multi-AZ + 삭제 보호 |
| Cache | 비활성 | 활성 (2 nodes) | 활성 (3 nodes) |

---

## 9. Phase-3 확장 포인트

### 9.1. CloudFront 연동

- us-east-1에 ACM 인증서 발급
- CloudFront Distribution 생성
- `dev-kyeol.msp-g1.click` → CloudFront → ALB

### 9.2. WAF 연동

- AWS WAF WebACL 생성
- ALB 또는 CloudFront에 연결
- OWASP Top 10 규칙 적용

### 9.3. 모니터링 강화

- CloudWatch Container Insights
- Prometheus + Grafana
- AlertManager 설정

---

## 부록 A: GitHub 레포지토리 구조

| # | 레포 이름 | 역할 |
|:-:|-----------|------|
| 1 | `kyeol-infra-terraform` | Terraform IaC |
| 2 | `kyeol-platform-gitops` | ArgoCD, Addons |
| 3 | `kyeol-app-gitops` | Storefront + Dashboard GitOps |
| 4 | `kyeol-storefront` | Storefront 앱 (Fork) |
| 5 | `kyeol-saleor-dashboard` | Dashboard 앱 (Fork) |

## 부록 B: kyeol-app-gitops 푸시

```powershell
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-app-gitops

# Git 초기화 (이미 안된 경우)
git init
git remote add origin https://github.com/mandoofu/kyeol-app-gitops.git

# 변경사항 커밋 및 푸시
git add .
git commit -m "Phase-2: Add STAGE/PROD overlays and Dashboard"
git branch -M main
git push -u origin main
```

## 부록 C: Ingress ACM ARN 교체 명령

```powershell
# STAGE ACM ARN 조회
aws acm list-certificates --region ap-southeast-2 --query "CertificateSummaryList[?DomainName=='origin-stage-kyeol.msp-g1.click'].CertificateArn" --output text

# PROD ACM ARN 조회
aws acm list-certificates --region ap-southeast-2 --query "CertificateSummaryList[?DomainName=='origin-prod-kyeol.msp-g1.click'].CertificateArn" --output text
```

파일에서 `STAGE_CERT_ID` / `PROD_CERT_ID`를 실제 ARN으로 교체
