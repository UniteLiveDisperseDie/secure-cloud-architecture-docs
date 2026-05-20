from https://aws.amazon.com/ko/solutions/case-studies/brandi/

![아키텍처 다이어그램](https://velog.velcdn.com/images/alstnalstn2/post/b83b2b01-be35-4962-8a81-e9342de14bba/image.png)

---

# 1. 개요

## 1.1 서비스 소개

브랜디는 여성·남성 패션을 아우르는 국내 커머스 플랫폼입니다. 세일, 신상 드롭, 인기 급상승 시즌처럼 특정 시점에 트래픽이 폭발적으로 몰리는 특성을 가지고 있습니다. 이러한 패턴은 일반적인 B2B SaaS나 기업 내부 시스템과 달리, 순간 피크를 버텨야 하는 아키텍처 설계가 핵심 과제가 됩니다.

## 1.2 아키텍처 구성 요약

브랜디의 AWS 아키텍처는 크게 다섯 가지 레이어로 나눌 수 있습니다.

- **트래픽 수신**: 서비스별로 분리된 ALB(Application Load Balancer) 구성
- **데이터 저장**: Aurora MySQL Multi-AZ, S3, DynamoDB
- **비동기 처리**: SNS, SQS, Lambda를 활용한 이벤트 기반 아키텍처
- **개인화 파이프라인**: Glue ETL → Amazon Personalize
- **배포 자동화**: CodeCommit → CodeBuild → CloudFormation 기반 CI/CD

---

# 2. 아키텍처 분석

## 2.1 핵심 구성 요소와 역할

### ALB 서비스별 분리 (BRANDI / HIVER / ADMIN)

브랜디 아키텍처에서 눈에 띄는 설계 결정 중 하나는 ALB를 하나로 통합하지 않고 서비스별로 분리했다는 점입니다.

BRANDI와 HIVER는 각각 독립적인 도메인과 트래픽 패턴을 가지고 있습니다. ALB를 분리하면 한 서비스에 트래픽이 집중되더라도 다른 서비스에 영향이 전파되지 않습니다. ADMIN은 일반 사용자가 접근하는 BRANDI·HIVER와 트래픽 성격이 근본적으로 다르기 때문에, 별도 ALB로 분리하는 것이 보안과 운영 두 측면 모두에서 유리합니다.

> #### ALB를 통한 Admin 접근 vs SSM Session Manager 접근
> 두 접근 방식은 목적이 다릅니다.
> - **ALB → Admin**: 상품 등록, 고객 관리 등 서비스 운영을 위한 HTTPS 기반 웹 접속입니다.
> - **SSM Session Manager**: 서버 로그 확인, 쉘 환경 접속 등 인프라 관리를 위한 접속입니다.
>
> 즉, 전자는 "서비스를 운영하는 사람"을 위한 것이고, 후자는 "인프라를 관리하는 사람"을 위한 것입니다.

### Aurora Multi-AZ

Aurora는 MySQL·PostgreSQL과 호환되는 AWS 관리형 관계형 데이터베이스입니다. 일반 RDS 대비 읽기 성능이 뛰어나고, 장애 발생 시 Failover 시간이 30초 이내로 짧습니다.

Multi-AZ 구성은 Primary 노드에 장애가 발생했을 때 Standby 노드가 자동으로 Primary로 승격되어 서비스 중단을 최소화합니다. 커머스 플랫폼에서 주문, 결제 데이터는 단 한 건의 유실도 허용되지 않기 때문에, 이중화 구성은 선택이 아닌 필수입니다.

### S3 + Glue + Personalize

세 서비스가 연결되어 개인화 추천 파이프라인을 구성합니다.

**S3**는 상품 이미지, 사용자 행동 로그 같은 비정형 데이터를 저장하는 오브젝트 스토리지입니다. **Glue**는 S3에 쌓인 로 데이터를 정제·변환하는 서버리스 ETL(Extract-Transform-Load) 서비스로, 별도의 서버 없이도 데이터 파이프라인을 구성할 수 있습니다. **Personalize**는 정제된 행동 데이터를 학습해 개인화 추천 모델을 만들어주는 머신러닝 서비스입니다.

전체 흐름은 다음과 같습니다.

> 사용자 행동 로그 → S3 적재 → Glue로 정제 → Personalize 학습 → 추천 결과 제공

### SNS + Lambda + DynamoDB

이 세 서비스는 주문 완료 이후의 후처리 작업을 비동기로 처리하는 데 활용됩니다.

**SNS**(Simple Notification Service)는 이벤트 발생 시 여러 구독자에게 메시지를 동시에 전달하는 메시지 브로커입니다. 예를 들어 주문 완료 이벤트가 발생하면 SNS가 이를 구독하는 Lambda 함수들에게 동시에 전달합니다. **Lambda**는 이벤트가 발생했을 때만 실행되는 서버리스 함수입니다. 트래픽이 없을 때는 비용이 발생하지 않기 때문에, 간헐적으로 실행되는 후처리 로직에 적합합니다. **DynamoDB**는 NoSQL 데이터베이스로, 주문 상태나 알림 내역처럼 빠른 읽기·쓰기가 필요한 데이터를 처리하는 데 사용됩니다.

이 구조를 활용하면 주문 완료 후 알림 발송, 포인트 적립, 통계 갱신 같은 여러 후처리 작업을 서로 독립적으로 비동기 처리할 수 있습니다.

### SQS + Batch + StepFunctions

정산, 쿠폰 발급, 재고 집계처럼 대량으로 처리해야 하는 배치 작업을 위한 구성입니다.

SNS가 "동시에 여러 곳에 전달"이라면, **SQS**(Simple Queue Service)는 "순서대로 하나씩 처리"에 가깝습니다. 처리 속도가 다른 두 시스템 사이에서 메시지를 버퍼링해 과부하를 방지합니다. **Batch**는 대량 배치 작업을 EC2 인스턴스 풀에서 실행하는 서비스입니다. **StepFunctions**는 여러 Lambda나 Batch 작업을 순서대로, 또는 조건에 따라 실행하는 워크플로 오케스트레이터 역할을 합니다.

예를 들어 다음과 같은 흐름을 StepFunctions 하나로 묶어 관리할 수 있습니다.

> 주문 마감 → 정산 집계(Batch) → 결과 검증(Lambda) → 정산 완료 알림(SNS)

### API Gateway + Lambda

**API Gateway**는 외부에서 들어오는 API 요청을 받아 Lambda나 EC2로 라우팅하는 관리형 API 서버입니다. Lambda와 조합하면 EC2를 상시 띄우지 않아도 API를 운영할 수 있습니다. 쿠폰 발급, 이벤트 조회처럼 요청 패턴이 산발적인 API는 EC2 24시간 운영보다 Lambda로 처리하는 것이 비용 면에서 훨씬 효율적입니다.

### CodeCommit → CodeBuild → CloudFormation (CI/CD)

**CodeCommit**은 AWS 관리형 Git 저장소입니다. 개발자가 코드를 Push하면 **CodeBuild**가 자동으로 빌드와 테스트를 실행하고, 통과하면 **CloudFormation**이 인프라 변경 사항을 배포합니다. CloudFormation은 AWS 리소스를 YAML·JSON 템플릿으로 정의하고 배포하는 IaC(Infrastructure as Code) 서비스입니다. 인프라 변경을 수작업이 아닌 코드로 관리하기 때문에 실수를 줄이고 변경 이력을 추적할 수 있습니다.

---

# 3. 보안 관점 분석

## 3.1 SNS 이벤트 검증

SNS → Lambda → DynamoDB로 이어지는 비동기 파이프라인은 유연하고 확장성 있는 구조입니다. 그런데 여기서 한 가지 생각해볼 점이 있습니다. Lambda가 SNS에서 받은 이벤트 메시지를 어디까지 신뢰하고 처리하느냐의 문제입니다.

예를 들어 주문 완료 이벤트가 SNS를 통해 Lambda로 전달되고, Lambda가 이를 그대로 DynamoDB에 적재하는 구조라면, SNS 토픽에 메시지를 Publish할 수 있는 권한이 있는 누군가가 악의적인 페이로드를 넣었을 때 그대로 처리될 수 있습니다. 실제로 내부 서비스만 Publish 권한을 가지도록 SNS 토픽 정책을 제한하고 있다면 문제가 없지만, IAM 권한이 느슨하게 설정되어 있다면 이벤트 주입(Event Injection) 경로가 열릴 수 있습니다.

Lambda 내부에서도 수신한 메시지의 형식과 범위를 검증한 뒤 처리하는 방어적 코딩이 필요합니다. 이벤트 기반 아키텍처는 구성 요소 간 결합이 느슨한 만큼, 신뢰 경계를 어디에 두는지를 명확히 설계해야 합니다.

## 3.2 Personalize 파이프라인 속 PII

S3 → Glue → Personalize로 이어지는 파이프라인에는 사용자의 행동 로그가 흐릅니다. 어떤 상품을 봤는지, 장바구니에 담았는지, 구매했는지 같은 데이터입니다. 추천 모델을 만들기 위한 데이터지만, 동시에 개인 식별이 가능한 PII(개인정보)가 포함될 수 있습니다.

이 파이프라인에서 확인해야 할 것은 두 가지입니다. 첫째, S3 버킷에 불필요하게 넓은 접근 권한이 열려 있지는 않은지입니다. 로그가 쌓이는 버킷이 퍼블릭이거나 계정 내 모든 서비스가 접근 가능한 상태라면, 한 서비스가 침해됐을 때 전체 행동 로그가 노출될 수 있습니다. 둘째, Glue 잡이 데이터를 정제하는 과정에서 사용자 식별자를 그대로 넘기는지, 아니면 가명처리나 해시를 거쳐 Personalize에 전달하는지입니다. 추천 모델 학습에는 굳이 원본 사용자 ID가 필요하지 않은 경우가 많습니다.

---

### Ref

브랜디 AWS 사례, https://aws.amazon.com/ko/solutions/case-studies/brandi/
AWS WAF 개요, https://aws.amazon.com/ko/waf/
AWS Secrets Manager, https://aws.amazon.com/ko/secrets-manager/
AWS CloudTrail, https://aws.amazon.com/ko/cloudtrail/
Amazon GuardDuty, https://aws.amazon.com/ko/guardduty/