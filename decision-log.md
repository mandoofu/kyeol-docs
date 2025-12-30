# KYEOL 설계 결정 로그

주요 설계 선택과 그 이유를 기록합니다.

---

## 1. 리전: ap-southeast-2 (Sydney)

**결정**: 기본 리전을 `ap-southeast-2`로 설정

**이유**:
- 한국(ap-northeast-2)이 아닌 이유: 글로벌 서비스 시나리오, 비용 다양성
- Sydney는 호주/오세아니아 서비스에 적합
- spec.md에 명시된 요구사항

---

## 2. NAT Gateway: VPC당 1개 (Regional)

**결정**: AZ별 NAT가 아닌 VPC당 단일 NAT

**이유**:
- 비용 절감: NAT Gateway는 시간당 과금
- DEV/MGMT 환경에서는 고가용성보다 비용이 우선
- 프로덕션에서는 AZ별 NAT 권장

**트레이드오프**: 단일 NAT 장애 시 전체 Private 서브넷 영향

---

## 3. ALB: Terraform 직접 생성 금지

**결정**: ALB는 AWS Load Balancer Controller가 Ingress 기반으로 자동 생성

**이유**:
- Ingress 기반 ALB는 애플리케이션과 생명주기 동기화
- Terraform ALB + 별도 Target Group 관리의 복잡성 회피
- EKS 네이티브 패턴 권장

**구현**: `alb.ingress.kubernetes.io` annotation 사용

---

## 4. DNS: ExternalDNS 자동 관리

**결정**: Route53 `origin-*` 레코드는 ExternalDNS가 자동 생성

**이유**:
- Ingress 변경 시 DNS 자동 동기화
- 수동 Route53 레코드 관리 오류 방지
- GitOps 친화적

**주의**: `txtOwnerId`를 환경별로 다르게 설정하여 충돌 방지

---

## 5. 인증서 순서: ACM 선발급

**결정**: ACM 인증서를 Ingress 적용 전에 먼저 발급

**이유**:
- HTTPS Ingress는 ACM ARN이 필수
- ALB ARN을 먼저 얻어 ACM에 적용하는 것은 불가능
- DNS 검증 완료 후 Ingress 적용

**대안**: HTTP로 먼저 배포 후 HTTPS 전환 (권장하지 않음)

---

## 6. CIDR 설계: spec.md 그대로

**결정**: spec.md의 CIDR 표를 그대로 사용

**이유**:
- 환경 간 IP 충돌 방지 (10.10.x, 10.20.x, 10.30.x, 10.40.x)
- 향후 VPC Peering/Transit Gateway 확장 가능
- 문서화된 표준 준수

---

## 7. DEV Cache: 생성 안 함

**결정**: DEV 환경에서는 Valkey/Redis 캐시를 생성하지 않음

**이유**:
- 개발 환경에서 캐시 복잡성 불필요
- 비용 절감
- spec.md 명시 사항

---

## 8. 네이밍: min-kyeol-[env]-...

**결정**: 모든 리소스에 `min-` 프리픽스 강제

**이유**:
- 팀원/프로젝트 식별 용이
- 멀티 환경 리소스 구분
- spec.md 명시 사항
- S3는 글로벌 유니크: `min-kyeol-[env]-s3-[usage]-<account>-<region>`

---

## 9. tfstate: S3 + DynamoDB

**결정**: Bootstrap에서 S3 + DynamoDB lock 테이블 생성

**이유**:
- 팀 협업을 위한 원격 상태
- DynamoDB로 동시 실행 방지
- KMS 암호화 옵션

---

## 10. IRSA 사용

**결정**: AWS LBC, ExternalDNS에 IRSA 적용

**이유**:
- Pod 레벨 IAM 권한 (최소 권한 원칙)
- Access Key 불필요
- EKS 보안 모범 사례

---

## 11. ArgoCD: MGMT 클러스터

**결정**: ArgoCD는 MGMT 클러스터에 중앙 설치

**이유**:
- 중앙 CD 통제
- DEV/STAGE/PROD 클러스터 외부에서 관리
- 환경 분리 + 중앙화 균형

---

## 12. Kustomize 오버레이 패턴

**결정**: base + overlays 구조로 환경별 분리

**이유**:
- DRY(Don't Repeat Yourself) 원칙
- 환경별 차이만 패치로 관리
- GitOps 표준 패턴
