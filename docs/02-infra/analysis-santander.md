![](https://velog.velcdn.com/images/hvvup/post/4aa3d491-5b4e-485e-a33d-71e8bc3cbf85/image.png)


# 1. 개요

## 1.1 인프라 구성 요약

Santander는 영국의 금융 회사로, 다음과 같은 특징을 가지고 있습니다.
- 매일 수십억 건의 트랜잭션을 처리하고 200개 이상의 핵심 시스템을 운영하는 초대형 금융 인프라를 보유
- 투자은행, 자산관리, 보험, 결제 솔루션까지 다양한 금융 서비스로 사업 영역이 확장되면서, 기술 인프라의 복잡성이 증가

이 상황은 단순한 운영 문제를 넘어서 보안 사고로까지 이어질 수 있습니다. 시스템 간 연계가 늘어날수록 공격 표면은 확대되고 가시성 확보는 점점 어려워집니다.

## 1.2 분석 범위 및 목적

Santander가 직면한 두 가지 문제가 있었는데, 각각이 무엇인지 보고 이 문제들이 어떻게 보안적으로 이어지는지를 확인해보겠습니다.

> **Cloud Provisioning**
클라우드 프로비저닝은 클라우드 리소스를 보다 효과적으로 관리하는 프로세스입니다. 여기에는 가상 머신(VM), 스토리지, 네트워크와 같은 클라우드 기반 리소스를 생성, 구성 및 배포하는 것이 포함됩니다. 클라우드 프로비저닝을 사용하면 클라우드 채택 또는 사내 또는 사내 리소스에서 클라우드 제공업체로 이전하는 것이 더 쉬워집니다. 클라우드 인프라가 조직의 특정 요구 사항을 충족하도록 보장하여 전반적인 효율성과 성능을 향상시킬 수 있습니다. [Cloud Provisioning Basics: Managing Cloud Resources](https://duplocloud.com/blog/cloud-provisioning-basics/)

이러한 배경 속에서 Santander는 Catalyst라는 플랫폼 엔지니어링 이니셔티브를 도입해 클라우드 인프라와 개발 관리 체계를 전면 재편하게 됩니다.


# 2. 아키텍처 분석

## 2.1 전체 구성도

<p align="center"><img src="https://d2908q01vomqb2.cloudfront.net/fc074d501302eb2b93e2554793fcaf50b3bf7291/2026/01/26/ARCHBLOG-12991.png" width="100%"></p>

## 2.2 구성 요소별 역할

### Control Plane

<p align="center"><img src="https://velog.velcdn.com/images/hvvup/post/4e3ea4d3-8d7f-454f-a93e-011f2f0308ca/image.png" width="70%"></p>

- Catalyst 전체를 제어하는 중앙 관리 영역
- Platform Namespace와 Data Plane Namespace로 구분됩니다

### Crossplane

 <p align="center"><img src="https://raw.githubusercontent.com/crossplane/artwork/master/logo/logo-horizontal-whitetext-bluebg.png" width="45%"></p>

우측에 보이는 아이스크림 모양의 서비스가 어떤 걸 가리키는 건지 모르겠어서 간단히 정리하고 넘어가려고 합니다.

[What's Crossplane](https://docs.crossplane.io/latest/whats-crossplane/?_gl=1*4qo7x5*_ga*MTAwNjcwMDQzOC4xNzc1Mzc0MjI0*_ga_SFCPQYSLHY*czE3NzUzNzQyMjQkbzEkZzAkdDE3NzUzNzQyMjQkajYwJGwwJGgw)에 따르면 Crossplane은 Cloud Native 소프트웨어를 관리할 수 있는 Control Plane을 구축할 수 있는 오픈소스라고 합니다.

여기서 클라우드 환경에서의 Control Plane에 대해 다루고 있어 간단히 개념에 대해 알아보았습니다.

<p align="center"><img src="https://cdn.prod.website-files.com/5ff66329429d880392f6cba2/675754fc02e7f87241950937_634e663b07cfd8b83e52a4cc_482.2-min.jpeg" width="65%"></p>

Control plane은 실제 데이터 전송이 아니라 네트워크에서 데이터 패킷의 흐름과 전달을 관리하는 라우팅 컴포넌트의 역할을 수행하면서 다음과 같은 과정에 관여합니다.

- 서로다른 네트워크 요소 사이에서 커뮤니케이션을 오케스트레이션
- 라우팅, 설정, 리소스 할당, 정책 강제와 관련된 결정

Cloud 환경은 본질적으로 분산 네트워크 시스템이므로 여러 서버, 서비스, 리전에 걸쳐 데이터가 오가기 때문에 패킷이 정확히 어디로 가야 하는지를 정밀하게 제어하는 Control Plane의 역할이 훨씬 커진다고 합니다.

특히 k8s 같은 컨테이너 오케스트레이션 환경에서 Control Plane은 클러스터 전체의 상태를 감시하고 스케줄링/정책을 결정하는 핵심 계층으로 작동합니다.

이 개념이 보안에서 중요한 이유는 다음과 같습니다.

#### Platform Namespace

- **Data Plane Claims (ArgoCD)**
   - ArgoCD가 Git 저장소를 지속적으로  Data Plane에 배포되어야 할 스택들의 상태를 동기화
   - 인프라의 의도된 상태를 지속적으로 실제 상태에 반영하는 역할을 합니다.
- **Policies Catalog (OPA)**
   - 모든 리소스 생성 요청에 대해 사전 정의된 정책을 검사합니다
   - 어떤 요청이든 실행 전에 반드시 정책 검증을 통과해야 함
- **Stacks Catalog (CRDs + Compositions)**
   - Crossplane의 Composite Resource Definitions과 Compositions의 라이브러리
   - 개발자가 선택 가능한 사전 승인된 인프라 패턴의 목록

참고) ArgoCD: Kubernetes 환경에서 GitOps 방식으로 애플리케이션 배포를 자동화하는 도구

#### Platform Namespace + Data Plane Namespace

개발자가 요청을 보내면 API Server가 수신하고, Admission Controller가 이를 가로채 create namespace, create XR(Composite Resource)를 실행합니다.
생성된 XR은 Data Plane Namespace에 위치하며, 이것이 실제 Data Plane으로 전달될 작업 단위가 됩니다.

이는 모든 인프라 요청이 단일 컨트롤 플레인을 통과하도록 강제함으로써, 표준화나 정책 준수 및 감사 추적을 구조적으로 보장하도록 한 설계입니다.

이렇게 하면 개발자가 직접 할 필요 없이, 모든 과정이 자동화됩니다.


### Data Plane

실제 워크로드가 배포되는 런타임 영역입니다.

<p align="center"><img src="https://velog.velcdn.com/images/hvvup/post/4d5c973b-347c-4c94-a398-cbbcfd053ec8/image.png" width="30%"></p>

Control Plane의 지시를 받아 실제 애플리케이션 스택이 배포되고 실행되는 영역으로, 이 다이어그램에는 네 가지 스택이 나열되어 있습니다.

|스택|역할|
|-|-|
|Rusteze Stack|내부 플랫폼 도구 또는 특수 목적 런타임 (Crossplane provider 관련 추정)|
|Microfrontend Stack|프론트엔드 서빙 레이어|
|Backend Stack|API 및 비즈니스 로직 처리 레이어|
|Data Platform Stack|데이터 분석 및 처리 레이어|

각 스택은 독립적인 단위로 관리되며, Stacks Catalog에서 정의된 Composition을 기반으로 표준화된 방식으로 생성됩니다.

위와 같이 스택을 목적별로 분리함으로써 각 워크로드의 생명주기를 독립적으로 관리할 수 있게 됩니다(하나의 스택에 장애나 변경이 생겨도 다른 스택에 영향을 주지 않는 격리된 구조).

또, Control Plane이 각 스택을 개별적으로 추적 및 동기화할 수 있어 세밀한 거버넌스가 가능해지는 장점도 있습니다.


# 3. 보안 관점 분석

## 3.1 현재 보안 구성 현황

<p align="center"><img src="https://velog.velcdn.com/images/hvvup/post/8128e46f-795d-4c80-aa1a-115562122eaa/image.png" width="70%"></p>

사용자의 요청을 WAF에서 악성 트래픽을 필터링한 뒤, CloudFront를 통해 서빙됩니다. 이때 모든 접근 기록은 Access Log로 남겨지며, 저장되는 데이터는 KMS로 암호화됩니다. 콘텐츠 자체도 별도의 KMS로 암호화되는 이중 암호화 구조를 확인할 수 있습니다.

외부 요청은 마찬가지로 WAF를 거쳐 API Gateway로 진입합니다. **VPCLink + NLB**를 통해 프라이빗 네트워크 내의 Microservices Namespace로 라우팅됩니다.

이 마이크로서비스들은 캐싱을 위한 Redis Serverless, 관계형 데이터를 위한 Postgres, 대규모 분석을 위한 Databricks Workspace를 활용하며, 각 스토리지는 개별 KMS 키로 암호화됩니다.

리소스 레이어에서 주목할 점은 모든 스택이 WAF로 시작하고 KMS로 끝난다는 패턴입니다. 개발자가 별도로 WAF나 암호화를 설정하지 않아도, 스택을 선택하는 순간 자동으로 적용되는 구조로 강제됩니다.

## 3.2 식별된 취약점

### 1. 프로비저닝된 서비스가 정의된 아키텍처를 준수하는지 보장하는 문제

금융권에서 아키텍처 정의를 벗어난 인프라 구성은 곧 컴플라이언스 위반으로 직결되는데 PCI-DSS, SOX, GDPR 등 다양한 규제를 동시에 준수해야 하는 금융기관에서  수작업 기반의 프로비저닝은 휴먼 에러와 보안 설정 오류로 이어질 수 있습니다.

### 2. 인프라 프로비저닝에 최대 90일이 소요됨
보안 측면에서 긴 프로비저닝 주기는 단순히 속도의 문제가 아니게 됩니다. 취약점이 발견되었을 때 패치된 인프라로 신속하게 전환하지 못하는 것을 의미하며, 이는 **Mean Time to Remediate(MTTR)**를 높여 사이버 공격에 대한 노출 시간을 늘릴 수 있습니다.

## 3.3 개선 권고사항

### VPC Link NLB

> VPC 링크를 사용하여 API 라우팅을 VPC의 프라이빗 리소스(예: Application Load Balancer 또는 Amazon ECS 컨테이너 기반 애플리케이션)에 연결하는 프라이빗 통합을 생성할 수 있습니다. 프라이빗 통합은 VPC 링크 V2를 사용하여 API Gateway와 대상 VPC 리소스 간의 연결을 캡슐화합니다. 서로 다른 리소스 및 API에서 VPC 링크를 재사용할 수 있습니다. [API Gateway에서 VPC 링크 V2 설정](https://docs.aws.amazon.com/ko_kr/apigateway/latest/developerguide/apigateway-vpc-links-v2.html)

<p align="center"><img src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2021/08/10/Screen-Shot-2021-08-10-at-1.33.13-PM.png" width="80%"></p>

API Gateway는 기본적으로 퍼블릭 인터넷에 있는 데 반해, 실제 백엔드 서비스들은 보안을 위해 프라이빗 VPC 안에 격리된 구조가 많습니다.

그렇다면 API gw가 어떻게 백엔드를 호출하게 할까요? 이 상황에서 vpc link를 통해 우회하도록 하는 것입니다. API GW가 L7 요청을 모두 처리하고 남은 tcp stream만 vpc link를 통해 백엔드로 넘깁니다.

또한 NLB가 이 구조에서 가지는 장점을 몇 가지 정리해보겠습니다.

- **고정 IP**: NLB는 가용영역당 고정 IP를 가지므로 VPC Link를 통해 들어오는 트래픽의 출처 IP를 보안 그룹 규칙에 명시적으로 고정시킬 수 있습니다. (ALB는 IP가 바뀜)
- **프로토콜 투명성**: NLB는 HTTP뿐 아니라 gRPC, WebSocket, 커스텀 TCP 프로토콜 등 TCP 위에서 동작하는 모든 것을 백엔드로 그냥 포워딩합니다. (ALB는 HTTP/HTTPS/gRPC만 이해할 수 있음)
- **레이턴시**: NLB는 패킷을 파싱하지 않아서 ALB보다 레이턴시가 수십 ms 정도 낮음

여기서 vpc link 다음으로 요구사항, 처리해야할 작업 등에 의해 nlb가 나올 수도, alb(RestAPI v2의 경우)가 나올 수도 있습니다... 이런 케이스, 그러니까 여기서 어떤 요청을 처리할 거고 이 요청에서 가져와야하는 정보가 무엇인지에 따라 달라지는 듯합니다. 이런 부분에 대한 고민, 공부도 더 필요할 것 같습니다.

이러한 구조를 사용함으로써 얻을 수 있는 보안성은 다음과 같습니다.

**1. 백엔드 서비스에 퍼블릭 IP가 필요 없음**
API Gateway가 백엔드를 HTTP로 직접 호출하려면 해당 서버가 인터넷에 노출되어 있어야 합니다.

VPC Link를 사용하면 백엔드 서버는 프라이빗 서브넷에 격리된 채로 있을 수 있고 아예 퍼블릭 IP 자체도 필요 없으므로 공격 표면이 줄어듭니다.
```
# 기존 방식 (백엔드가 퍼블릭에 노출되어 있어야 함)
[API Gateway] ──HTTPS──→ [백엔드 퍼블릭 IP:8080] ← 누구나 접근 가능

# VPC Link 방식
[API Gateway] ──VPC Link──→ [NLB 프라이빗 IP] ──→ [백엔드 프라이빗 IP]
                                                         ↑ 인터넷에서 접근 불가
```

**2. 트래픽이 AWS 내부 백본을 통해서만 이동**
VPC Link를 통과하는 트래픽은 퍼블릭 인터넷을 거치지 않습니다. AWS의 프라이빗 네트워크 인프라(AWS Global Backbone)를 통해서만 이동하기 때문에 중간 경로에서 패킷이 노출되거나 도청될 가능성이 없습니다.

**3. 보안 그룹으로 세밀한 접근 제어가 가능**
이 구조에서 VPC Link의 트래픽은 결국 NLB를 통해 들어옵니다. NLB의 보안 그룹과 백엔드의 보안 그룹으로 접근을 더 세밀하게 제한하거나 관리하는 것이 가능합니다.
```
NLB 보안 그룹:
  - allow 443: API Gateway VPC Link ENI CIDR만

백엔드 보안 그룹:
  - allow 8080: NLB 보안 그룹에서 오는 트래픽만 허용하고 나머지 차단
  ```
이렇게 하면 `NLB → 백엔드` 의 트래픽을 NLB에서 오는 것만 허용하므로, 다른 경로의 백엔드 접근은 무시됩니다.

**4. IAM 정책으로 VPC Link 자체를 잠금**
API Gateway에서 VPC Link를 호출하는 행위 자체도 IAM 정책으로 제어할 수 있습니다. 어떤 API Gateway 스테이지/라우트가 어떤 VPC Link를 사용할 수 있는지를 정책 레벨에서 강제할 수 있는 것입니다.

여러 레이어를 통한 보안이 필요한 만큼, 이를 통해 더 안전하게 api gw를 구성할 수 있을 것입니다.

---

출처:

- [Digital Transformation at Santander: How Platform Engineering is Revolutionizing Cloud Infrastructure](https://aws.amazon.com/ko/blogs/architecture/digital-transformation-at-santander-how-platform-engineering-is-revolutionizing-cloud-infrastructure/)
- [What is the Control Plane? Explained by Wallarm](https://www.wallarm.com/what/what-is-the-control-plane)
- [Control Plane in Cloud Security: Control Plane vs. Data Plane](https://www.crowdstrike.com/en-us/cybersecurity-101/cloud-security/control-plane/#:~:text=What%20is%20a%20control%20plane,and%20emulate%20legitimate%20user%20behavior.)
- [Understanding VPC links in Amazon API Gateway private integrations](https://aws.amazon.com/ko/blogs/compute/understanding-vpc-links-in-amazon-api-gateway-private-integrations/)
