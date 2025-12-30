# KYEOL Saleor 멀티환경 클라우드 아키텍처 런북 (MCP 작업명령용: 전체 정리 + 디렉토리/파일 구조 포함)
- 기준일: 2025-12-29
- 목적: **MCP 기능으로 AI에게 ‘그대로 실행’ 작업명령을 내릴 수 있도록**, 아키텍처 구현에 필요한 **리소스/순서/표준 템플릿 + 생성해야 할 디렉토리/파일을 누락 없이** 정리한 런북

> 원본 런북 요약/전체를 기반으로 확장 정리했습니다. (원본: `/mnt/data/KYEOL_Saleor_MultiEnv_Runbook.md`) fileciteturn0file0

---

## 0. 한눈에 보는 목표 (요약)

### 🎯 사용자가 원하는 핵심(고정 요구사항)
- Saleor 오픈소스 기반 이커머스
- **DEV / STAGE / PROD / MGMT** 총 4개 VPC 분리
- DEV·STAGE·PROD는 **동일한 Saleor UI** 사용
- **서브도메인으로 환경 분리**
  - `dev-kyeol.msp-g1.click`
  - `stage-kyeol.msp-g1.click`
  - `kyeol.msp-g1.click` (PROD)
- ALB는 Terraform으로 직접 생성 ❌  
  → **Ingress + AWS Load Balancer Controller로 자동 생성**
- **CloudFront 3개 배포판(DEV/STAGE/PROD)** 으로 각 환경 트래픽 제어
- **ExternalDNS**로 Route53 자동 관리
- **DEV = 단일 DB**, **STAGE/PROD = Multi-AZ DB**
- 각 VPC별 **Regional NAT Gateway (EIP 수동 지정, manual mode)**
- MGMT VPC에서 **중앙 CI/CD(GitHub Actions + ArgoCD)** 통제

➡️ 목적: **실무 MSP/DevOps 포트폴리오 수준의 “운영 가능한 멀티환경 Saleor 아키텍처”**

---

## 1. 아키텍처 개요 (논리 흐름)

### 1.1 트래픽 흐름(환경 공통 패턴)
User  
→ Route53  
→ CloudFront(환경별 1개)  
→ `origin-*.msp-g1.click`  
→ ALB (Ingress 자동 생성)  
→ EKS (환경별)  
→ Saleor 서비스

### 1.2 환경별 CloudFront 분리
| 환경 | CloudFront | Origin(도메인) |
|---|---|---|
| DEV | CF-DEV | `origin-dev-kyeol.msp-g1.click` |
| STAGE | CF-STAGE | `origin-stage-kyeol.msp-g1.click` |
| PROD | CF-PROD | `origin-prod-kyeol.msp-g1.click` |

> 외부 고객용 도메인(=서비스 도메인)은 `dev-kyeol`, `stage-kyeol`, `kyeol` 이고, **CloudFront Origin은 origin- 도메인**으로 고정합니다.

---

## 2. 환경별 리소스 사이징 (테스트/포트폴리오 기준)

### 2.1 EKS 노드(Managed Node Group)
| Env | Instance | Desired | Min | Max |
|---|---|---:|---:|---:|
| DEV | t3.medium | 2 | 1 | 2 |
| STAGE | t3.medium | 2 | 2 | 4 |
| PROD | t3.medium | 3 | 2 | 5 |
| MGMT | t3.medium | 2 | 1 | 3 |

### 2.2 RDS (PostgreSQL)
| Env | Instance | Multi-AZ |
|---|---|---|
| DEV | db.t3.small | ❌ |
| STAGE | db.t3.medium | ✅ |
| PROD | db.t3.medium | ✅ |

### 2.3 Cache (Valkey/Redis)
| Env | Instance | 구성 |
|---|---|---|
| DEV | cache.t3.micro | 단일 |
| STAGE | cache.t3.small | 단일(옵션: Replication Group) |
| PROD | cache.t3.small | 단일(옵션: Replication Group) |

> 비용/복잡도를 낮추려면 STAGE/PROD도 “단일(Non-Clustered)”로 시작하고, 추후 Replication Group으로 전환합니다.

---

## 3. NAT Gateway 정책 (VPC별 1개, EIP 수동 지정)

- **VPC별 1개 NAT Gateway**
- **Regional NAT Gateway**
- **Elastic IP 수동 지정(manual mode)**

Terraform 개념:
```hcl
resource "aws_eip" "nat" {
  domain = "vpc"
  tags = { Name = "${var.name}-nat-eip" }
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = var.public_subnet_id
  tags = { Name = "${var.name}-natgw" }
}
```

---

## 4. 인증서(ACM) 전략 (중요)

| 용도 | 리전 | 비고 |
|---|---|---|
| CloudFront 인증서 | **us-east-1** | CloudFront 전용(필수) |
| ALB(Ingress) 인증서 | **EKS 리전(예: ap-northeast-3 등)** | Ingress annotation에 ARN 연결 |

> 운영 팁: `msp-g1.click` 와일드카드 또는 `*.msp-g1.click` + 필요한 SAN을 함께 관리하면 인증서 운영이 단순해집니다.

---

## 5. 필수 EKS Add-on

### 5.1 AWS Load Balancer Controller
- Ingress 기반 **ALB 자동 생성**
- IRSA 권장(서비스어카운트에 IAM Role 연동)

### 5.2 ExternalDNS
- Ingress → Route53 레코드 자동 생성
- `origin-*.msp-g1.click` 레코드를 자동 유지

### 5.3 (권장) metrics-server / cluster-autoscaler(or Karpenter)
- 노드/파드 오토스케일링의 기반
- PROD는 최소한 HPA + metrics-server는 필수

---

## 6. Ingress 표준 템플릿 (복붙용)

> 아래는 **origin 도메인** 기준입니다. (DEV/STAGE/PROD는 hostname만 바꿔서 동일 적용)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kyeol-dev
  namespace: kyeol
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: <ACM_ARN_IN_EKS_REGION>
    external-dns.alpha.kubernetes.io/hostname: origin-dev-kyeol.msp-g1.click
spec:
  rules:
  - host: origin-dev-kyeol.msp-g1.click
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: storefront-svc
            port:
              number: 3000
```

---

## 7. CloudFront + Route53 (Terraform 개념)

- CloudFront 3개 배포판(DEV/STAGE/PROD)
- Route53 A(Alias) → CloudFront 연결
- Origin은 **ALB DNS가 아니라 `origin-*.msp-g1.click` 도메인**

> 즉, CloudFront는 “도메인 이름(origin-*)”만 보며, 해당 도메인이 가리키는 대상은 ExternalDNS가 관리합니다.

---

## 8. CI/CD & MGMT VPC 전략

### 8.1 권장 패턴(중앙 통제)
- MGMT EKS
  - ArgoCD(중앙 CD)
  - Observability(로그/메트릭/트레이싱)
- GitHub Actions
  - Build & Push → ECR
- 배포는 ArgoCD (GitOps)

### 8.2 장점
- 환경별 완전 분리(네트워크/계정/권한)
- 중앙 통제(승인/감사/릴리즈 정책)
- 보안/확장/유지보수 용이

---

## 9. 배포 순서 (운영 순서 고정)

1) Terraform: VPC / NAT / EKS / RDS / Cache / ECR / (Route53 HostedZone/Record 일부)  
2) ACM 발급 (us-east-1 + EKS 리전)  
3) EKS Add-ons 설치: ALB Controller / ExternalDNS / (metrics-server 등)  
4) ArgoCD 설치(MGMT) + App-of-Apps 구성  
5) 각 환경 앱 배포(Kustomize/Helm) → Ingress 생성 → ALB 생성  
6) CloudFront 배포판 생성(DEV/STAGE/PROD) + WAF(선택)  
7) Route53: 서비스 도메인(dev-kyeol/stage-kyeol/kyeol) → CloudFront Alias  
8) 접속/헬스체크/배포 파이프라인 검증

---

# 10. (추가) 구현에 필요한 “디렉토리/파일 구조” 완전 정리 (누락 금지)

아래는 “멀티환경 4 VPC + Terraform + GitOps + CloudFront/Route53 + ExternalDNS + Ingress(ALB)”를 **실제로 구현하기 위해 생성/유지해야 하는 디렉토리와 파일 목록**입니다.  
(레포 분리는 권장사항이며, 단일 레포로 합쳐도 되지만 MSP 운영 기준으로는 분리 권장)

## 10.1 권장 레포지토리 분리(3~5개)
- `kyeol-infra-terraform` : AWS 인프라(IaC)
- `kyeol-platform-gitops` : 클러스터 공통 애드온/플랫폼(ArgoCD App-of-Apps)
- `kyeol-app-gitops` : Saleor 앱 배포(환경별 오버레이)
- (옵션) `kyeol-app-*` : 애플리케이션 소스(backend/storefront/dashboard 등)
- (옵션) `kyeol-docs` : 런북/아키텍처/운영문서

> MCP로 “작업명령” 내릴 때도, 레포 단위로 명령을 분리하면 충돌이 급감합니다.

---

## 10.2 `kyeol-infra-terraform` (IaC) 디렉토리/파일 구조

```text
kyeol-infra-terraform/
  README.md

  modules/
    vpc/
      main.tf
      variables.tf
      outputs.tf
      versions.tf
      locals.tf
      route_tables.tf
      nat.tf
      igw.tf
      endpoints.tf              # (권장) S3/ECR/STS 등 VPC Endpoint
      security_groups.tf
    eks/
      main.tf
      variables.tf
      outputs.tf
      versions.tf
      iam_irsa.tf               # ALB Controller / ExternalDNS / ArgoCD 등 IRSA
      addons.tf                 # (옵션) EKS Addon 리소스
      nodegroups.tf
      kms.tf                    # (권장) secrets encryption
      oidc.tf
    rds_postgres/
      main.tf
      variables.tf
      outputs.tf
      parameter_group.tf
      subnet_group.tf
      kms.tf
    valkey/
      main.tf
      variables.tf
      outputs.tf
      subnet_group.tf
      parameter_group.tf
    ecr/
      main.tf
      variables.tf
      outputs.tf
    route53/
      main.tf                   # hosted zone / records (외부 DNS 정책에 따라 분리)
      variables.tf
      outputs.tf
    cloudfront/
      main.tf                   # distribution + OAC/OAI + behaviors + logging
      variables.tf
      outputs.tf
    acm/
      main.tf                   # (선호) terraform으로 인증서 발급/검증 자동화 시
      variables.tf
      outputs.tf
    iam/
      main.tf                   # (선택) 공통 IAM Role/Policy 모음
      variables.tf
      outputs.tf
    observability/
      main.tf                   # (선택) CW logs, AMP/AMG, OpenSearch 등
      variables.tf
      outputs.tf

  envs/
    bootstrap/
      README.md
      backend.tf                # tfstate S3 + DynamoDB lock
      versions.tf
      providers.tf
      variables.tf
      terraform.tfvars.example
      main.tf                   # state bucket, lock table, kms, logging bucket 등
      outputs.tf

    dev/
      README.md
      backend.tf
      versions.tf
      providers.tf
      variables.tf
      terraform.tfvars          # 실값(보관 정책에 따라 비공개)
      main.tf                   # module wiring
      outputs.tf

    stage/
      README.md
      backend.tf
      versions.tf
      providers.tf
      variables.tf
      terraform.tfvars
      main.tf
      outputs.tf

    prod/
      README.md
      backend.tf
      versions.tf
      providers.tf
      variables.tf
      terraform.tfvars
      main.tf
      outputs.tf

    mgmt/
      README.md
      backend.tf
      versions.tf
      providers.tf
      variables.tf
      terraform.tfvars
      main.tf
      outputs.tf

  global/
    us-east-1/
      README.md
      backend.tf
      versions.tf
      providers.tf              # region = us-east-1
      variables.tf
      terraform.tfvars
      main.tf                   # CloudFront ACM(필수) + (선택) CloudFront 자체
      outputs.tf

  scripts/
    tf-init.ps1
    tf-plan.ps1
    tf-apply.ps1
    tf-destroy.ps1
    kubeconfig.ps1
    kubeconfig.sh
    validate.ps1
    fmt.ps1

  .github/
    workflows/
      terraform-plan.yml
      terraform-apply.yml
      terraform-drift-detect.yml

  .gitignore
```

### IaC 레포에서 “반드시 관리되어야 하는 핵심 파일/역할”
- `envs/bootstrap/*` : **tfstate 버킷 + lock(DynamoDB) + (선택) KMS, logging bucket** 생성
- `envs/{dev,stage,prod,mgmt}/*` : 각 VPC/EKS/RDS/Valkey/ECR 등을 구성
- `global/us-east-1/*` : **CloudFront용 ACM(필수)** (CloudFront를 IaC로 만들면 여기에 배치)
- `modules/*` : 환경 공통 재사용 모듈 (MSP 운영 표준)

---

## 10.3 `kyeol-platform-gitops` (플랫폼/애드온) 디렉토리/파일 구조

```text
kyeol-platform-gitops/
  README.md

  argocd/
    bootstrap/
      install.yaml              # ArgoCD 설치(manifest 또는 helm template 산출물)
      namespace.yaml
      kustomization.yaml
    app-of-apps/
      root-app.yaml             # 환경별 앱을 호출하는 루트 Application
      projects/
        kyeol-platform-project.yaml
        kyeol-app-project.yaml

  clusters/
    dev/
      kubeconfig-ref.md         # 직접 kubeconfig 저장 금지(참조만)
      values/
        aws-load-balancer-controller.values.yaml
        external-dns.values.yaml
        metrics-server.values.yaml
        (optional) cert-manager.values.yaml
      addons/
        aws-load-balancer-controller/
          kustomization.yaml    # helm template 산출물 또는 helm chart wrapper
          release.yaml          # (선택) HelmRelease/ArgoCD Application
        external-dns/
          kustomization.yaml
          release.yaml
        metrics-server/
          kustomization.yaml
          release.yaml
        (optional) karpenter/
          kustomization.yaml
          release.yaml

    stage/
      values/
      addons/
        ... (dev와 동일 구조)

    prod/
      values/
      addons/
        ...

    mgmt/
      values/
      addons/
        argocd/                 # (선택) argocd self-manage
        observability/          # grafana/loki/prometheus 등
        ...

  common/
    namespaces/
      kube-system-extra.yaml
      kyeol-platform.yaml
    rbac/
      readonly.yaml
      cicd-deployer.yaml
    policies/
      networkpolicy-default-deny.yaml   # (선택, CNI에 따라)
      pod-security.yaml                 # PSA 정책(권장)
    secrets/
      README.md                 # SOPS/External Secrets 운영 가이드(실값 저장 금지)

  scripts/
    argocd-login.ps1
    argocd-sync.ps1
    render-helm.ps1
    render-helm.sh

  .github/
    workflows/
      validate-manifests.yml
      security-scan.yml

  .gitignore
```

### 플랫폼 레포에서 “반드시 다뤄야 하는 것”
- ALB Controller / ExternalDNS 설치
- IRSA 연동(서비스어카운트 annotation + IAM Role은 IaC에서 생성)
- 환경별 values 파일(도메인, hosted zone id, txt owner id 등) 분리

---

## 10.4 `kyeol-app-gitops` (앱 배포: Saleor) 디렉토리/파일 구조

```text
kyeol-app-gitops/
  README.md

  apps/
    saleor/
      base/
        namespace.yaml
        configmap.yaml
        deployment-api.yaml
        service-api.yaml
        deployment-dashboard.yaml
        service-dashboard.yaml
        deployment-storefront.yaml
        service-storefront.yaml
        hpa-api.yaml
        hpa-storefront.yaml
        pdb-api.yaml
        ingress.yaml             # base에서는 host 비워두거나 placeholder
        kustomization.yaml

      overlays/
        dev/
          kustomization.yaml
          patches/
            ingress-host.yaml    # origin-dev-kyeol.msp-g1.click
            replicas.yaml
            resources.yaml
            env-vars.yaml        # env-specific
          secrets/
            README.md            # SOPS/ExternalSecrets 사용 시
        stage/
          kustomization.yaml
          patches/
            ingress-host.yaml
            replicas.yaml
            resources.yaml
            env-vars.yaml
        prod/
          kustomization.yaml
          patches/
            ingress-host.yaml
            replicas.yaml
            resources.yaml
            env-vars.yaml

  argocd/
    applications/
      saleor-dev.yaml
      saleor-stage.yaml
      saleor-prod.yaml
    projects/
      kyeol-apps-project.yaml

  common/
    images/
      README.md                 # 이미지 태그 정책(semver/sha)
    policies/
      resource-quotas.yaml
      limit-ranges.yaml

  .github/
    workflows/
      kustomize-build-validate.yml
      policy-check.yml

  .gitignore
```

### 앱 GitOps에서 “반드시 포함돼야 하는 설정들”
- Ingress host를 **origin-*.msp-g1.click** 로 분리
- 리소스/레플리카를 env별로 분리(DEV/STAGE/PROD)
- Secret은 Git에 평문 저장 금지(권장: SOPS or External Secrets Operator)

---

## 10.5 (옵션) 애플리케이션 소스 레포(예: `kyeol-saleor-*`) 최소 구조
> 이미 Saleor/Nimara 등 소스 레포가 별도라면, 아래는 참고 최소 구조입니다.

```text
kyeol-saleor-api/
  Dockerfile
  docker-compose.dev.yml        # 로컬 개발용(선택)
  src/ ...
  charts/ or k8s/ (선택)
  .github/workflows/build-ecr.yml

kyeol-saleor-storefront/
  Dockerfile
  src/ ...
  .github/workflows/build-ecr.yml

kyeol-saleor-dashboard/
  Dockerfile
  src/ ...
  .github/workflows/build-ecr.yml
```

---

# 11. MCP 작업명령 관점 체크리스트 (AI가 실행할 때 “필수 입력값”)

MCP로 명령 내릴 때 아래 값들이 “변수/시크릿/파라미터”로 준비되어야 합니다.

## 11.1 공통
- AWS Account ID
- 기본 리전(EKS/RDS 리전) 예: `ap-northeast-3`
- Route53 Hosted Zone ID: `msp-g1.click`
- 도메인 6개:
  - 서비스: `dev-kyeol`, `stage-kyeol`, `kyeol`
  - 오리진: `origin-dev-kyeol`, `origin-stage-kyeol`, `origin-prod-kyeol`
- ACM ARN 2종:
  - us-east-1(CloudFront)
  - EKS 리전(ALB Ingress)
- ECR repo 목록(서비스별): `api`, `storefront`, `dashboard` 등
- GitHub OIDC Role ARN(액션 → ECR 푸시)
- ArgoCD 접근 정보(초기 admin secret or SSO)

## 11.2 환경별(Terraform tfvars)
- VPC CIDR / Subnet CIDR(ENV별)
- NAT EIP 할당 방식(수동) + public subnet id
- RDS/Valkey 사이즈/멀티AZ 옵션
- EKS 노드그룹 desired/min/max 및 instance type

---

# 12. 내가 보기엔 “추가로 더 필요한 내용” 제안(강력 권장)

아래는 지금 런북에 없지만, **운영/보안/장애대응** 관점에서 실무 MSP 수준으로 올리려면 추가하는 게 좋습니다.

1) **CloudFront 보안 강화**
- Origin 접근을 “origin-도메인 + ALB”로 두되, 가능하면 **CloudFront → ALB만 허용**(보안그룹/헤더 검증)  
- CloudFront 표준 로그(S3) + ALB access log 활성화

2) **WAF 적용 위치 결정**
- (권장) CloudFront 앞단 WAF: 봇/공격 차단, 비용/정책 중앙화
- (선택) ALB WAF도 가능하나 관리 포인트 증가

3) **Secret 관리 표준**
- SOPS + KMS (GitOps 친화) 또는 External Secrets Operator + AWS Secrets Manager
- DB 비밀번호/키/토큰을 Git 평문 저장 금지

4) **백업/복구 런북**
- RDS 자동백업/스냅샷 정책 + PITR
- (선택) Velero로 클러스터 리소스 백업
- S3 버전닝/라이프사이클/삭제보호

5) **관측성(Observability) 최소셋**
- CloudWatch Container Insights 또는 Prometheus/Grafana + Loki
- ALB Controller / ExternalDNS / ArgoCD 로그 수집
- 에러 알림(슬랙/이메일) 규칙

6) **DR/환경 복구 시나리오**
- “terraform apply + argocd sync”로 복구 가능한지 정기 점검
- Drift detection(주기적 plan) 자동화

7) **네트워크/보안 기본기**
- VPC Flow Logs(필수) + 보존 정책
- (권장) Private Subnet에서 ECR/S3 접근을 VPC Endpoint로 최적화
- Security Group 최소 권한(특히 RDS/Valkey inbound 제한)

---

# 13. 마지막으로: 지금 런북에서 내가 바로 추가해주길 원하는 것(선택지)
- (A) 각 env별 **정확한 CIDR/서브넷 표**까지 포함해서 확정(DEV/STAGE/PROD/MGMT)
- (B) ExternalDNS values(HostedZone, TXT registry, owner id) “복붙 가능한 values.yaml” 완성본
- (C) CloudFront Distribution Terraform “완성 템플릿” (OAC/Cache Policy/Origin Request Policy 포함)
- (D) ArgoCD App-of-Apps “완성 템플릿” (플랫폼+앱 동시에)

원하면 위 (A)~(D)를 **그대로 적용 가능한 코드 형태로** 이번 디렉토리 구조에 맞춰서 생성해줄게요.

---

⚠️ **운영 반영 전 테스트 환경에서 먼저 검증하세요.**
