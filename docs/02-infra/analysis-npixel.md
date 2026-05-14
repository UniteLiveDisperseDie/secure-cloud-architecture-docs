

# 1. 개요

## 1.1 인프라 구성 요약

회사명: NPIXEL

분야: 게임

고객의 많은 트래픽에 대응하고 해외 시장 진출을 위한 많은 리전이 필요했기에 AWS를 선택하게 되었다고 합니다.

## 1.2 분석 범위 및 목적

가장 먼저 눈에 띄는 것은 ALB와 NLB로 나누어져 트래픽이 들어온다는 것입니다.

- **ALB: Auth / Front 서버 (웹, API)**
- **NLB: Game / Link / Billing 서버 (실시간 서버)**


# 2. 아키텍처 분석

## 2.1 전체 구성도

<p align="center"><img src="https://d2908q01vomqb2.cloudfront.net/91032ad7bbcb6cf72875e8e8207dcfba80173f7c/2020/07/14/stateful-stateless-arch-1020x1024.png" width="55%"></p>

이 구조는 npixel의 인프라와 상당히 유사해 보입니다. (아마 이걸 참고해서 설계했을 수도 있지 않을까요?)

> Some services need a persistent connect, but many can use REST APIs. These two approaches are called stateful and stateless, which is often referred to as RESTful. With RESTful services, the player's mobile device, tablet, PC, or console makes requests to your servers for data such as login, sessions, friends, leaderboards, and trophies. It doesn't maintain long-lived connections to the server, so these services can be on-demand instead of persistent demand. However, for multiplayer features, it's important to keep that connection to the server through a stateful approach.

어쨌든 이 문서에서 말하는 핵심 요소는 **실시간 게임 로직은 지속 연결 서버에서, 나머지는 REST API로 분리하자**는 것입니다. 이러한 연결의 지속성을 고려하는 것이 확장성과 비용 측면에서 유리합니다.

## 2.2 구성 요소별 역할

### Load Balancer

**Application Load balancer**

- L7의 트래픽 처리
- URL 기반 라우팅
- 쿠키/세션 처리 가능

**Network Load Balancer**

- 패킷을 그대로 전달하므로 처리 속도 빠름
- TCP/UDP 지원
- IP 기반 라우팅

여기서 이렇게 '외부에서 접근하는 lb가 두 개로 나뉘는 이유'는 아마도 게임사의 특성일 것입니다. 게임은 특성상

- 프로토콜이 https/http와 같은 웹이 아닌 tcp, udp 등을 사용
- latency 측면에서 nlb가 빠름(ip 기반으로 바로 라우팅)
- NLB는 ALB와 달리 받은 패킷을 그대로 들여보냄

따라서 인증이나 프론트 페이지를 제공하는 것은 alb 쪽으로, 직접적인 게임 서버와의 통신은 nlb로 제공하도록 설계한 것입니다.

당연히 alb/nlb의 역할이 다르겠지만 지금까지 봤던 아키텍처는 alb를 사용한 경우가 대부분이어서 사용자의 트래픽을 nlb가 앞단에서 받는 것은 처음 봤습니다. 대용량 트래픽을 latency 적게 처리하려면 nlb를 두는 것이 효과적이라는 것을 기억해두면 좋겠습니다!!

### CloudFront + S3

CloudFront는 이전에도 구조에서 정말 여러번 다뤄봤지만, 간단히 설명하자면 **엣지에 정적 콘텐츠를 캐싱해두고 사용자의 위치에 따라 가장 가까운 엣지에서 가져와 서빙하는** AWS의 CDN 서비스입니다.

그러니까 프론트엔드 파일들을 s3에 넣어두고, cloudfront가 이걸 끌어와서 빠르게 서비스할 수 있습니다. 그런데 여기서 의아했던 건, 프론트엔드 서비스를 담당하는 서비스가 있다는 것입니다. 그렇다면 cloudfront가 담당하는 부분과, 해당 FE 서버가 제공하는 리소스가 다르다는 말일 것입니다.

왜 이렇게 설계했을지 궁금해져서 자료를 찾아보는데 아래와 같은 bp를 찾았습니다.

게임 서버는 실시간으로 상태가 변하고, 유저간의 상호작용이 계속해서 일어납니다. 이런 연결에서는 서버가 지속적으로 연결을 유지할 필요가 있고 latency가 낮아야 합니다. Rest API는 응답이 들어오면 요청을 처리하는 stateless가 가능한 구조입니다. 여기서 api 서버가 담당하는 것은 사용자의 인증 외에도 인벤토리 조회, 사용자 매칭, 리더보드 등을 보여주는 게임 관련 api일 수도 있습니다.

그리고 이 그림을 통해 알 수 있는 것 중 하나가 S3에 게임 asset binary를 넣어 CloudFront에서 바로 전달했다는 점입니다. 아마도 npixel의 구조도 이러한 용도로 cloudfront를 사용한 것이 아닐까요?

이전까지는 정적 콘텐츠 서빙을 프론트엔드 서버를 대체하는 용으로만 생각했었는데, 이렇게 사용자가 동일하게 다운로드 해야할 데이터를 캐싱해두고 빠르게 다운로드 받을 수 있도록 cdn을 사용하는 것이 효율적으로 보입니다.

### BI/Ops

BI는 Business Intelligence의 약자로 BI/Ops란 **서비스에서 발생한 데이터를 수집·분석하여 운영(Ops)과 의사결정(BI)에 활용하는 시스템입니다.**

- Lambda: 로그 전처리, 데이터 파싱 등의 처리 자동화
- Athena: S3에 있는 로그를 SQL 쿼리 이용해 분석
- Elastic Search: 로그를 빠르게 검색/분석, 해당 아키텍처에서는 aws 네이티브로 제공하는 es를 사용하는 것으로 보임
- Gafna: 대시보드, 실시간 모니터링
- NLP 서버: 로그 분석 자동화, 이상 탐지 및 패턴 분석
- Crawler: 외부 데이터 수집 혹은 로그 보강

여기서는 내가 알고 있는 서비스도, 모르고 있었던 서비스도 있는데 우선은 이 정도만 알고 넘어가려고 합니다. 나중에 기회가 있으면 BIOps에 대해서도 다뤄보면 좋을 것 같습니다.

이 구조에서는 로그 등의 데이터를 보안의 관점에서만 사용하고 있지 않고, 비즈니스 분석에도 활용하고 있다고 이해하면 될 것 같습니다.


# 3. 보안 관점 분석

## 3.1 현재 보안 구성 현황

현재 아키텍처 상으로 구현되어 있는 엣지 로케이션의 보안은 다음과 같습니다.

- WAF + CloudFront
- WAF + ALB
- Shield Advanced + NLB

이 사진은 Shield Standard와 Advanced의 차이점을 잘 보여줘서 가져와봤습니다.

<p align="center"><img src="https://d2908q01vomqb2.cloudfront.net/7b52009b64fd0a2a49e6d8a939753077792b0554/2020/05/06/DDoS3.jpg" width="55%"></p>

보통은 Shield와 ALB를 같이 붙여주면서 보안을 가져가는 것 같은데, 이 구조에서는 그게 불가능합니다. WAF는 L7 방화벽이므로 **NLB에 붙여 사용할 수 없기 때문**입니다.

우선 이전 글에서도 다뤘듯이, 기본적인 L3~4 DDoS에 대응하는 Shield Standard는 AWS에서 모든 계정 및 리전에 기본적으로 제공하고 있습니다. 따라서 ALB에는 따로 붙이지 않은 것으로 보입니다. 당연히 사용하면 좋겠지만 비용도 고려해봤을때 선택하자면 엄청난 양의 네트워크 트래픽이 오가는 NLB에 적용하는 것이 나아 보입니다.

## 3.2 식별된 취약점

그렇다면 Shield Advanced로 NLB에 들어오는 모든 공격을 처리하고 대응하는 것이 가능한 걸까요? 아마도 아닐 것입니다. Shield 서비스는 기본적인 전제가 DDoS 방어에 대한 것입니다. 그러니까 Advanced라고 하더라도 대규모 트래픽에 대한 처리를 담당하지, 패킷 자체의 악성 여부를 보지는 않는다는 것입니다. 그러니까 Shield가 신경쓰는 것은 '어떤 트래픽인지'보다는 '트래픽의 양'입니다.

Shield는 다음과 같은 처리는 할 수 없습니다.

- 특정 포트/프로토콜 제어
- 패킷 내용 검사
- 악성 payload 탐지
- 룰 기반 허용/차단

그래서 ANF(AWS Network Firewall)을 적용하면 어떨까 했습니다. 어쨌든 게임 서버에도 악성 패킷이 들어오지 말라는 법은 없으니까요. 그런데 이 단계에서 ANF를 건너뛴 이유가 다음과 같이 있는 것 같습니다.

Network Firewall을 넣게 되면, 트래픽이 서버에 닿기까지의 홉이 증가합니다.

```
Client → Firewall → NLB → Game Server
```

와 같은 단계로, 한 단계가 더 생기는 것입니다. 그렇게 되면 latency의 측면에서 보장할 수 없습니다.

### 사례 분석

이 부분에 대해 더 궁금해져서..(NLB를 쓰지 않고 애플리케이션 단위에서 보안을 한다는 건 위험한 선택이 아닐까요?) 찾아보았더니 다음과 같은 사례를 발견했습니다: [**How to restrict access to my game servers hosted on AWS**](https://www.reddit.com/r/aws/comments/ctxi34/how_to_restrict_access_to_my_game_servers_hosted/)

이 글의 작성자가 설명한 현재 본인의 서버는 각 세 대로, 유저 그룹에 따라 ip만 다르게 부여하고 있는 상황이었습니다.

```jsx
Group A → Instance A만 접근
Group B → Instance B만 접근
Group C → Instance C만 접근
```

이러한 구조로 바꾸고자 했다고 합니다.

1. Client → Reverse Proxy → EC2 (private subnet)

서버를 private으로 숨기는 구조를 먼저 고려해보았으나, 현재 사용하는 프로토콜인 photon(tcp/udp 바이너리 프로토콜이라고 합니다)는 http가 아니므로 웹 기반 서비스(cloudfront, alb 등)은 사용할 수 없었습니다.

이를 통해 알 수 있는 중요한 점은, L4에서는 유저 identity 기반 라우팅이 불가능하다는 것입니다. 이건 NLB의 특성때문에 발생하는데, NLB는 packet만 보고 전달하고 JWT 같은 세션/토큰 인증 데이터는 payload 안에 있습니다. 따라서 NLB는 이러한 인증 정보를 가지고 해석하지 않으므로 사용할 수 없습니다.

2. 게임 서버에서 인증/인가 처리

이 경우는 JWT 토큰을 서버 단에서 처리하는 것입니다(위에서 말했던). 유저가 로그인하고, 토큰을 발급받으면 이걸 게임 서버에서 확인하고 허용 여부를 판단하게 됩니다.

결론적으로 정리하자면, LB 단에서 인증 처리를 할 수 없는 이유는 다음과 같습니다.

- **프로토콜 문제**
- payload 접근 불가
- 실시간 특성상 LB에서 매번 검사하기 어려움

이외에도 분명 실무에서는 다른 보안을 하고 있겠지만(…그래야하지 않을까요?) 우선은 이 정도까지 알아보고 넘어가려고 합니다.

여기서 얻을 수 있는 건, 상황마다 다른 보안을 적용해야하고 영향을 주는 로직, 상황 등이 다양하다는 것입니다.

이를 모두 반영해서 보안 구조를 설계할 수 있어야 할 것입니다.

## 3.3 개선 권고사항

대신 게임 서버로 가는 트래픽을 보호하기 위해 다음과 같은 부분들을 고려해볼 수 있습니다.

- SG/NACL 사용: NLB에서 서버로 가는 포트만 열어놓기, 아웃바운드 최소화
- SSM 사용: ssh 사용하기 위해서는 또 포트를 열어둬야 함, 이 경우 키 관리 등의 문제가 생깁니다
<권장 방식>
    - 퍼블릭 IP 없이 운영
    - Bastion 대신 Systems Manager Session Manager 사용
    - IAM 기반으로 접속 통제 및 로그 남기기
- IAM Role 최소 권한: EC2가 접근할 수 있는 최소의, 필요한 권한만을 부여해서 해당 EC2가 탈취되었더라도 다른 리소스에는 접근이 불가능하도록 처리
- Secrets Manager/**SSM Parameter Store** 사용하여 시크릿 관리
- 게임 프로토콜 수준에서의 검증

### OAC (Origin Access Control)

CloudFront + S3 조합에 필요한 보안 처리를 간단히 정리하며 마무리하려고 합니다.

- CloudFront만 S3에 접근 가능하도록 만드는 방식
- CloudFront가 SigV4로 S3 요청을 서명
- 정적 리소스가 들어있는 S3 버킷의 퍼블릭 접근을 차단할 수 있음

특정 CloudFront distribution만 접근 허용하고, S3 public access 완전히 차단하는 것이 중요합니다.

### S3 Block Public Access

- 반드시 전체 활성화하고 실수로 퍼블릭 열리는 것 방지
- cloudfront를 사용해서 정적 컨텐츠를 서빙하기 위한 이유로 s3를 사용할 경우 퍼블릭 접근이 필요한 경우가 거의(아예) 없기 때문에 외부 접근은 기본적으로 막아두는게 좋습니다.

### 데이터 보호

- S3 Server-Side Encryption (SSE)
- KMS Key 정책도 최소 권한으로 부여
- Object-level 접근 제어 — s3는 특정 파일 경로, object 단위로 세세하게 접근 제한 가능


---

출처:

- [NPIXEL 고객 사례(2021)](https://aws.amazon.com/ko/solutions/case-studies/npixel/)
- [Stateful or Stateless? Choose the right approach for each of your game services](https://aws.amazon.com/ko/blogs/gametech/stateful-or-stateless/)
- [AWS 기반 게임 개발자를 위한 안내서 – 1부. DDoS 공격 방어 방법](https://aws.amazon.com/ko/blogs/korea/anti-ddos-for-game/)
