from https://aws.amazon.com/ko/solutions/case-studies/hitachi-global-life-solutions

<p align="center"><img src="https://velog.velcdn.com/images/alstnalstn2/post/4f18a696-dd1a-4c1e-8fbd-37d1bb1c8af5/image.png" width="40%"></p>

# 1. 개요

## 1.1 인프라 구성 요약
<p align="center"><img src="https://velog.velcdn.com/images/alstnalstn2/post/7fb26d82-3229-4700-9dae-746b279315cd/image.png" width="200%"></p>

### Usecase
- ;冷蔵庫カメラ : 냉장고 카메라
- 冷蔵庫のドアを開けた際に自動で撮影し、外出先などからスマートフォンアプリで冷蔵庫内の食材をチェックできる : 냉장고 문을 열면 자동으로 촬영하고, 외부에서도 스마트폰 앱으로 냉장고 안의 식재료를 확인할 수 있음
- 通信ログ : 통신 로그
- 家電運転ログ : 가전 운전 로그 또는 가전 동작 로그
- ファームウェア : 펌웨어
- デプロイ : 배포
- 社内業務システム : 사내 업무 시스템
- 開発者 : 개발자

Hitachi GLS는 2018년부터 스마트폰과 가전을 연결하는 홈 IoT 서비스를 클라우드 환경에서 운영해 왔습니다. 기존 플랫폼은 IaaS 형태였지만, 연결 기기 수가 늘어나면서 확장성이 문제가 되었고, 이에 따라 AWS 관리형 서비스를 활용한 새 IoT 플랫폼을 구축했습니다. 이 새 플랫폼은 최대 600만 대 연결을 목표로 설계되었고, 기존 사용자에게 영향을 주지 않으면서 이전이 진행되었습니다. 

새 플랫폼은 **AWS IoT Core, AWS Lambda, Amazon Kinesis, Amazon DynamoDB** 등을 사용하고, 애플리케이션은 컨테이너화하여 **Amazon ECS on AWS Fargate** 위에서 실행하도록 구성되었습니다. 또한 짧은 주기로 기능을 반영하기 위해 **CI/CD 파이프라인**도 함께 구축되었습니다.


---

# 2. 아키텍처 분석

## 2.1 핵심 구성 요소와 역할

### AWS IoT Core

냉장고, 세탁기, 에어컨 같은 기기들은 MQTT 또는 HTTP 프로토콜로 통신하는데, AWS IoT Core가 이 두 프로토콜을 기본으로 지원하기 때문에 기기 펌웨어 수정 없이 바로 연결할 수 있습니다.

IoT Core가 하는 일은 크게 두 가지라고 할 수 있습니다. 
- 하나는 디바이스 연결 관리로, 최대 220만 동시 접속(초당 20,000 패킷 기준으로 PoC 검증)을 처리하면서 각 기기의 연결 상태를 유지합니다. 
- 다른 하나는 메시지 라우팅으로, 기기에서 올라온 메시지를 규칙에 따라 Lambda, Kinesis, DynamoDB 같은 다음 처리 단계로 자동으로 전달합니다. 즉 IoT Core는 수백만 대 기기가 보내는 메시지를 받아서 백엔드 처리 파이프라인으로 넘기는 중계 역할을 담당합니다.

> #### - [MQTT](https://aws.amazon.com/ko/what-is/mqtt/)란? 
머신 대 머신 통신에 사용되는 표준 기반 메시징 프로토콜 또는 규칙 세트입니다.
스마트 센서, 웨어러블 및 기타 사물 인터넷(IoT) 디바이스는 일반적으로 리소스 제약이 있는 네트워크를 통해 제한된 대역폭으로 데이터를 전송하고 수신하기 위해 구현이 쉽고 IoT 데이터를 효율적으로 전달할 수 있는 MQTT를 사용합니다. MQTT는 디바이스에서 클라우드로, 클라우드에서 디바이스로의 메시징을 지원합니다.

### ECS on Fargate

ECS는 컨테이너 오케스트레이터(어떤 컨테이너를 몇 개 띄울지, 장애 발생 시 어떻게 복구할지를 관리하는 시스템)이고, Fargate는 컨테이너가 실제로 실행되는 인프라를 AWS가 대신 관리해주는 서버리스 컴퓨팅 엔진입니다.
본문에서는 기존 인프라에서는 패치 적용 때마다 새벽 12시~5시에 시스템을 중단했다고 언급하는데, Fargate로 전환한다면 이러한 작업을 줄일 수 있습니다. Fargate는 서버 없이 컨테이너 단위로 리소스를 할당하기 때문에, 가전기기 접속이 늘어날 때 컨테이너만 자동으로 늘어나는 구조를 사용할 수 있습니다. 

### Lambda + Kinesis + DynamoDB

Amazon Kinesis는 수백만 대 기기에서 실시간으로 쏟아지는 데이터 스트림을 받아서 순서를 보장하며 버퍼링하는 역할을 합니다. 가전기기는 온도, 작동 상태, 에러 코드 같은 데이터를 지속적으로 보내는데, 이를 곧바로 처리하려 하면 순간 트래픽 폭증 시 뒤에 있는 처리 시스템이 버티기 어렵습니다. Kinesis는 데이터를 스트림으로 받아 일정 속도로 다음 단계에 전달하는 버퍼 역할을 한다고 할 수 있습니다.

AWS Lambda는 Kinesis에서 데이터가 들어올 때마다 자동으로 트리거되어 실행되는 함수 단위 처리 서비스입니다. 서버를 상시 띄워둘 필요 없이, 이벤트가 발생했을 때만 실행되고 처리가 끝나면 종료됩니다. 기기 상태 변화 감지, 이상값 필터링, 데이터 가공 같은 작업을 함수 단위로 처리하는 데 있어서 효율적입니다.

Amazon DynamoDB는 Lambda가 처리한 결과나 기기의 현재 상태를 저장하는 NoSQL 데이터베이스입니다. RDB(관계형 DB)와 달리 스키마가 유연하고, 읽기/쓰기 처리량을 자동으로 확장할 수 있습니다. 수백만 대 기기의 상태를 동시에 읽고 쓰는 환경에서 RDB는 병목 현상이 생기기 쉬운 반면, DynamoDB는 큰 규모에서도 빠른 응답 속도인 밀리초 응답을 유지할 수 있습니다.



---

# 3. 보안 관점 분석

## 3.1 기기 인증

IoT 서비스에서 가장 먼저 봐야 할 부분은 기기가 정상 기기인지 확인하는 것입니다. 일반 웹서비스는 사용자가 로그인해서 서버에 접근하지만, 이 사례에서는 냉장고, 세탁기, 청소기 같은 가전기기가 클라우드에 직접 연결됩니다. 연결 대상이 사람이 아니라 기기이므로, 누가 로그인했는가보다 어떤 기기가 연결했는가가 먼저 확인되어야 합니다.

이 구조에서는 AWS IoT Core의 인증서 기반 인증을 사용할 수 있습니다. AWS IoT Core는 기기 연결 시 [X.509 인증서](https://learn.microsoft.com/ko-kr/azure/iot-hub/reference-x509-certificates)를 사용할 수 있고, 각 기기 또는 클라이언트에 고유 인증서를 부여하는 방식을 주로 사용합니다. 이렇게 하면 기기마다 고유한 신원을 가질 수 있고, 문제가 생긴 기기의 인증서만 비활성화하거나 교체하는 방식으로 대응할 수 있습니다.

히타치 사례에서는 새로 서비스를 0부터 시작하는 것이 아닌 이미 출하된 제품이 MQTT/HTTP 프로토콜로 동작하고, 제품 소프트웨어를 바꾸지 않는 호환성이 중요한 경우였습니다. 따라서 이 조건에서는 새 보안 기능을 마음대로 추가하기 어렵기 때문에, 클라우드에 연결되는 지점에서 기기 인증서, 인증서 만료, 인증서 폐기 상태를 관리하는 구조가 중요하다고 할 수 있습니다.

## 3.2 기기별 권한

기기가 정상적으로 인증되더라도, 모든 작업을 허용해서는 안되는데, 이를 위해 인증뿐만아니라 권한 제한 또한 중요하다고 할 수 있습니다. 

AWS IoT Core에서는 [IoT Policy](https://docs.aws.amazon.com/ko_kr/iot/latest/developerguide/iot-policies.html)를 통해 인증된 기기가 수행할 수 있는 작업을 제한할 수 있습니다. 즉, 이러한 작업 범위를 IoT Policy, Topic, Publish/Subscribe, Device Shadow를 통해 관리합니다.

- Topic
  - MQTT 메시지가 오가는 주소입니다. 예를 들어 냉장고 A가 자신의 상태를 클라우드로 보낸다면 refrigerator/A/status 같은 topic을 사용할 수 있습니다.
- Publish
  - 기기가 특정 topic으로 메시지를 보내는 행위입니다. 예를 들어 냉장고 A가 현재 온도나 문 열림 상태를 refrigerator/A/status로 보내는 것이 publish입니다.
- Subscribe
  - 기기가 특정 topic의 메시지를 받아보는 행위입니다. 예를 들어 냉장고 A가 클라우드에서 내려오는 제어 명령을 refrigerator/A/command에서 받아보는 것이 subscribe입니다.
- Device Shadow
  - 기기의 상태를 클라우드에 저장해 두는 JSON 문서입니다. 기기가 꺼져 있거나 일시적으로 연결되지 않아도, 앱이나 백엔드가 기기의 원하는 상태와 현재 상태를 저장하고 확인할 수 있습니다. 예를 들어 앱에서 냉장고 목표 온도를 3도로 설정하면, 이 값이 shadow의 원하는 상태에 저장되고 기기가 다시 연결되었을 때 반영될 수 있습니다.
- IoT Policy
  - 인증된 기기가 어떤 topic에 publish할 수 있는지, 어떤 topic을 subscribe할 수 있는지, 어떤 shadow에 접근할 수 있는지를 정하는 정책입니다.
  - 예를 들어 어떤 기기는 자기 상태값만 publish할 수 있고, 다른 기기의 topic을 읽거나 명령을 보낼 수 없도록 제한할 수 있습니다. AWS IoT Core 정책은 X.509 인증서, Cognito identity, thing group에 연결할 수 있고, 인증된 identity가 어떤 IoT 작업을 할 수 있는지 통제할 수 있습니다.

이 사례에 대입하면, 냉장고가 세탁기의 topic에 publish하거나, 특정 기기가 다른 기기의 shadow를 읽는 구조는 피해야 합니다. 기기별로 허용되는 topic, 명령, 데이터 범위를 좁혀야 합니다. 예를 들어서

> 냉장고 A → refrigerator/A/status 에만 publish
냉장고 A → refrigerator/A/command 만 subscribe
냉장고 A → washer/* 접근 불가

이런 식으로 기기별 topic 경계를 나누면, 한 기기의 인증서가 유출되더라도 다른 기기 데이터나 제어 명령까지 바로 확장되는 위험을 줄일 수 있습니다.

## 3.3 펌웨어 배포

IoT 서비스는 한 번 판매된 기기가 몇 년 동안 계속 사용됩니다. 시간이 지나면 취약점이 발견될 수 있고, 암호화 라이브러리나 통신 모듈도 갱신이 필요할 수 있습니다. 이러한 관점에서 펌웨어 업데이트는 필수라고 할 수 있는데, 펌웨어는 기기 동작을 바꾸는 코드이기 때문에, 공격자가 펌웨어를 바꿔치기 하는 등의 공격을 가하면 기기 자체가 공격자의 통제를 받을 수 있습니다. 따라서 펌웨어 배포에서는 단순히 파일을 내려보내는 것보다 무결성 검증과 배포 통제가 먼저 필요합니다.

AWS 관점에서는 [AWS IoT Jobs](https://docs.aws.amazon.com/iot/latest/developerguide/iot-jobs.html)와 [AWS IoT Device Management](https://aws.amazon.com/ko/iot-device-management/)를 활용해 펌웨어 업데이트 명령을 기기에 전달하고, 업데이트 상태를 추적할 수 있습니다. [AWS IoT Lens(IoT 워크로드를 설계할 때 참고할 수 있는 AWS 공식 아키텍처 가이드) 문서](https://docs.aws.amazon.com/wellarchitected/latest/iot-lens/firmware-updates.html)에서도 IoT Jobs를 통해 펌웨어 업데이트 명령과 상태를 MQTT 기반으로 주고받을 수 있고, Code Signer와 연동해 승인되지 않은 펌웨어 업데이트나 중간자 공격을 줄이는 데 도움을 줄 수 있다고 설명합니다.

펌웨어 업데이트 과정에서는 펌웨어 이미지 서명, 기기 업데이트, 업데이트 대상 관리, 업데이트 실패 시 처리 방식, 기기 별 펌웨어 버전 추적 등의 관리가 필요합니다.

---
### Ref

Hitachi GLS 사례, https://aws.amazon.com/solutions/case-studies/hitachi-global-life-solutions/
AWS IoT Core 개요, https://aws.amazon.com/documentation-overview/iot-core/
MQTT Topic 개념, https://docs.aws.amazon.com/iot/latest/developerguide/topics.html
AWS IoT Core 정책, https://docs.aws.amazon.com/iot/latest/developerguide/iot-policies.html
AWS IoT Core 클라이언트 인증, https://docs.aws.amazon.com/iot/latest/developerguide/client-authentication.html
X.509 클라이언트 인증서, https://docs.aws.amazon.com/iot/latest/developerguide/x509-client-certs.html
AWS IoT Device Shadow, https://docs.aws.amazon.com/iot/latest/developerguide/iot-device-shadows.html
펌웨어 업데이트 / 기기 관리,https://docs.aws.amazon.com/wellarchitected/latest/iot-lens/firmware-updates.html
AWS IoT Jobs, https://docs.aws.amazon.com/iot/latest/developerguide/iot-jobs.html
