참고 아키텍처: [드림 시큐리티](https://aws.amazon.com/ko/solutions/case-studies/dreamsecurity/)

# 1. 개요

## 1.1 인프라 구성 요약

이 인프라에 접근하는 몇 가지 시나리오가 있습니다.

1. ALB의 DNS 주소
2. public 서브넷에 위치한 OpenVPN EC2의 Elastic IP
3. CLB의 DNS 주소

여기서 일반 사용자의 경우 ALB의 DNS로 접근하게 될 것입니다.

## 1.2 분석 범위 및 목적

해당 글을 살펴보면, 당사의 문제는 다음과 같았습니다.

> 드림시큐리티는 물류 고객사의 요청으로 안면인식 소프트웨어인 'FGMS(Face Gate Mobile System)'를 개발해 제공했습니다. 그러나 현장에서 FGMS를 적용할 때 **유연성과 확장성을 고려하지 않아** 대규모 인력을 수용해야 사업장에서 운영 이슈가 발생했습니다. 한 터미널에서 문제가 발생할 경우, 모든 터미널로 문제가 확산되는 직렬 형태의 구조를 가지고 있어 4000명의 동시접속자를 수용할 수 없었습니다. **이 때문에 사용이 급증하는 퇴근 시간대에 서버가 다운되는 이슈가 자주 발생**했습니다. 과금 체계의 확장성 부재로 애드온 제품을 추가할 수 없었고, 사용자 및 관리자를 대상으로 한 대한 교육을 제공하지 않아 **사용자 편의성 개선이 필요**했습니다.

이러한 요구사항을 바탕으로, 어떻게 인프라를 구축하였는지 분석해보고자 합니다. 간단히 요약하면 다음과 같습니다.

1. **시스템 아키텍처의 확장성 부족**
FGMS가 대규모 인원을 수용하도록 설계되지 않아 4000명 동시 접속 환경에서 안정적으로 운영되지 못하는 문제가 발생했습니다.

2. **구조적 장애 확산 문제 (직렬 구조)**
한 터미널에서 문제가 발생하면 다른 터미널로 장애가 전파되는 구조로 인해 전체 시스템 안정성이 저하됩니다.

3. **트래픽 증가 시간대의 서버 안정성 문제**
퇴근 시간 등 사용자 접속이 급증하는 시간대에 서버 다운이 빈번하게 발생합니다.

4. **과금 및 서비스 확장 모델 부재**
확장 가능한 과금 체계가 없어 애드온 제품이나 추가 기능을 유연하게 도입하기 어렵습니다.

5. **사용자 교육 및 운영 지원 부족**
사용자와 관리자를 대상으로 한 교육이 제공되지 않아 시스템 활용도와 사용자 편의성이 낮습니다.

AWS에서 제공하고 있는 아키텍처는 어떻게 위의 사항들을 반영하였는지를 기준으로 살펴보겠습니다.


# 2. 아키텍처 분석

## 2.1 전체 구성도

![](https://velog.velcdn.com/images/hvvup/post/f354ba6a-4372-4592-bda5-1fc60f8dbda6/image.png)

## 2.2 구성 요소별 역할

이 구조에서 특이한 점이 ALB와 CLB가 혼용되고 있다는 것입니다.

CLB와 ELB의 정확한 차이는 [Differences: ALB vs. NLB vs. CLB](https://nidhiashtikar.medium.com/differences-alb-vs-nlb-vs-clb-29f25fc1033b) 이 문서를 참고해보면 좋습니다.

정리하자면, CLB는 다른 로드밸런서와 달리 좀 더 폭넓은 계층에서 동작한다는 점이 있습니다. ALB는 L7, NLB는 L4만 본다면 CLB는 L4를 주로 보지만 L7에서도 어느정도 동작합니다. NLB에 ALB 기능 약간을 추가했다고 보면 될 것 같습니다.

이 아키텍처에서 각각의 로드밸런서는 다음과 같은 상황에 쓰였습니다.

- CLB: Public Subnet → Private Subnet 구간에서 hrface_3rdparty-a, hrface_3rdparty-c 등 3rd Party 연동 서비스 앞단
- ALB: 메인 faceone enterprise business 트래픽 처리

<br>

왜 여기서 CLB를 선택했을지 몇 가지 예상해보면 다음과 같습니다.

1. **Layer 4 (TCP) 레벨 처리 필요**

   - hrface 시리즈 서비스들이 HTTP 외의 프로토콜(TCP, SSL Passthrough)을 사용할 가능성이 있습니다.
   - CLB는 L4/L7 모두 지원하며, 특히 단순 TCP 포트 포워딩에 적합합니다.

2. **3rd Party 연동 특수성**

   - 외부 제휴사(3rd party) 시스템과의 연동 시 고정 IP나 특정 포트가 필요한 경우에 CLB는 고정된 단순 구조로 외부 연동 인터페이스 관리가 쉽습니다.
   - 반면 ALB는 기본적으로 확장성 및 유지를 위해 dynamic IP 사용으로 설계되어 있습니다.
   (다음 글을 참고해보시기 바랍니다! [Application Load Balancer IP Change Event](https://repost.aws/questions/QUyjryn7t7SOOhYQDtpfR2pg/application-load-balancer-ip-change-event))

3. **레거시/단계적 마이그레이션**

   - 초기 구축 시 CLB로 시작 후 ALB로 완전 전환을 못한 시점의 구조일 가능성도 존재할 것 같습니다.
   - 실제로 AWS도 CLB → ALB/NLB 마이그레이션을 권장하고 있습니다 ([Migrate your Classic Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/migrate-classic-load-balancer.html))
   
   > Classic Load Balancers are the previous generation of load balancers from Elastic Load Balancing. We recommend that you migrate to a current generation load balancer. [출처](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/introduction.html)

4. **보안 아키텍처 관점**

   - OpenVPN + CLB 조합으로 VPN을 통한 내부 트래픽과 일반 퍼블릭 트래픽을 분리합니다.
   - CLB에서 SSL termination을 처리하고 내부는 평문 통신합니다.
   
   이 부분은 []() 이 게시글에서 더 자세히 알아봅니다. (추후 추가 예정)
   
그런데 아마도 CLB를 사용한 것은 요구사항 상 어쩔 수 없는 선택이었을 것 같습니다. AWS에서 CLB의 대체를 권고하는 데 반해 CLB의 기능과 ELB의 기능이 상당히 다릅니다.

아무래도 ELB는 각각의 로드밸런서 기능이 L7과 L4로 분리되어 있다보니 CLB는 그 둘의 교집합 정도로, CLB가 적합한 환경이 따로 있을 것 같습니다.


# 3. 보안 관점 분석

## 3.1 현재 보안 구성 현황

### VPC Endpoint 활용

ssm, s3, rds와 같은 서비스에 vpc endpoint를 통해서 접근하도록 설계되어 있습니다.

이전에는 막연히 '그래, 내부망이 당연히 안전하겠지~'하고 말았는데 생각해보니 만약 엔드포인트를 사용하지 않는다면 내부 리소스임에도 불구하고 `ec2 -> igw -> s3`처럼 접근할테니 외부망을 타게 되고 보안상 취약해질 수밖에 없습니다!

endpoint는 gateway 방식과 interface 방식이 있는데, gateway는 s3와 dynamodb만 지원하고 있습니다. 라우팅 테이블에 s3.amazonaws.com은 우선적으로 라우팅하도록 라우팅 테이블에 추가하는 방식입니다. interface는 eni를 생성해서 트래픽을 가져옵니다.

여기서는 어떤 방식을 사용해서 구현했는지는 알 수 없지만, 어쨌든 외부망 접근을 최소화했다고 볼 수 있습니다.

### 보안 모니터링

Config와 GuardDuty를 CloudWatch로 연동해 metric 필터링을 하고, 이를 SNS로 퍼블리시하는 파이프라인이 나와있습니다.

이는 가장 간단한 방식으로 AWS에서 위협 알림을 자동화하는 방법 같습니다. 이러한 방식이 여러 아키텍처에서 사용됩니다. CloudTrail이나 VPC flow logs같은 것도 연동해 사용할 수 있겠지만 여기서는 Config(보안 오구성 탐지)와 GuardDuty(로그를 바탕으로 침해사고/위협 탐지 및 식별)을 중점적으로 구현해두었습니다.

### WAF

WAF는 ALB 앞단에 붙어 트래픽을 검사하게 될 것입니다.

그런데 이 구조가 익숙하지 않아서 뭔가 했더니 **이 구조에서는 바로 ALB의 DNS 주소로 사용자가 접근**하게 됩니다. 지금까지는 cloudfront를 거쳐서 진입하는 구조를 보통 봤었습니다.

## 3.2 식별된 취약점

### WAF 단독 구성 시 DDoS 위험

ALB의 ip 주소를 공격자가 바로 알 수 있게 되고 이렇게 되면 단일 엔드포인트로 접근이 몰리다보니 ddos 공격의 위험성이 커질 것입니다. cloudfront의 경우 엣지 로케이션으로 트래픽을 분산하고 분산된 트래픽 이후의 요청을 alb에 넘기는 방식이므로 확실히 두 방식에 따라 alb에 가는 부담이 다릅니다.

### OpenVPN 인스턴스 보안

여기서는 VPN 접속을 위해 OpenVPN을 띄워놓은 인스턴스가 있습니다.

[OpenVPN](https://github.com/openvpn/openvpn)은 오픈소스 vpn 데몬이라고 합니다. 그렇다는 건 이 인스턴스에 대한 보안, 오픈소스로 구축한 서비스에 대한 보안이 또 추가된다는 것입니다.

이렇게 오픈소스를, 특히 vpn을 회사에서 자체 구축하기 위해 보안적으로 신경써야할 점이 무엇이 있는지 간단히 알아보려고 합니다.

 1. OpenVPN 서버 자체 취약점 관리

OpenVPN에서 발견된 취약점을 패치를 통해 관리하는 것이 무엇보다 중요할 것입니다. 다시 보니 SSM Patch Manager이 보입니다. 이를 통해서 패치 자동화를 했을 것으로 보입니다. 새로운 버전이 나오면 업그레이드하고 보안 취약점이 있는 버전을 쓰지 않도록 하는게 중요합니다.

2. 인증 체계에 대한 고민

기본 OpenVPN 인증 방식은 인증서(PKI) 기반과 ID/PW 기반, 그리고 둘을 혼용하는 방식이 있습니다.

여기서 발생하는 문제는
- 인증서 탈취 시 VPN 무단 접속 가능
- ID/PW 단독으로 사용하면 크리덴셜 스터핑 공격에 취약
- 인증서 만료 관리 누락 시 갑작스러운 접속 불가

등등이 생길 수 있습니다.

3. OpenVPN EC2 인스턴스 자체 보안

아키텍처에 그려져 있듯이 vpn 인스턴스가 public 서브넷에 위치하고, 이는 외부 접근이 가능하다는 뜻입니다. EIP가 노출되면 이 인스턴스 자체가 공격 포인트가 될 수 있습니다.

또한 이 서버의 권한이 탈취될 경우 인스턴스에 붙은 role로 피해가 확산되거나 잘못된 접근이 생기는 등의 문제가 생길 수 있습니다.

4. PKI 인증서 관리

여기서도 CA 키 보안, 인증서 만료 및 CRL(인증서 폐기 목록) 관리와 같은 포인트가 생깁니다.

5. 로깅 및 모니터링

AWS에서 제공하는 것은 기본적으로 ec2 인스턴스 자체에 대한 로그와 vpc flow 로그 정도가 있을 것입니다. 여기서 OpenVPN은 따로 모아 alert할 수 있도록 프로세스 구현이 필요합니다.

## 3.3 개선 권고사항

따라서 이 구조에서는 Shield Advanced를 사용하거나 CloudFront를 사용하는 등 트래픽을 분산시키고 DDoS에 대응할 수 있도록 고려하는 것이 필요해 보입니다.

OpenVPN 인스턴스에 대한 인증 체계 개선으로 MFA 사용, 인증서 만료 관리 자동화 등의 고려가 필요합니다.

EC2 인스턴스 보안을 위해 서브넷 분리, security group 설정, iam 권한 분리 등에 대한 고민도 필요합니다.

여기서 보안적으로는 IP를 고정 및 DDoS/트래픽 완화를 위해 NLB를 배치하는 것을 고려해볼 수 있을 것 같습니다.
