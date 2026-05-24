
# 1. 개요

기존 OpenCampus의 인프라는 온프레미스와 비탄력적 클라우드 환경이 혼재된 구조였습니다. 이로 인해 학기 시작·종료 시점에 수강 신청과 성적 확인이 몰리며 트래픽이 급격히 치솟는 문제가 반복되었습니다. 팀이 필요로 했던 것은 다음과 같습니다.

- 수요 변동에 유연하게 대응하는 탄력적 클라우드 환경
- AI 서비스 도입 및 컴플라이언스 준수
- 운영 부담을 줄여 핵심 제품 개발에 집중할 수 있는 구조

## 1.1 인프라 구성 요약

OpenCampus는 AWS Professional Services 팀과 협력하여 마이그레이션을 진행하고, 신규 고객을 위한 인프라 설정을 자동화하는 시스템을 구축하였습니다. 핵심 인프라 구성은 다음과 같습니다.

- Amazon ECS: 완전 관리형 컨테이너 오케스트레이션으로 안정적인 서비스 운영
- Amazon Aurora: 고성능·고가용성 데이터베이스 환경 구현
- AWS Lambda: 서버 관리 없이 코드 실행이 가능한 서버리스 아키텍처 적용

## 1.2 분석 범위 및 목적
본 문서는 OpenCampus AWS 아키텍처를 보안 관점에서 분석하는 것을 목적으로 합니다. 분석 대상은 다음 세 영역에 집중합니다.

- GitHub Actions 기반 CI/CD 파이프라인 구성 방식과 도입 가능한 보안 통제
- Amazon Bedrock을 활용한 AI 서비스 구조와 적용 가능한 보안 요소
- AWS Organizations OU 분리 전략, 계정 간 리소스 연결 방식, 권한 관리 설계

---

# 2. 아키텍처 분석

## 2.1 전체 구성도

![](https://velog.velcdn.com/images/hvvup/post/33c676bb-20be-4d1a-8bc0-6dc5b0608150/image.png)
   
## 2.2 구성 요소별 역할

### GitHub Actions CI/CD 파이프라인

OpenCampus의 배포 자동화는 GitHub Actions를 중심으로 구성되어 있습니다. 파이프라인의 핵심은 정적 IAM 자격증명(Access Key) 없이 AWS에 접근하는 OIDC 페더레이션 방식입니다.

동작 흐름은 다음과 같습니다.

1. 개발자가 GitHub에 코드를 푸시하거나 PR을 병합합니다.
2. GitHub Actions Runner가 GitHub의 OIDC IdP로부터 단기 JWT 토큰을 발급받습니다.
3. AWS STS가 해당 토큰을 검증하고 IAM 역할을 수임합니다.
4. 수임된 역할의 임시 자격증명으로 ECR 이미지 푸시, ECS 태스크 정의 업데이트, CloudFormation/CDK 배포 등을 수행합니다.
5. Shared Services Account의 ECR에서 이미지를 Pull하여 각 환경 계정에 배포합니다.

이 방식의 핵심 이점은 장기 자격증명이 GitHub Secrets에 저장되지 않는다는 점입니다. OIDC 연동 자체가 자격증명 노출 위험을 구조적으로 제거합니다.

배포 파이프라인의 단계별 역할 구분은 아래와 같습니다.

| 단계 | 수임 역할 | 대상 계정 | 주요 권한 |
| --- | --- | --- | --- |
| Build & Push | `github-ecr-push-role` | Shared Services | ECR:PutImage, ECR:GetAuthorizationToken |
| Deploy Dev | `github-deploy-dev-role` | Dev Account | ECS:UpdateService, CloudFormation:* |
| Deploy Staging | `github-deploy-staging-role` | Staging Account | ECS:UpdateService (승인 게이트 필요) |
| Deploy Prod | `github-deploy-prod-role` | Prod Account | ECS:UpdateService (수동 승인 필수) |

### Amazon Bedrock 호출 구조

Bedrock은 Prod Account 내에서 VPC 인터페이스 엔드포인트를 통해 프라이빗하게 호출됩니다. OpenCampus는 AWS Partner인 Ankercloud와 협력하여 AI 호출을 파라미터화하였으며, 이를 통해 각 대학 고객이 자신의 요구에 맞는 FM을 선택할 수 있도록 하였습니다.

멀티테넌트 Bedrock 호출 구조는 다음과 같습니다.

1. 대학 고객(테넌트)이 OpenCampus 대시보드에서 사용할 FM을 선택합니다.
2. OpenCampus 백엔드(ECS Task)가 테넌트 컨텍스트와 함께 Bedrock API를 호출합니다.
3. IAM 역할은 ECS Task에 부여된 Task Execution Role을 통해 제어됩니다.
4. Bedrock 호출 결과는 테넌트별로 격리되어 반환됩니다.

> Amazon ECS는 컨테이너 오케스트레이션을 담당하고, Amazon Aurora는 고가용성 데이터베이스를 제공하며, Amazon EFS는 컨테이너 간 공유 파일 스토리지로 활용됩니다. AWS Lambda는 서버리스 이벤트 처리에 사용됩니다. 이들 서비스에 대한 상세 보안 분석은 본 문서의 범위에서 제외합니다.

# 3. 보안 관점 분석
## 3.1 현재 보안 구성 현황

### OU 분리 전략과 보안 경계
AWS Organizations에서 OU를 분리하는 핵심 목적은 Blast Radius 최소화입니다. 하나의 계정이 침해되더라도 다른 OU의 리소스에는 영향이 미치지 않도록 경계를 구성합니다.

Security OU를 독립 분리한 이유는 보안 감사 로그와 탐지 도구가 Workload 계정의 침해 영향에서 벗어나야 하기 때문입니다. Log Archive 계정의 S3 버킷에는 Workload 계정에서는 삭제 권한이 없도록 SCP(Service Control Policy)로 강제합니다. 공격자가 Workload 계정을 탈취하더라도 감사 로그는 보존됩니다.

Infrastructure OU를 분리한 이유는 ECR 이미지, Transit Gateway, DNS 등 여러 Workload 계정이 공유하는 인프라를 단일 계정에서 중앙 관리하기 위함입니다. 이를 통해 이미지 빌드 및 배포 파이프라인의 신뢰 경계를 명확히 하고, Workload 계정이 직접 이미지를 빌드하지 않도록 설계합니다.

Workload OU 내 Dev/Staging/Prod를 계정 단위로 분리한 이유는 환경 간 IAM 권한 오염을 방지하기 위함입니다. 개발 환경에서 발생한 광범위한 권한 설정이 운영 환경에 전파되지 않도록 계정 수준의 경계를 둡니다.

### 계정 간 리소스 연결 방식
**1. ECR 이미지 공유**
Shared Services Account의 ECR 리포지토리에 리소스 기반 정책(Resource-based Policy)을 설정하여, Workload 계정의 ECS Task Execution Role에만 ecr:Pull 권한을 부여합니다. 이미지 빌드 권한은 GitHub Actions 역할에만 존재합니다.

**2. Transit Gateway를 통한 네트워크 연결**
Infrastructure OU의 Shared Services Account에서 Transit Gateway를 소유하고, AWS RAM(Resource Access Manager)을 통해 각 Workload 계정에 공유합니다. 이를 통해 Dev/Staging/Prod 간 VPC 라우팅을 중앙에서 제어하며, 필요하지 않은 계정 간 트래픽은 라우팅 테이블에서 차단합니다.

**3. 감사 로그 중앙화**
모든 계정의 CloudTrail은 Security OU의 Log Archive 계정 S3로 집중됩니다. Workload 계정에는 해당 버킷에 대한 s3:DeleteObject 권한이 SCP로 차단되어 있습니다.

### 계정 간 권한 관리
계정 간 권한 관리는 IAM 역할 Trust Policy와 SCP의 이중 레이어로 구성됩니다.

**GitHub Actions**가 Prod 계정에 배포하려면, Prod 계정의 github-deploy-prod-role의 신뢰 정책이 GitHub OIDC IdP(token.actions.githubusercontent.com)를 신뢰하고, Condition에서 특정 리포지토리와 브랜치(repo:opencampus/app:ref:refs/heads/main)만 허용하도록 제한합니다. 

**ECS**에서 Bedrock을 호출하기 위해서는 ECS Task Role이 bedrock:InvokeModel 권한을 가져야 하며, 이 권한은 허용된 모델 ARN에만 범위를 제한합니다. 고객이 선택한 FM 이외의 모델은 호출되지 않도록 IAM Condition으로 통제합니다.

**SCP 레벨**에서는 Workload OU 전체에 대해 허가된 AWS 리전 이외에서의 리소스 생성을 차단하고, Security OU에 대해서는 GuardDuty 비활성화 API 호출을 차단합니다.


## 3.2 식별된 취약점
### CI/CD 파이프라인 관련

**OIDC Subject Claim 미세 조정 부재**가 첫 번째 취약점입니다. OIDC 신뢰 정책의 sub 조건을 리포지토리 수준(repo:org/repo:*)으로만 설정할 경우, 해당 리포지토리의 모든 브랜치와 PR에서 역할 수임이 가능합니다. 포크된 리포지토리의 악의적인 PR이 Prod 배포 역할을 수임할 위험이 있습니다.

**Runner 환경의 비분리**도 문제가 될 수 있습니다.. GitHub-hosted Runner를 사용할 경우, 동일한 Runner 인스턴스에서 Dev 배포와 Prod 배포가 순차 실행될 수 있습니다. 이전 잡의 임시 자격증명이 환경 변수에 잔류하는 경우, 다음 잡에서 의도치 않게 접근될 위험이 있습니다.

**배포 파이프라인에서의 컨테이너 이미지 서명 검증 부재**가 세 번째 취약점입니다. ECR에 푸시된 이미지가 빌드 파이프라인에서 생성된 것인지, 외부에서 삽입된 것인지를 ECS 배포 시점에 검증하지 않으면 공급망 공격에 노출됩니다.

### Amazon Bedrock 관련

**테넌트 간 프롬프트 컨텍스트 혼재 위험**이 있습니다. 멀티테넌트 구조에서 동일한 ECS Task가 여러 대학 고객의 Bedrock 호출을 처리할 경우, 프롬프트 구성 로직의 버그로 인해 이전 테넌트의 컨텍스트가 다른 테넌트의 요청에 포함될 수 있습니다.

**Bedrock 호출량 무제한 허용**도 위험 요소입니다. 테넌트별 호출량 제한이 없을 경우, 특정 고객의 과도한 사용이 전체 서비스의 Bedrock 할당량(Quota)에 영향을 주거나 비정상적인 비용을 유발할 수 있습니다.

**VPC 엔드포인트 미사용 시 인터넷 경유 호출**이 발생할 수 있습니다. Bedrock 엔드포인트를 VPC 인터페이스 엔드포인트 없이 인터넷 게이트웨이를 통해 호출할 경우, 트래픽이 AWS 내부 네트워크를 벗어납니다.

### OU 간 경계 관련
**SCP 정책 공백**이 있을 수 있습니다. Workload OU에 s3:PutBucketPolicy를 허용하는 경우, Workload 계정의 S3 버킷을 통해 Security OU 외부로 데이터가 유출되는 우회 경로가 생길 수 있습니다.

**RAM 공유 리소스의 과도한 노출**도 주의가 필요합니다. Transit Gateway를 RAM으로 공유할 때, 연결 가능한 계정을 OU 단위로 제한하지 않으면 Organization 내 모든 계정이 해당 게이트웨이에 연결 요청을 보낼 수 있습니다.

## 3.3 개선 권고사항
### CI/CD 파이프라인 보안 강화

**OIDC Subject Claim을 브랜치 단위로 제한**
Prod 배포 역할의 신뢰 정책 Condition을 아래와 같이 설정하여, main 브랜치의 push 이벤트에서만 역할 수임이 가능하도록 합니다.

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:sub":
      "repo:opencampus/app:ref:refs/heads/main",
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  }
}
```

**환경별 Self-hosted Runner 분리**
Prod 배포는 Prod VPC 내 격리된 Runner 인스턴스에서만 실행하도록 runs-on 레이블을 환경별로 구분합니다. Runner 인스턴스의 IMDSv2를 필수로 활성화하여 메타데이터 탈취 공격을 차단합니다.

**컨테이너 이미지 서명 및 검증**
AWS Signer 또는 Cosign을 활용해 빌드 단계에서 이미지에 서명하고, ECS 태스크 정의 업데이트 전 서명 검증 단계를 파이프라인에 삽입합니다. Amazon Inspector를 ECR에 통합하여 이미지 취약점 스캔을 자동화합니다.

**배포 이력의 감사 추적**
GitHub Actions의 배포 이벤트를 Amazon EventBridge를 통해 수집하고, Security OU의 Log Archive에 보관하는 구조를 구성합니다. 이를 통해 누가, 언제, 어떤 커밋을 Prod에 배포했는지 추적할 수 있습니다.


### Amazon Bedrock 보안 강화
**VPC 인터페이스 엔드포인트 적용**
com.amazonaws.[region].bedrock-runtime 엔드포인트를 Prod VPC에 생성하고, ECS Task의 보안 그룹이 인터넷 게이트웨이가 아닌 해당 엔드포인트로만 Bedrock 트래픽을 보내도록 라우팅을 강제합니다.

**IAM 정책에서 허용 모델 ARN을 명시적으로 제한**
아래와 같이 고객이 선택 가능한 모델만 호출 가능하도록 범위를 좁힙니다.

```
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": [
    "arn:aws:bedrock:eu-central-1::foundation-model/anthropic.claude-*",
    "arn:aws:bedrock:eu-central-1::foundation-model/amazon.titan-*"
  ]
}
```

**테넌트별 호출량 통제**
AWS Service Quotas와 AWS Budgets를 조합하여 Bedrock 호출 비용을 테넌트별로 태깅하고 모니터링합니다. 애플리케이션 레이어에서는 테넌트 ID를 기반으로 호출 속도 제한(Rate Limiting)을 적용합니다.

**프롬프트 인젝션 방어**
사용자 입력을 시스템 프롬프트와 엄격히 분리하고, Bedrock Guardrails를 활성화하여 유해 콘텐츠 필터링과 PII(개인식별정보) 마스킹을 적용합니다. 모든 Bedrock 호출 입력과 출력을 CloudWatch Logs에 기록하여 이상 패턴 탐지에 활용합니다.

### OU 간 경계 및 권한 관리 강화
**SCP를 통한 최소 허용 원칙 강제**
Workload OU에는 필요한 서비스만 허용하는 Allowlist 방식의 SCP를 적용하고, 다음 API는 명시적으로 차단합니다.

- Security OU 계정 내 S3 버킷에 대한 s3:DeleteObject, s3:PutBucketPolicy
- guardduty:DisassociateFromMasterAccount, securityhub:DisableSecurityHub
- cloudtrail:DeleteTrail, cloudtrail:StopLogging
- 허가된 리전 외에서의 모든 리소스 생성

**RAM 공유 범위를 OU 단위로 제한**
Transit Gateway 공유 시 allowExternalPrincipals를 false로 설정하고, 공유 대상을 Workload OU의 ARN으로 명시합니다. Dev 계정과 Prod 계정 사이의 직접 VPC 피어링은 허용하지 않으며, Shared Services의 Transit Gateway를 반드시 경유하도록 강제합니다.

**계정 간 Assume Role 체인을 명시적으로 제한**
특정 계정의 IAM 역할이 다른 계정의 역할을 체인 방식으로 수임하는 경우, 신뢰 정책의 sts:ExternalId 조건을 활용하여 예상치 못한 역할 수임을 차단합니다.
AWS Config 규칙을 Security Tooling 계정에서 중앙 집중 관리하여, 모든 Workload 계정의 설정 준수 여부를 단일 대시보드에서 확인합니다. 특히 Bedrock 엔드포인트 구성 여부, IAM 역할의 과도한 `*` 권한 사용, 암호화되지 않은 EBS/RDS 볼륨 등을 필수 감사 항목으로 설정합니다.


---

참고 및 출처:

- [OpenCampus builds AI-native education solutions using Amazon Bedrock](https://aws.amazon.com/ko/solutions/case-studies/opencampus/)
- [Security, Guardrails, and Observability in Amazon Bedrock](https://docs.aws.amazon.com/ko_kr/bedrock/latest/userguide/security.html)
- [AWS Bedrock Security Best Practices: Building Secure Generative AI Applications](https://dev.to/brayanarrieta/aws-bedrock-security-best-practices-building-secure-generative-ai-applications-g2j)
- [Security in every stage of CI/CD pipeline](https://docs.aws.amazon.com/ko_kr/whitepapers/latest/practicing-continuous-integration-continuous-delivery/security-in-every-stage-of-cicd-pipeline.html)
- [Use IAM roles to connect GitHub Actions to actions in AWS](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/)
- [GitHub Actions + AWS Role Chaining](https://dev.to/aws-builders/github-actions-aws-role-chaining-a-security-upgrade-worth-making-3ibb)