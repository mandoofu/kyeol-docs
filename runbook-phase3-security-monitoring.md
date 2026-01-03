# Phase-3 보안 & 모니터링 고도화 런북

> **작성일**: 2026-01-03  
> **버전**: 1.0  
> **대상 환경**: MGMT, DEV, STAGE, PROD  
> **목표**: WAF 도입, 중앙 로그 수집, Lambda@Edge 캐싱 최적화

---

## 목차

1. [사전 분석: 비용 절감 관점](#1-사전-분석-비용-절감-관점)
2. [아키텍처 구성 점검 결과](#2-아키텍처-구성-점검-결과)
3. [Phase 3-1: WAF 도입](#3-phase-3-1-waf-도입)
4. [Phase 3-2: 로그 & 모니터링](#4-phase-3-2-로그--모니터링)
5. [Phase 3-3: Lambda@Edge 적용](#5-phase-3-3-lambdaedge-적용)
6. [미구현 항목 보완](#6-미구현-항목-보완)
7. [운영 주의사항](#7-운영-주의사항)

---

## 1. 사전 분석: 비용 절감 관점

### 1.1. 사용자 판단 검증

> 사용자 판단: "EKS 노드 수 조절 + RDS 중지 정도면 충분"

✅ **전문가 검증: 대부분 맞습니다. 단, 몇 가지 추가 고려 필요**

---

### 1.2. 리소스 분류표

#### ❌ 중지 가능한 리소스 (비용 절감 효과 높음)

| 리소스 | 중지 방법 | 예상 절감 | 복구 시간 |
|--------|----------|----------|----------|
| **EKS Node Group** | `min_size=0` 조정 | 💰💰💰 (70%+) | 3-5분 |
| **RDS 인스턴스** | 콘솔/CLI 중지 | 💰💰 (30-40%) | 5-10분 |
| **NAT Gateway** | 삭제 후 재생성 | 💰 (시간당 $0.045+) | 10분+ |

```powershell
# EKS 노드 축소 (DEV 환경)
aws eks update-nodegroup-config \
  --cluster-name min-kyeol-dev-eks \
  --nodegroup-name min-kyeol-dev-ng \
  --scaling-config minSize=0,desiredSize=0,maxSize=3 \
  --region ap-southeast-2

# RDS 중지 (최대 7일, 이후 자동 재시작)
aws rds stop-db-instance \
  --db-instance-identifier min-kyeol-dev-rds \
  --region ap-southeast-2
```

---

#### ⚠️ 중지 시 영향 있는 리소스

| 리소스 | 중지 시 영향 | 권장 |
|--------|-------------|------|
| **ElastiCache (Valkey)** | 세션/캐시 데이터 손실 | ❌ 중지 비권장 (스냅샷 후 삭제 가능) |
| **NAT Gateway** | EKS → 인터넷 통신 불가 (이미지 pull 실패) | ⚠️ 완전 미사용 시만 삭제 |
| **EIP** | NAT 삭제 시 IP 변경 (PG 연동 시 이슈) | ⚠️ EIP는 유지 권장 |

---

#### ⛔ 중지하면 안 되는 리소스

| 리소스 | 이유 | 비용 |
|--------|------|------|
| **VPC / Subnets** | 무료, 삭제 시 전체 재구성 필요 | $0 |
| **Route53 Hosted Zone** | 무료 (쿼리 비용만), 삭제 시 DNS 단절 | $0.50/월 |
| **Security Groups** | 무료, 삭제 시 재설정 복잡 | $0 |
| **IAM Roles/Policies** | 무료, 삭제 시 IRSA 연동 깨짐 | $0 |
| **ECR Repositories** | 이미지 비용만, 삭제 시 이미지 손실 | 저장량 비례 |
| **S3 (tfstate)** | 삭제 시 Terraform 상태 손실 | 저장량 비례 |
| **ACM 인증서** | 무료, 삭제 시 HTTPS 불가 | $0 |

---

### 1.3. 환경별 최적 비용 절감 전략

| 환경 | 평일 업무시간 외 | 주말 | 장기 미사용 |
|:----:|:---------------:|:----:|:----------:|
| **DEV** | 노드 0 + RDS 중지 | 노드 0 + RDS 중지 | 전체 Terraform destroy |
| **STAGE** | 노드 1 유지 | 노드 0 + RDS 중지 | 노드 0 + RDS 중지 |
| **PROD** | ⛔ 변경 금지 | ⛔ 변경 금지 | ⛔ 변경 금지 |

---

## 2. 아키텍처 구성 점검 결과

### 2.1. 점검 항목별 현황

| 항목 | 상태 | 근거 | 조치 필요 |
|------|:----:|------|:--------:|
| **CloudFront Distribution** | ❌ 없음 | 모듈/리소스 미존재, ACM 인증서만 global/us-east-1에 존재 | ✅ 신규 구현 |
| **NAT Gateway - Regional** | ✅ 있음 | modules/vpc/nat.tf에 정의, PROD는 Multi-AZ | - |
| **NAT Gateway - PG 전용 고정IP** | ❌ 없음 | 단일 NAT만 존재, PG 전용 분리 없음 | ⚠️ 필요 시 구현 |
| **S3 - 이미지 저장용** | ❌ 없음 | 서비스용 S3 버킷 미생성 (tfstate 전용만 존재) | ✅ 신규 구현 |
| **S3 - 로그 저장용** | ⚠️ 부분 | tfstate 로그용만 존재, 서비스 로그용 없음 | ✅ 신규 구현 |
| **S3 Gateway VPC Endpoint** | ❌ 없음 | enable_vpc_endpoints=true 변수만 존재, 실제 리소스 미구현 | ✅ 신규 구현 |
| **WAF** | ❌ 없음 | 모듈/리소스 미존재 | ✅ 신규 구현 |
| **CloudWatch Logs 중앙 수집** | ❌ 없음 | MGMT 환경에 수집 구조 없음 | ✅ 신규 구현 |

---

### 2.2. 현재 아키텍처 vs 목표 아키텍처

```
[현재 상태]
Internet → ALB → EKS Pods → RDS/Valkey
                    ↓
              NAT Gateway → Internet (outbound)

[Phase 3 목표]
Internet → CloudFront → WAF → ALB → EKS Pods → RDS/Valkey
              ↓                          ↓
        Lambda@Edge               S3 VPC Endpoint → S3
              ↓
         S3 (Static)
              ↓
         CloudWatch/S3 (Logs) ← MGMT 수집
```

---

## 3. Phase 3-1: WAF 도입

### 3.1. WAF 적용 대상

| 대상 | 연결 방식 | 우선순위 |
|------|----------|:--------:|
| ALB (각 환경) | Regional WAF | 1순위 |
| CloudFront (신규) | Global WAF (us-east-1) | 2순위 |

---

### 3.2. WAF 룰 구성 전략

#### 관리형 룰 (AWS Managed Rules)

```hcl
# modules/waf/main.tf 예시
resource "aws_wafv2_web_acl" "main" {
  name        = "${var.name_prefix}-waf"
  scope       = "REGIONAL"  # ALB용, CloudFront는 "CLOUDFRONT"
  
  default_action {
    allow {}
  }

  # 1. AWS 코어 룰셋
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
    }
  }

  # 2. SQL Injection 방어
  rule {
    name     = "AWSManagedRulesSQLiRuleSet"
    priority = 2
    # ... 생략
  }

  # 3. 알려진 악성 IP 차단
  rule {
    name     = "AWSManagedRulesAmazonIpReputationList"
    priority = 3
    # ... 생략
  }
}
```

#### 커스텀 룰 (Rate Limiting)

```hcl
# Rate Limit: 분당 1000 요청 제한
rule {
  name     = "RateLimitRule"
  priority = 10
  action { block {} }
  statement {
    rate_based_statement {
      limit              = 1000
      aggregate_key_type = "IP"
    }
  }
}
```

---

### 3.3. WAF 로그 저장 구조

```
S3 Bucket: min-kyeol-waf-logs
├── AWSLogs/
│   └── <account-id>/
│       └── WAFLogs/
│           └── <region>/
│               └── <web-acl-name>/
│                   └── YYYY/MM/DD/HH/...

→ Athena 쿼리로 분석
→ CloudWatch Logs Insights로 실시간 모니터링
```

---

## 4. Phase 3-2: 로그 & 모니터링

### 4.1. 로그 수집 대상

| 로그 유형 | 소스 | 저장 위치 | 보존 기간 |
|----------|------|----------|----------|
| WAF 로그 | WAF | S3 + CloudWatch | 90일 |
| ALB Access Log | ALB | S3 | 30일 |
| CloudFront Access Log | CloudFront | S3 | 30일 |
| EKS Control Plane 로그 | EKS | CloudWatch | 30일 |
| 애플리케이션 로그 | Pods | CloudWatch (Fluent Bit) | 14일 |

---

### 4.2. MGMT 중앙 수집 아키텍처

```
[DEV/STAGE/PROD VPC]
      ↓ (Cross-Account)
[MGMT VPC]
├── CloudWatch Logs (집중)
│   ├── /aws/waf/...
│   ├── /aws/alb/...
│   └── /aws/eks/...
├── S3 (장기 보관)
│   └── min-kyeol-central-logs/
└── OpenSearch (옵션)
    └── 로그 분석 대시보드
```

---

### 4.3. Fluent Bit DaemonSet (EKS 로그 수집)

```yaml
# kyeol-platform-gitops/common/fluent-bit/values.yaml
config:
  outputs: |
    [OUTPUT]
        Name cloudwatch_logs
        Match *
        region ap-southeast-2
        log_group_name /aws/eks/${CLUSTER_NAME}/containers
        log_stream_prefix fluentbit-
        auto_create_group true
```

---

## 5. Phase 3-3: Lambda@Edge 적용

### 5.1. 적용 목적

| 목적 | 구현 방식 |
|------|----------|
| 정적 페이지 캐싱 향상 | Origin Request 단계에서 Cache-Control 헤더 조작 |
| 서비스별 분기 | Host 헤더 기반 Origin 분기 |
| A/B 테스트 | Cookie 기반 Origin 분기 |

---

### 5.2. Lambda@Edge 설계

```javascript
// lambda/origin-request/index.js
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const host = request.headers.host[0].value;
  
  // 서비스별 분기
  if (host.startsWith('origin-')) {
    request.origin.custom.domainName = 'storefront-origin.internal';
  } else if (host.includes('dashboard')) {
    request.origin.custom.domainName = 'dashboard-origin.internal';
  }
  
  return request;
};
```

---

### 5.3. 캐시 키 전략

| 콘텐츠 유형 | 캐시 키 | TTL |
|------------|--------|-----|
| 정적 자산 (JS/CSS/이미지) | URI + Query String | 1년 |
| HTML 페이지 | URI + Host | 5분 |
| API 응답 | 캐시 안 함 | 0 |

---

### 5.4. 배포 및 롤백 전략

```powershell
# Lambda@Edge 배포 (us-east-1 필수)
aws lambda publish-version \
  --function-name min-kyeol-edge-function \
  --region us-east-1

# CloudFront 연결
aws cloudfront update-distribution \
  --id E1234567890 \
  --distribution-config file://dist-config.json

# 롤백 (이전 버전으로)
aws cloudfront update-distribution \
  --id E1234567890 \
  --distribution-config file://dist-config-rollback.json
```

---

## 6. 미구현 항목 보완

### 6.1. S3 VPC Endpoint 구현

**파일**: `modules/vpc/endpoints.tf` (신규 생성)

```hcl
# S3 Gateway Endpoint
resource "aws_vpc_endpoint" "s3" {
  count = var.enable_vpc_endpoints ? 1 : 0

  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${data.aws_region.current.name}.s3"
  vpc_endpoint_type = "Gateway"

  route_table_ids = concat(
    aws_route_table.private[*].id,
    aws_route_table.public[*].id
  )

  tags = merge(var.tags, {
    Name = "${var.name_prefix}-s3-endpoint"
  })
}

# ECR API/DKR Interface Endpoints (옵션: NAT 비용 절감)
resource "aws_vpc_endpoint" "ecr_api" {
  count = var.enable_vpc_endpoints ? 1 : 0

  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${data.aws_region.current.name}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.app_private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints[0].id]
  private_dns_enabled = true

  tags = merge(var.tags, {
    Name = "${var.name_prefix}-ecr-api-endpoint"
  })
}
```

---

### 6.2. 서비스용 S3 버킷 구현

**파일**: `modules/s3/main.tf` (신규 모듈)

```hcl
# 이미지 저장용
resource "aws_s3_bucket" "media" {
  bucket = "${var.name_prefix}-media"
  
  tags = merge(var.tags, {
    Purpose = "media-storage"
  })
}

# 로그 저장용
resource "aws_s3_bucket" "logs" {
  bucket = "${var.name_prefix}-logs"
  
  tags = merge(var.tags, {
    Purpose = "log-storage"
  })
}

# Lifecycle 정책 (로그 90일 후 삭제)
resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    id     = "expire-old-logs"
    status = "Enabled"
    expiration {
      days = 90
    }
  }
}
```

---

### 6.3. CloudFront Distribution 구현

**파일**: `modules/cloudfront/main.tf` (신규 모듈)

```hcl
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  aliases             = var.domain_aliases
  price_class         = "PriceClass_200"  # 아시아/유럽/북미

  origin {
    domain_name = var.alb_dns_name
    origin_id   = "alb-origin"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "alb-origin"
    viewer_protocol_policy = "redirect-to-https"
    
    forwarded_values {
      query_string = true
      cookies {
        forward = "all"
      }
    }
  }

  viewer_certificate {
    acm_certificate_arn      = var.acm_certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  # WAF 연결
  web_acl_id = var.waf_web_acl_arn

  tags = var.tags
}
```

---

### 6.4. 구현 우선순위

| 순서 | 항목 | 복잡도 | 비용 영향 | 보안 영향 |
|:----:|------|:------:|:--------:|:--------:|
| 1 | S3 VPC Endpoint | 낮음 | 절감 | 향상 |
| 2 | WAF (ALB) | 중간 | 증가 | 🔒 필수 |
| 3 | 중앙 로그 수집 | 중간 | 소폭 증가 | 향상 |
| 4 | 서비스용 S3 | 낮음 | 소폭 증가 | - |
| 5 | CloudFront | 높음 | 상황에 따라 | 향상 |
| 6 | Lambda@Edge | 높음 | 소폭 증가 | - |

---

## 7. 운영 주의사항

### 7.1. WAF 도입 시

- ⚠️ **Count 모드 먼저 적용** - 즉시 Block 하지 말고 로그 분석 후 조정
- ⚠️ **Rate Limit 설정 주의** - 너무 낮으면 정상 트래픽도 차단
- ✅ **룰 예외 처리** - 내부 IP, 모니터링 봇 등 화이트리스트

### 7.2. CloudFront 도입 시

- ⚠️ **캐시 무효화 비용** - 1,000건 이후 $0.005/경로
- ⚠️ **TTL 설정 주의** - 동적 콘텐츠에 긴 TTL 설정 금지
- ✅ **Origin Shield 고려** - 오리진 부하 감소 (추가 비용)

### 7.3. Lambda@Edge 도입 시

- ⛔ **VPC 연결 불가** - Lambda@Edge는 VPC 내부 리소스 접근 불가
- ⚠️ **실행 제한** - 1MB 패키지, 5초 타임아웃 (Viewer), 30초 (Origin)
- ✅ **리전 us-east-1 필수** - 배포는 us-east-1에서만 가능

---

## 완료 체크리스트

- [ ] S3 VPC Endpoint Terraform 구현
- [ ] WAF 모듈 Terraform 구현
- [ ] WAF ALB 연결 테스트
- [ ] Fluent Bit DaemonSet 설치
- [ ] CloudWatch 로그 그룹 생성
- [ ] 중앙 로그 수집 구조 검증
- [ ] CloudFront 모듈 Terraform 구현 (옵션)
- [ ] Lambda@Edge 함수 개발 (옵션)

---

> **문서 끝**
