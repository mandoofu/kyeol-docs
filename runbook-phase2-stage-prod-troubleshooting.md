# Phase-2 트러블슈팅 가이드

Phase-2 STAGE/PROD 환경 배포 시 발생한 오류와 해결 방법을 정리합니다.

---

## 0. Valkey 엔진 num_cache_nodes 제한 (신규 이슈)

### 오류 메시지

```
Error: engine "valkey" does not support num_cache_nodes > 1

  with module.valkey[0].aws_elasticache_cluster.main,
  on ..\..\modules\valkey\main.tf line 3, in resource "aws_elasticache_cluster" "main":
```

### 원인

- `aws_elasticache_cluster` 리소스에서 valkey 엔진은 `num_cache_nodes > 1`을 지원하지 않음
- valkey/redis에서 멀티 노드/HA를 구성하려면 `aws_elasticache_replication_group` 사용 필요

### 해결 방법

1. **modules/valkey/main.tf** - `aws_elasticache_cluster` → `aws_elasticache_replication_group` 전환
2. **modules/valkey/variables.tf** - `num_cache_nodes` 제거, `replicas_per_node_group` 추가
3. **envs/stage/prod main.tf** - 새 변수 사용

### 환경별 설정

| 환경 | replicas_per_node_group | automatic_failover | multi_az |
|:----:|:-----------------------:|:------------------:|:--------:|
| DEV | 0 (단일 노드) | false | false |
| STAGE | 1 | true | true |
| PROD | 1~2 | true | true |

### 수정된 파일

- `modules/valkey/main.tf` - Replication Group 리소스 사용
- `modules/valkey/variables.tf` - HA 관련 변수 추가
- `modules/valkey/outputs.tf` - Replication Group 기반 outputs
- `envs/stage/main.tf` - replicas_per_node_group=1 사용
- `envs/prod/main.tf` - replicas_per_node_group 사용

---

## 0-1. ECR RepositoryAlreadyExistsException

### 오류 메시지

```
Error: creating ECR Repository (min-kyeol-api): RepositoryAlreadyExistsException: 
The repository with name 'min-kyeol-api' already exists in the registry with id '827913617839'
```

### 원인

- ECR 레포지토리명이 환경(dev/stage/prod)과 무관하게 동일(`min-kyeol-api` 등)
- DEV에서 이미 생성된 레포지토리가 있어 STAGE/PROD에서 충돌

### 해결 방법

**envs/stage/main.tf, envs/prod/main.tf 수정**:

```hcl
# 수정 전 (충돌 발생)
name_prefix = "${var.owner_prefix}-${var.project_name}"
# 결과: min-kyeol-api

# 수정 후 (환경별 고유)
name_prefix = "${var.owner_prefix}-${var.project_name}-${var.environment}"
# 결과: min-kyeol-stage-api, min-kyeol-prod-api
```

### 재발 방지

- 환경간 공유되면 안되는 리소스는 반드시 `environment` 포함
- ECR은 환경별로 분리하거나, 공유 시 한 곳에서만 생성

---

## 0-2. Valkey CacheParameterGroupNotFound

### 오류 메시지

```
Error: creating ElastiCache Replication Group: CacheParameterGroupNotFound: 
Cache parameter group default.valkey72 is not found.
```

### 원인

- 코드에서 `parameter_group_name = "default.valkey72"` 처럼 추정된 이름을 사용
- AWS에 해당 기본 Parameter Group이 존재하지 않음
- Valkey 엔진의 기본 파라미터 그룹 네이밍이 예상과 다를 수 있음

### 해결 방법

**modules/valkey/main.tf 수정**:

```hcl
# 수정 전 (추정 네이밍)
parameter_group_name = "default.${var.engine}${replace(var.engine_version, ".", "")}"

# 수정 후 (null이면 AWS 기본값 사용, 속성 생략)
parameter_group_name = local.effective_parameter_group_name
# effective_parameter_group_name = null 이면 AWS가 적절한 기본 Parameter Group 사용
```

**modules/valkey/variables.tf 수정**:

```hcl
variable "parameter_group_name" {
  description = "사용할 Parameter Group 이름 (null이면 AWS 기본값 사용)"
  type        = string
  default     = null
}

variable "create_parameter_group" {
  description = "커스텀 Parameter Group 생성 여부"
  type        = bool
  default     = false
}
```

### 재발 방지

- AWS 기본 리소스명을 추정/하드코딩하지 않는다
- `null`로 설정하여 AWS 기본 동작에 위임하거나
- 명시적으로 커스텀 리소스를 생성하여 사용

---

## 0-3. Terraform coalesce() null 오류

### 오류 메시지

```
Error: Error in function call

  on ..\..\modules\valkey\main.tf line 10, in locals:
  10:   effective_parameter_group_name = coalesce(
  ...
Call to function "coalesce" failed: no non-null, non-empty-string arguments.
```

### 원인

- `coalesce(arg1, arg2, ...)` 함수는 모든 인자가 null이면 에러 발생
- `coalesce(null, null)` 형태가 되어 실패

### 해결 방법

**coalesce 대신 삼항 연산자 사용**:

```hcl
# 수정 전 (오류 발생)
effective_parameter_group_name = coalesce(
  var.parameter_group_name,
  var.create_parameter_group ? aws_elasticache_parameter_group.main[0].name : null
)

# 수정 후 (null 안전)
effective_parameter_group_name = (
  var.parameter_group_name != null ? var.parameter_group_name :
  (var.create_parameter_group ? aws_elasticache_parameter_group.main[0].name : null)
)
```

### 재발 방지 원칙

1. **coalesce() 사용 금지** - 모든 인자가 null일 가능성이 있으면 사용하지 않는다
2. **조건부 삼항 연산자 사용** - null 허용이 필요하면 삼항 연산자로 처리
3. **빈 tuple 인덱싱 금지** - `count`로 생성한 리소스는 조건문 안에서만 인덱싱한다

```hcl
# ❌ 잘못된 예 (count=0이면 에러)
name = aws_elasticache_parameter_group.main[0].name

# ✅ 올바른 예 (조건문 안에서만 인덱싱)
name = var.create_parameter_group ? aws_elasticache_parameter_group.main[0].name : null
```

---

## 0-4. Valkey Invalid Cache Node Type (r6g.medium)

### 오류 메시지

```
Error: creating ElastiCache Replication Group: InvalidParameterValue: 
Invalid Cache Node Type: cache.r6g.medium.
```

### 원인

- ElastiCache(Valkey/Redis)에서 **r6g 계열은 medium 크기를 지원하지 않음**
- r6g 계열은 `large` 이상부터 사용 가능
- t3, t4g 계열은 micro, small, medium 모두 지원

### 환경별 권장 노드 타입

| 환경 | 권장 노드 타입 | 비고 |
|:----:|---------------|------|
| DEV | `cache.t3.micro` | 비용 절감 |
| STAGE | `cache.t3.small` | 테스트용 |
| PROD | `cache.r6g.large` | 고성능, medium 불가 ⚠️ |

### 해결 방법

**envs/prod/terraform.tfvars 수정**:

```hcl
# ❌ 잘못된 설정
cache_node_type = "cache.r6g.medium"

# ✅ 올바른 설정
cache_node_type = "cache.r6g.large"
```

### 재발 방지

- modules/valkey/variables.tf에 validation 추가됨:
  - r6g, r6gd, r5, r4 계열의 medium은 자동 차단
  - t3, t4g 계열은 모든 크기 허용

---

## 0-5. GitHub Actions OIDC 인증 실패

### 오류 메시지

```
Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

### 원인

- IAM Role의 **Trust Policy**에 해당 GitHub 레포지토리가 등록되어 있지 않음
- 새로운 레포(`kyeol-saleor-dashboard` 등)를 추가할 때 Trust Policy 업데이트 필요

### 해결 방법

**1. Trust Policy 파일 생성** (`trust-policy.json`):

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

**2. AWS CLI로 업데이트**:

```powershell
$env:AWS_PAGER=''
aws iam update-assume-role-policy --role-name github-actions-ecr-push --policy-document file://trust-policy.json
```

**3. GitHub Actions 재실행**:
- GitHub > Actions > "Re-run all jobs" 또는 새 커밋 푸시

### 재발 방지

- 새 레포지토리 추가 시 반드시 Trust Policy에 `repo:{owner}/{repo}:*` 추가
- `trust-policy.json`은 `kyeol-infra-terraform/` 디렉토리에 보관

## 1. 발생한 오류 유형

### 1.1. Unsupported argument 오류

Terraform plan/apply 시 모듈 호출부에서 모듈이 지원하지 않는 인자를 전달할 때 발생합니다.

```
Error: Unsupported argument

  on main.tf line 55, in module "eks":
  55:   security_groups = []

An argument named "security_groups" is not expected here.
```

### 1.2. Unsupported attribute 오류

Terraform outputs에서 모듈이 제공하지 않는 output을 참조할 때 발생합니다.

```
Error: Unsupported attribute

  on outputs.tf line 63:
  value = module.rds.endpoint

An attribute named "endpoint" is not supported here.
```

---

## 2. 해결한 오류 목록

### 2.1. EKS 모듈 (envs/stage/main.tf, envs/prod/main.tf)

| 잘못된 인자 | 수정 후 |
|------------|--------|
| `security_groups = []` | 제거 |
| `enable_alb_controller = true` | `enable_alb_controller_irsa = true` |
| `enable_external_dns = true` | `enable_external_dns_irsa = true` |
| `hosted_zone_id = var.hosted_zone_id` | `external_dns_hosted_zone_id = var.hosted_zone_id` |

### 2.2. VPC 모듈 (envs/stage/main.tf, envs/prod/main.tf)

| 잘못된 인자 | 수정 후 |
|------------|--------|
| `cluster_name = local.cluster_name` | `eks_cluster_name = local.cluster_name` |

### 2.3. ECR 모듈 (envs/stage/main.tf, envs/prod/main.tf)

| 잘못된 인자 | 수정 후 |
|------------|--------|
| `environment = var.environment` | 제거 (모듈은 이 인자를 받지 않음) |
| `image_scanning_enabled = true` | `scan_on_push = true` |

### 2.4. RDS 모듈 outputs (envs/stage/outputs.tf, envs/prod/outputs.tf)

| 잘못된 output | 올바른 output |
|--------------|--------------|
| `module.rds.endpoint` | `module.rds.db_instance_endpoint` |
| `module.rds.port` | `module.rds.db_instance_port` |
| `module.rds.secret_arn` | `module.rds.db_secret_arn` |

### 2.5. Valkey 모듈 outputs (envs/stage/outputs.tf, envs/prod/outputs.tf)

| 잘못된 output | 올바른 output |
|--------------|--------------|
| `module.valkey[0].endpoint` | `module.valkey[0].cache_endpoint` |
| `module.valkey[0].port` | `module.valkey[0].cache_port` |

---

## 3. 점검 체크리스트

### 환경별 코드 작성 전 점검 순서

1. **모듈 variables.tf 확인**
   ```powershell
   # 모듈이 어떤 input을 받는지 확인
   Get-Content .\modules\eks\variables.tf | Select-String "variable"
   Get-Content .\modules\rds_postgres\variables.tf | Select-String "variable"
   Get-Content .\modules\ecr\variables.tf | Select-String "variable"
   Get-Content .\modules\valkey\variables.tf | Select-String "variable"
   Get-Content .\modules\vpc\variables.tf | Select-String "variable"
   ```

2. **모듈 outputs.tf 확인**
   ```powershell
   # 모듈이 어떤 output을 제공하는지 확인
   Get-Content .\modules\eks\outputs.tf | Select-String "output"
   Get-Content .\modules\rds_postgres\outputs.tf | Select-String "output"
   Get-Content .\modules\ecr\outputs.tf | Select-String "output"
   Get-Content .\modules\valkey\outputs.tf | Select-String "output"
   Get-Content .\modules\vpc\outputs.tf | Select-String "output"
   ```

3. **DEV 환경 참조 (정답 기준)**
   ```powershell
   # DEV가 정상 동작하므로 참조
   code .\envs\dev\main.tf
   code .\envs\dev\outputs.tf
   ```

4. **terraform validate로 정적 오류 검증**
   ```powershell
   cd .\envs\stage
   terraform init
   terraform validate
   
   cd ..\prod
   terraform init
   terraform validate
   ```

---

## 4. 모듈별 필수 인자 정리

### 4.1. modules/eks

| 변수명 | 필수 | 설명 |
|--------|:----:|------|
| `name_prefix` | ✅ | 리소스 이름 프리픽스 |
| `environment` | ✅ | 환경 이름 |
| `cluster_name` | ✅ | EKS 클러스터 이름 |
| `cluster_version` | | Kubernetes 버전 (기본: 1.29) |
| `vpc_id` | ✅ | VPC ID |
| `subnet_ids` | ✅ | 노드 서브넷 ID 목록 |
| `node_instance_types` | | 인스턴스 타입 (기본: t3.medium) |
| `node_desired_size` | | 희망 노드 수 |
| `node_min_size` | | 최소 노드 수 |
| `node_max_size` | | 최대 노드 수 |
| `enable_irsa` | | IRSA 활성화 (기본: true) |
| `enable_alb_controller_irsa` | | ALB Controller IRSA 생성 |
| `enable_external_dns_irsa` | | ExternalDNS IRSA 생성 |
| `external_dns_hosted_zone_id` | | Route53 Hosted Zone ID |
| `tags` | | 추가 태그 |

### 4.2. modules/rds_postgres 제공 outputs

| Output 이름 | 설명 |
|------------|------|
| `db_instance_id` | RDS 인스턴스 ID |
| `db_instance_arn` | RDS 인스턴스 ARN |
| `db_instance_endpoint` | RDS 엔드포인트 (host:port 형식) |
| `db_instance_address` | RDS 호스트 주소만 |
| `db_instance_port` | RDS 포트 |
| `db_name` | 데이터베이스 이름 |
| `db_secret_arn` | Secrets Manager 시크릿 ARN |

### 4.3. modules/valkey 제공 outputs

| Output 이름 | 설명 |
|------------|------|
| `cache_cluster_id` | 클러스터 ID |
| `cache_cluster_arn` | 클러스터 ARN |
| `cache_endpoint` | 엔드포인트 주소 |
| `cache_port` | 포트 번호 |

---

## 5. 해결 후 검증

```powershell
# STAGE 검증
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\stage
terraform validate
# 결과: Success! The configuration is valid.

# PROD 검증
cd d:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\prod
terraform validate
# 결과: Success! The configuration is valid.
```

---

## 6. 예방 원칙

1. **DEV를 정답 기준으로 삼는다** - DEV가 plan 통과했으면 해당 패턴을 따른다
2. **모듈 인터페이스를 먼저 확인한다** - 새 환경 코드 작성 전 modules/*/variables.tf, outputs.tf 확인
3. **terraform validate를 자주 실행한다** - 코드 작성 직후 validate로 정적 오류 조기 발견
4. **복붙 금지** - DEV 코드를 그대로 복사하지 않고, 환경별 차이를 반영하며 참조만 한다
