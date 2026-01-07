# Phase 4 로그 분석 자동화 파이프라인 런북 v1.0

> **버전**: 1.0  
> **작성일**: 2026-01-07  
> **의존성**: Phase 3 CloudTrail 중앙 수집 완료 필요

---

## 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     로그 분석 자동화 파이프라인 (MGMT)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [CloudTrail]──┐                                                            │
│  [CloudWatch]──┼──→[S3 Audit Logs]──→[Athena]──→[Lambda]──→[Bedrock]        │
│  [EKS Logs]────┘           │              │          │          │           │
│                            │              │          │          ▼           │
│                            ▼              │          │    [AI Report]       │
│                    [EventBridge]──────────┘          │          │           │
│                     (스케줄링)                        │          ▼           │
│                      │  │  │                         │  [S3 Reports]        │
│                   일/주/월 스케줄                     │          │           │
│                      │  │  │                         └────→[Slack]          │
│                      ▼  ▼  ▼                     #kyeol-security-alerts     │
│                    [Lambda]                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. 사전 준비

### 1.1 Slack Webhook 생성

1. **Slack App 생성**: https://api.slack.com/apps
2. **Incoming Webhooks 활성화**
3. **채널 선택**: `#kyeol-security-alerts`
4. **Webhook URL 복사**

```
형식: https://hooks.slack.com/services/[WORKSPACE]/[CHANNEL]/[TOKEN]
```

### 1.2 Bedrock 모델 접근 권한

```powershell
# Bedrock 모델 사용 가능 여부 확인
aws bedrock list-foundation-models `
  --region us-east-1 `
  --query "modelSummaries[?modelId=='anthropic.claude-3-haiku-20240307-v1:0']"
```

> ⚠️ **주의**: Claude 모델 사용을 위해 AWS 콘솔에서 모델 접근 요청이 필요할 수 있습니다.

---

## 2. 실행 순서

| 순서 | 작업 | 예상 시간 |
|:----:|------|:--------:|
| 1 | Phase 3 CloudTrail 활성화 확인 | 1분 |
| 2 | Slack Webhook 생성 | 5분 |
| 3 | terraform.tfvars 설정 | 2분 |
| 4 | Terraform Apply | 5분 |
| 5 | 동작 확인 | 5분 |

---

## 3. Terraform 설정

### 3.1 terraform.tfvars 수정

```hcl
# envs/mgmt/terraform.tfvars

# Phase 3 CloudTrail (필수)
enable_cloudtrail = true

# Phase 4 로그 분석 파이프라인
enable_log_analytics = true

# Slack 알림 설정
slack_webhook_url = "https://hooks.slack.com/services/YOUR/ACTUAL/WEBHOOK"
slack_channel     = "#kyeol-security-alerts"

# 리포트 스케줄
enable_daily_report   = true   # 매일 09:00 KST
enable_weekly_report  = true   # 매주 월요일 09:00 KST
enable_monthly_report = true   # 매월 1일 09:00 KST
```

### 3.2 Terraform Apply

```powershell
cd D:\4th_Parkminwook\WORKSPACE\saleor-demo\kyeol-infra-terraform\envs\mgmt

# 변경사항 확인
terraform plan

# 적용
terraform apply -auto-approve
```

---

## 4. 리소스 확인

### 4.1 EventBridge 규칙 확인

```powershell
# 스케줄 규칙 목록
aws events list-rules --region ap-southeast-2 --name-prefix "min-kyeol"
```

예상 규칙:
- `min-kyeol-daily-report` (일간)
- `min-kyeol-weekly-report` (주간)
- `min-kyeol-monthly-report` (월간)
- `min-kyeol-security-events` (실시간)

### 4.2 Lambda 함수 확인

```powershell
# Lambda 함수 목록
aws lambda list-functions --region ap-southeast-2 `
  --query "Functions[?contains(FunctionName, 'log-analytics')]"
```

### 4.3 Athena Workgroup 확인

```powershell
# Athena Workgroup 확인
aws athena list-work-groups --region ap-southeast-2
```

---

## 5. 수동 테스트

### 5.1 리포트 생성 Lambda 수동 실행

```powershell
# 일간 리포트 수동 생성
aws lambda invoke `
  --region ap-southeast-2 `
  --function-name min-kyeol-report-generator `
  --payload '{"report_type": "daily"}' `
  --cli-binary-format raw-in-base64-out `
  response.json

# 결과 확인
cat response.json
```

### 5.2 실시간 알람 테스트

```powershell
# IAM 사용자 생성으로 보안 이벤트 트리거
aws iam create-user --user-name test-security-alert

# Slack 알림 확인 후 삭제
aws iam delete-user --user-name test-security-alert
```

---

## 6. ISMS-P 모니터링 이벤트

### 6.1 실시간 모니터링 대상 (20개)

| 카테고리 | 이벤트 | 심각도 |
|---------|-------|:------:|
| **인증/권한** | ConsoleLogin | 🔴 높음 |
| | CreateUser, DeleteUser | 🔴 높음 |
| | CreateAccessKey, DeleteAccessKey | 🔴/🟠 |
| | AttachUserPolicy, DetachUserPolicy | 🔴/🟠 |
| | AttachRolePolicy | 🔴 높음 |
| | CreateRole, DeleteRole | 🔴/🟠 |
| **네트워크** | AuthorizeSecurityGroupIngress | 🟠 중간 |
| | AuthorizeSecurityGroupEgress | 🟠 중간 |
| | CreateSecurityGroup, DeleteSecurityGroup | 🟠 중간 |
| **데이터 보호** | PutBucketPolicy, DeleteBucketPolicy | 🟡/🟠 |
| | PutBucketPublicAccessBlock | 🟡 낮음 |
| **암호화** | DisableKey, ScheduleKeyDeletion | 🔴 높음 |
| | CreateKey | 🟡 낮음 |

---

## 7. 리포트 샘플

### 7.1 일간 리포트 구조

```markdown
# KYEOL DAILY 보안 리포트

> **생성일**: 2026-01-07 00:00:00 UTC  
> **분석 기간**: 2026-01-06 ~ 2026-01-07

---

## 1. 주요 이벤트 요약
- 총 API 호출: 1,234건
- 콘솔 로그인: 5회
- 보안그룹 변경: 2건

## 2. 보안 이상 징후
- 발견된 이상 징후 없음

## 3. 통계
- 상위 이벤트: DescribeInstances (45%), ListBuckets (20%)
- 상위 사용자: admin (60%), terraform (30%)

## 4. 권장 조치사항
- 없음 (정상 운영 상태)
```

---

## 8. 비용 산출

| 서비스 | 월 비용 | 비고 |
|--------|:------:|------|
| EventBridge | ~$1 | 스케줄 규칙 4개 |
| Athena | ~$5 | 쿼리당 $5/TB |
| Bedrock (Claude Haiku) | ~$3 | 일 1회 호출 |
| Lambda | $0 | 프리티어 |
| S3 Reports | ~$0.5 | 리포트 저장 |
| **총계** | **~$10** | |

---

## 9. 트러블슈팅

### 9.1 Lambda 타임아웃

```powershell
# Lambda 타임아웃 증가 (기본 5분)
aws lambda update-function-configuration `
  --region ap-southeast-2 `
  --function-name min-kyeol-report-generator `
  --timeout 600
```

### 9.2 Athena 쿼리 실패

```powershell
# Athena 쿼리 기록 확인
aws athena list-query-executions `
  --region ap-southeast-2 `
  --work-group min-kyeol-log-analytics

# 실패한 쿼리 상세 확인
aws athena get-query-execution `
  --region ap-southeast-2 `
  --query-execution-id [QUERY_ID]
```

### 9.3 Slack 알림 실패

```powershell
# Lambda 로그 확인
aws logs tail /aws/lambda/min-kyeol-report-generator `
  --region ap-southeast-2 `
  --since 1h
```

---

## 파일 경로

| 구성 요소 | 경로 |
|----------|------|
| 모듈 | `modules/log_analytics/` |
| Lambda 코드 | `modules/log_analytics/lambda_code/` |
| MGMT 설정 | `envs/mgmt/main.tf`, `variables.tf` |
| 설정 예시 | `envs/mgmt/terraform.tfvars.example` |

---

**운영 반영 전 테스트 환경에서 먼저 검증하세요.**
