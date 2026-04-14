
<p align="center"><img src="https://www.sotatek.com/wp-content/uploads/2024/01/Madworld.png" width="80%"></p>


[MADworld: AWS Successful Case Study](https://www.sotatek.com/blogs/software-development/madworld-aws-successful-case-study/)

## 1. 개요

### 1.1 인프라 구성 요약

MADworld는 NFT 기반의 블록체인 게임 플랫폼으로, SotaTek이 설계 및 구축한 AWS 클라우드 인프라 위에서 운영됩니다. 참고한 문서도 SotaTek의 Case study에서 참고했습니다. 해당 아키텍처는 사용자 트래픽을 처리하는 프론트엔드 레이어부터 백엔드 API 서버, 미디어 처리 파이프라인, 데이터 저장소, 외부 블록체인 노드 연동까지 복수의 계층으로 구성되었습니다.

이 아키텍처의 키워드는 다음과 같습니다.

- Serverless
- Scalability
- Cost optimization

전체 구성은 크게 다음의 세 가지 영역으로 나눌 수 있습니다.

- **사용자 접근 및 콘텐츠 전달 레이어**: Route 53, CloudFront, ACM, AWS WAF, ELB
- **API 및 비즈니스 로직 레이어**: API Gateway, AWS Lambda (3종), ECS (Service API / Batch Jobs)
- **데이터 및 미디어 처리 레이어**: S3, SNS, SQS, PostgreSQL, Redis Cluster, Blockchain Fullnode (3rd-party)

인프라 전반에 걸쳐 서버리스(Lambda)와 컨테이너(ECS)를 혼합 활용하고, SNS/SQS를 사용한 아키텍처를 통해 미디어 처리를 비동기로 분리한 점이 특징적입니다. 또한 블록체인 플랫폼의 특성상 외부 풀노드(Blockchain Fullnode 3rd)와의 연동이 아키텍처의 핵심 구성 요소 중 하나를 차지하고 있습니다.

---

## 2. 아키텍처 분석

### 2.1 전체 구성도

전반적으로 계층화된 구조를 가지고 있으며, 각 계층 간 책임이 비교적 명확하게 분리되어 있습니다. 사용자 접근 경로는 Route 53을 기점으로 두 갈래로 나뉩니다. 정적 콘텐츠는 CloudFront를 통해 직접 서빙되며, 동적 요청은 ELB ECS 경로를 통해 처리됩니다.

### 2.2 구성 요소별 역할

|서비스|설명|
|-|-|
|API Gateway|외부에서 Lambda 함수로의 진입점으로 인증(ApiGwAuthorizedFunction)을 포함한 요청 제어를 담당|
|SNS / SQS|미디어 업로드 이벤트를 SNS로 발행하고, SQS를 통해 Lambda가 폴링 방식으로 소비하는 비동기 이벤트 파이프라인. 처리 실패 시 메시지 재처리 및 DLQ(Dead Letter Queue) 구성이 중요|
|ECS (Elastic Container Service)|컨테이너 기반으로 Service API와 Service Batch Jobs를 운영하며 Auto Scaling을 통해 부하에 따른 탄력적 확장을 지원|
|Blockchain Fullnode (3rd-party)|외부 블록체인 네트워크와의 연동을 담당하는 외부 의존성. Service API가 직접 호출하는 구조로, 외부 네트워크와의 경계 보안이 중요|


#### AWS Lambda

| 함수명 | 역할 |
|--------|------|
| ApiGwAuthorizedFunction | API 요청의 인증/인가 처리, SSM을 통해 보안 파라미터 조회 |
| GenPresignedS3Function | S3 업로드를 위한 Pre-signed URL 생성 및 Callback 처리 |
| MediaResizeImageFunction | S3 이벤트 또는 SQS 메시지를 수신하여 이미지 리사이징 처리 |


---

## 3. 보안 관점 분석

### 3.1 현재 보안 구성 현황

아키텍처 전반에 걸쳐 다음과 같은 보안 메커니즘이 적용되어 있는 것을 확인할 수 있습니다.

**전송 구간 암호화 (TLS/SSL)**
ACM을 통해 발급된 인증서가 CloudFront 및 ELB에 적용되어 있어 사용자와 서버 간의 통신 구간은 HTTPS로 암호화됩니다.

**애플리케이션 레이어 방화벽 (WAF)**
AWS WAF가 ELB 앞단에 배치되어 있어, SQL Injection, XSS, 비정상 요청 등 L7 수준의 공격에 대한 기본적인 방어 레이어가 존재합니다.

**인증/인가 분리 (ApiGwAuthorizedFunction)**
API Gateway에서 Lambda Authorizer를 통해 요청 인증을 분리하고 있는 것으로 보입니다. 이렇게 인증을 중앙화하는 구조는 몇 가지 관점에서 보안적으로 안전하다고 볼 수 있습니다.

1. 인증 로직 누락 방지
인증 코드를 각 Lambda 함수나 API 핸들러마다 따로 작성하면, 개발자가 특정 엔드포인트에서 실수로 빠뜨릴 수 있습니다. 중앙화하면 인증을 거치지 않고는 아예 비즈니스 로직에 도달할 수 없는 구조가 되므로, 인간 실수로 인한 인증 우회 가능성이 줄어듭니다.
2. 취약점 수정 범위 최소화
인증 로직에 버그나 취약점이 발견됐을 때 분산된 구조라면 관련된 모든 함수를 찾아서 일일이 수정해야 합니다. 중앙화된 구조에서는 한 곳만 수정하면 전체에 즉시 반영되므로 패치 누락으로 인한 보안 구멍이 생기지 않습니다.
3. 일관된 인증 정책 적용
토큰 검증 방식, 만료 시간 체크, 권한 레벨 판단 기준 등을 한 곳에서 통제할 수 있어, 서비스 전체에 동일한 보안 기준이 적용됩니다. 함수마다 구현이 달라 생기는 정책 불일치 문제를 방지합니다.
4. 감사(Audit) 및 모니터링 집중화
인증 시도, 실패, 토큰 이상 등의 이벤트를 한 함수에서만 로깅하면 되므로 비정상적인 접근 패턴을 탐지하고 대응하기가 훨씬 쉬워집니다. 분산된 구조에서는 여러 함수의 로그를 전부 수집해서 상관분석을 해야 하는 부담이 생깁니다.

**Pre-signed URL 기반 S3 업로드**
클라이언트가 S3에 직접 파일을 업로드할 때 GenPresignedS3Function을 통해 임시 서명 URL을 발급받는 구조는 클라이언트에 AWS 자격증명을 노출하지 않을 수 있습니다.

**시크릿 관리 (SSM Parameter Store)**
Lambda에서 하드코딩된 자격증명 대신 SSM을 통해 파라미터를 동적으로 로드하는 방식은 시크릿 관리의 기본 원칙에 부합합니다.

**비동기 미디어 처리 분리 (SNS/SQS)**
미디어 처리 파이프라인을 이벤트 기반으로 분리함으로써 미디어 처리 실패가 핵심 API 서비스에 영향을 미치지 않도록 격리한 설계는 가용성 측면에서도 고려해볼 수 있는 구조입니다.

---

### 3.2 식별된 취약점

현재 아키텍처 다이어그램을 기반으로 보안 관점에서 잠재적 취약점으로 판단되는 영역은 다음과 같습니다.

(다만 해당 아키텍처에서는 간소화되었거나 구현되었으나 표시되지 않은 부분이 있을 수 있습니다. 이 글에서는 해당 가능성을 고려하지 않고 작성하였습니다.) 

**1. CloudFront와 S3 간의 오리진 접근 제어 불명확**
다이어그램 상에서 CloudFront와 S3(UI 버킷) 간의 접근 제어 방식이 명시되어 있지 않습니다. S3 버킷이 퍼블릭으로 열려 있을 경우 CloudFront를 우회하여 S3 URL로 직접 접근이 가능해집니다. CloudFront Origin Access Control(OAC) 또는 구 방식의 OAI(Origin Access Identity)가 적용되지 않았다면, 버킷 정책 수준에서 직접 접근이 허용될 수 있습니다.

**2. Blockchain Fullnode(3rd-party) 연동의 신뢰 경계 불명확**
Service API가 외부 블록체인 풀노드와 직접 통신하는 구조에서 해당 외부 노드에 대한 인증 및 검증 메커니즘이 다이어그램에 표현되지 않습니다. 블록체인 트랜잭션 서명 키, RPC 엔드포인트 자격증명이 노출되거나 중간자 공격(MITM)에 노출될 수 있고 이는 추가적인 공격 및 데이터 유출로 이어질 수 있습니다.

**3. WAF가 API Gateway 경로에는 미적용**
현재 WAF는 ELB에서 ECS로 이어지는 경로에만 위치하고 있습니다. API Gateway 경로는 WAF가 적용되어 있지 않은 것으로 보이며, 해당 경로를 통한 Lambda 호출은 L7 필터링 없이 처리될 수 있습니다. API Gateway에 자체 Throttling은 존재하지만, WAF 수준의 악성 요청 차단은 별도로 구성해야 합니다.

여기서 고려해볼 수 있는 몇 가지 방법이 있습니다.

1. API Gateway를 VPC Endpoint로 전환 (Private API)
- 인터넷에서 API Gateway로의 직접 접근 자체가 불가능해지므로, WAF 적용 이전에 공격 노출 표면 자체를 제거하는 효과가 있습니다
(단, 이 경우 ECS나 Lambda 같은 내부 리소스에서만 API를 호출할 수 있으므로, 외부 클라이언트-모바일 앱, 브라우저-가 직접 호출해야 하는 구조라면 적용이 어렵기는 합니다.)

2. API Gateway에 WAF 직접 적용
- API Gateway 자체 Throttling은 단순히 요청 수만 제한하지만, WAF는 요청의 내용(헤더, 바디, URI 패턴) 을 분석하여 차단하므로 더 높은 수준의 보안이 가능합니다.
- AWS Managed Rule Group을 적용하면 SQL Injection, XSS, 알려진 악성 IP 등을 별도 구현 없이 차단할 수 있습니다
- Rate-based Rule을 WAF 레벨에서 추가 구성하면 특정 IP의 과도한 요청을 Throttling보다 정교하게 제어할 수 있습니다

3. API Gateway 리소스 정책으로 허용 IP/소스 제한
- WAF보다 단순하지만 설정 부담이 적고 화이트리스트 기반 접근 제어를 빠르게 구현할 수 있습니다
- WAF와 함께 적용하면 WAF가 뚫리더라도 리소스 정책이 2차 방어선 역할을 합니다
- CloudFront의 IP 대역을 허용 목록으로 지정해 CloudFront를 반드시 경유하도록 강제할 수 있습니다

**WAF 배치 위치별 비교**
WAF는 AWS에서 여러 레이어에 붙일 수 있으며, 위치마다 탐지할 수 있는 영역이 다릅니다.

|배치 위치|적용 범위|주요 장점|한계|
|-|-|-|-|
|CloudFront|엣지(전 세계 POP)|한국 지역에 도달하기 전에 차단, DDoS 흡수, 지리적 IP 차단|오리진 직접 접근 시 우회 가능|
|API Gateway|API 엔드포인트 진입 직전|Lambda 호출 전 L7 필터링, API 특화 룰 적용 가능|정적 콘텐츠 경로(S3/CloudFront)에는 미적용
|ELB (ALB)|ECS 서비스 진입 직전|컨테이너 서비스 보호, 기존 구성에 이미 적용됨|API Gateway 경로는 별도 적용 필요|
|AppSync|GraphQL API|GraphQL 특화 공격 차단|GraphQL을 사용하지 않으면 해당 없음|

각 위치에 WAF를 추가했을 때의 특징을 살펴보겠습니다.

- CloudFront
글로벌 엣지에서 가장 먼저 차단하므로 지연 없이 대규모 트래픽을 필터링할 수 있지만, 오리진(API Gateway)의 URL이 노출되면 CloudFront를 완전히 우회할 수 있습니다.
- API Gateway
오리진 직접 접근까지 방어할 수 있지만, 악성 트래픽이 CloudFront를 통과하여 엣지 캐시 오염이나 대역폭 낭비가 발생할 수 있습니다.


이 아키텍처에서는 ELB에만 WAF를 두는 구조를 채택하고 있으나, 추가적으로 WAF를 적용한다면 CloudFront에 두는 것을 고려해볼 수 있을 것으로 보입니다.

```
경로 A: [Client] → [CloudFront] → [S3 / API Gateway]
경로 B: [Client] → [Route 53] → [ELB] → [ECS]
```

이는 MADworld의 아키텍처가 가지고 있는 두 갈래의 유저 접근 경로 모두의 상단에서 글로벌 봇, 지리적 IP 차단, 대규모 DDoS를 엣지에서 먼저 흡수합니다. API Gateway 경로도 CloudFront를 통해 강제 경유하도록 리소스 정책을 설정하면, CloudFront WAF가 사실상 API Gateway까지 커버할 수 있습니다.


**4. API Gateway에 대한 인증 범위 불확실**
ApiGwAuthorizedFunction이 모든 API 엔드포인트에 적용되는지, 일부 퍼블릭 엔드포인트에는 적용되지 않는지 다이어그램만으로는 파악하기 어렵습니다. 인증되지 않은 엔드포인트가 존재할 경우, 의도치 않은 데이터 노출 또는 기능 오용의 위험이 있습니다.

**5. SQS 메시지에 대한 암호화 및 DLQ 구성 불명확**
SNS → SQS 파이프라인을 통해 전달되는 메시지에 민감한 메타데이터(파일 경로, 사용자 ID 등)가 포함될 수 있습니다. SQS 서버 측 암호화(SSE) 적용 여부 및 처리 실패 시 DLQ(Dead Letter Queue) 구성 여부가 확인되지 않습니다. DLQ 미적용 시 실패 메시지가 무한 재처리되거나 유실될 수 있습니다.

**6. ECS 컨테이너 IAM 역할 과잉 권한 위험**
ECS Task에 부여된 IAM Role의 권한 범위가 명시되지 않습니다. PostgreSQL, Redis, Blockchain Fullnode에 접근하는 Service API 컨테이너가 필요 이상의 IAM 권한을 보유할 경우, 컨테이너 탈취 시 피해 범위가 넓어집니다. 최소 권한 원칙(Least Privilege)의 실제 적용 여부 점검이 필요합니다.

**7. PostgreSQL / Redis의 네트워크 격리 수준 불명확**
데이터베이스 레이어가 VPC 내에 위치하는 것으로 추정되지만, 서브넷 분리(퍼블릭/프라이빗) 여부 및 Security Group 설정이 다이어그램에 명시되지 않습니다. 프라이빗 서브넷에 격리되지 않았다면 불필요한 공격 노출면이 발생합니다.

**8. 로깅 및 감사 추적(Audit Trail) 체계 불명확**
CloudTrail, VPC Flow Logs, API Gateway Access Logging, ECS 컨테이너 로깅 등 감사 추적 인프라의 구성 여부가 다이어그램에 표현되지 않습니다. 블록체인·NFT 플랫폼의 특성상 모든 트랜잭션 관련 이벤트에 대한 감사 로그는 규정 준수 및 침해사고 대응 측면에서 필수적입니다.




