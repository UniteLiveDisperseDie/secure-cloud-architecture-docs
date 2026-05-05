## 1. 개요

### 1.1 사고 배경
<figure align="center">
  <img src="https://velog.velcdn.com/images/alstnalstn2/post/7fd0ca9a-d847-41f6-9a24-a762f727b631/image.png" alt="SAS Token 노출 흐름" />
  <figcaption>SAS Token 권한 오설정으로 인한 데이터 노출</figcaption>
</figure>

Microsoft AI 연구팀은 이미지 인식용 오픈소스 코드와 AI 모델을 GitHub 저장소를 `robust-models-transfer`라는 이름을 통해 공개하고 있었습니다. 사용자는 GitHub에 있는 가이드에 따라 Azure Storage URL에서 모델 파일을 다운로드하도록 되어 있었는데, MS 연구팀은 오픈소스 AI 모델과 학습 데이터를 외부 연구자와 개발자에게 공유하기 위해 Azure Storage를 사용하고 있었습니다. 

이 과정에서 Microsoft AI 연구팀은 Azure Storage의 **SAS token**을 사용했습니다. 
> [**SAS token**](https://learn.microsoft.com/ko-kr/azure/storage/common/storage-sas-overview)은 Azure Storage 데이터에 접근할 수 있도록 해주는 서명된 URL으로, 사용자가 접근 범위, 권한, 만료 시간을 직접 설정할 수 있습니다. 접근 범위는 단일 파일, 컨테이너, 전체 스토리지 계정 단위로 지정할 수 있고, 권한도 읽기 전용부터 전체 제어 권한까지 설정할 수 있습니다.
(AWS에서는 [S3 presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)과 유사하다고 할 수 있습니다.)

- 이 사건의 가장 큰 문제점은 연구팀이 공개하려던 AI 모델 파일만 공유한 것이 아니라, SAS token이 **전체 storage account에 대한 접근 권한**을 부여하도록 잘못 설정되어 있었다는 점입니다. 
- 이로 인해 원래 공개 대상이 아니었던 약 **38TB의 추가 내부 데이터**가 함께 노출되었습니다. 노출된 데이터에는 Microsoft 직원 2명의 워크스테이션 디스크 백업, 비밀번호, secret key, private key, 그리고 30,000개 이상의 내부 Microsoft Teams 메시지가 포함되어 있었습니다. 

### 1.2 사고 요약

이 사건은 AI 연구 데이터를 외부에 공유하는 과정에서 **과도한 권한이 부여된 SAS token을 GitHub에 공개한 결과** 발생한 데이터 노출 사고입니다. Microsoft AI 연구팀은 오픈소스 모델 파일을 공유하기 위해 Azure Storage URL을 GitHub 저장소에 게시했지만, 해당 URL에 포함된 SAS token은 특정 모델 파일만이 아니라 전체 storage account에 접근할 수 있도록 설정되어 있었습니다.

<figure align="center">
  <img src="https://velog.velcdn.com/images/alstnalstn2/post/b29037c7-34f0-4f11-bc8d-4f0755af5ba2/image.png" alt="SAS Token 노출 흐름" />
  <figcaption>'robustnessws4285631339' storage 계정 내의 노출된 컨테이너들</figcaption>
</figure>

또한 해당 token은 읽기 전용 권한이 아니라 **full control 권한**을 가지고 있었습니다. 따라서 외부 사용자는 파일을 조회하는 것뿐만 아니라, 기존 파일을 삭제하거나 덮어쓸 수 있는 권한까지 가질 수 있었습니다. Wiz는 이 점 때문에 공격자가 AI 모델 파일에 악성 코드를 삽입할 경우, Microsoft GitHub 저장소를 신뢰하고 모델을 내려받는 사용자가 영향을 받을 수 있는 공급망 공격 가능성도 있었습니다.

- [Wiz 자료](https://www.wiz.io/blog/38-terabytes-of-private-data-accidentally-exposed-by-microsoft-ai-researchers)에 따르면 해당 SAS token은 2020년 7월 20일 GitHub에 처음 커밋되었고, 2021년 10월 6일 만료 시간이 2051년 10월 6일까지로 연장되었습니다. 
- Wiz Research는 2023년 6월 22일 문제를 발견해 MSRC에 제보했고, Microsoft는 2023년 6월 24일 해당 SAS token을 무효화했습니다. 이후에 2023년 9월 18일 사고가 발표되었습니다. 

---

## 2. 노출 경로 및 위험 분석 

### 2.1 전체 흐름

이 사건의 흐름을 정리해보면 다음과 같습니다. 

1. GitHub 저장소에 Azure Storage URL 공개
2. URL 안에 SAS token 포함
3. SAS token이 전체 storage account 범위로 설정됨
4. read-only가 아니라 full control 권한까지 부여됨
5. 공개 대상이 아닌 내부 데이터까지 접근 가능
6. 데이터 조회, 삭제, 덮어쓰기, 모델 변조 가능성 발생

이후 Wiz Research가 인터넷에 노출된 클라우드 저장소를 조사하던 중 해당 문제를 발견하고 Microsoft에 제보한 후 Microsoft가 2023년 6월 24일 해당 SAS token을 무효화하고 외부 접근을 차단했습니다.  

### 2.2 발생 가능 위험
![](https://velog.velcdn.com/images/alstnalstn2/post/15c144cb-ebbf-4b91-adb4-0878a839a06e/image.png)


#### 권한 측면

SAS token은 읽기, 목록 조회, 쓰기, 삭제와 같은 권한을 부여할 수 있습니다. 또한 접근 범위도 특정 파일 하나에만 제한할 수도 있고, 컨테이너 또는 전체 storage account 수준까지 넓힐 수도 있습니다. 따라서 잘못 설정된 SAS token은 단순 다운로드 링크가 아니라, storage account에 대한 높은 수준의 접근 권한을 부여할 수 있습니다.

이 사건에서도 공개 목적은 AI 모델 파일을 다운로드하게 하는 것이었지만, 실제 token은 전체 storage account 범위에 적용되었고 full control 권한까지 포함하고 있었습니다. 이 경우 외부 사용자는 의도한 모델 파일뿐만 아니라 같은 storage account 안의 다른 컨테이너와 데이터에도 접근할 수 있습니다. 또한 read-only가 아니라 쓰기와 삭제 권한까지 포함된 경우, 파일을 단순히 보는 것을 넘어 덮어쓰기나 삭제까지 가능해집니다.

이런 설정은 기밀성 침해뿐 아니라 무결성 침해로도 이어질 수 있습니다. 특히 AI 모델 파일처럼 외부 사용자가 신뢰하고 내려받는 데이터가 변조되면, 모델을 사용하는 연구자나 개발자에게 악성 파일이 연쇄적으로 전달되는 공급망 공격이 발생할 수 있습니다.

#### 토큰 수명 관리 측면

SAS token은 만료 시간 설정이 중요합니다. 하지만 SAS token에는 만료 시간 상한이 없기 때문에, 사용자가 매우 긴 유효기간을 설정할 수 있습니다. Wiz 자료에 따르면 많은 조직이 유효기간이 매우 긴 token을 사용하며, Microsoft의 token도 2051년까지 유효하도록 설정되어 있었습니다.

이처럼 장기 유효 token이 공개 GitHub 저장소나 문서, 웹사이트에 포함되면 노출 기간이 길어집니다. 특히 공개 저장소에 포함된 URL은 검색 엔진, 보안 연구자, 공격자에게 발견될 수 있으므로, token의 유효기간이 길수록 악용 가능성도 커집니다.

#### 관리 및 모니터링 측면

Account SAS token은 Azure가 중앙에서 발급 목록을 관리하는 객체가 아니라, storage account key를 이용해 클라이언트 측에서 생성되는 서명된 URL입니다. 따라서 token 생성 자체가 Azure 이벤트로 남지 않고, 생성된 token도 Azure 리소스 객체로 등록되어 모니터링하기는 어렵습니다.
따라서 이 구조에서는 storage account가 private으로 보이더라도 실제로는 외부에 배포된 SAS token을 통해 접근이 가능할 수 있습니다. 즉, 관리자가 스토리지 계정이 public이 아니라고 확인하더라도, 이미 외부에 유통된 token이 있으면 데이터가 노출될 수 있습니다.

---

## 3. 대응 방안

### 3.1 즉각 대응 절차

SAS token 노출이 확인되면 가장 먼저 해야 할 일은 추가 접근을 차단하는 것입니다. 이 사건처럼 Account SAS token이 공개된 경우, 개별 token 하나만 선택적으로 폐기하기 어렵습니다. 따라서 해당 token을 서명하는 데 사용된 storage account key를 회전하여 기존 token을 무효화해야 합니다. 다만 같은 key로 발급된 다른 SAS token도 함께 무효화될 수 있으므로, 정상 서비스에 미칠 영향도 동시에 고려해야 합니다.

그다음에는 노출된 URL이 어떤 권한을 가지고 있었는지 확인해야 합니다. 단순 read 권한이었는지, list, write, delete 권한까지 포함되어 있었는지에 따라 사고 범위 및 대처가 달라질 수 있습니다. 특히나 해당 사건처럼 full control 권한이 포함된 경우에는 단순 데이터 열람뿐 아니라 파일 삭제, 덮어쓰기, 모델 파일 변조 가능성까지 함께 조사해야 합니다.

이후에는 영향을 받은 storage account의 데이터를 전수 조사해야 합니다. 공개 대상이 아니었던 내부 데이터가 같은 storage account 안에 있었기 때문에, 어떤 컨테이너와 파일이 token 권한 범위 안에 있었는지 확인해서 비밀번호, secret key, private key, 내부 메시지처럼 다른 시스템 보안에 영향을 줄 수 있는 데이터가 포함되어 있었다면, 해당 값들은 더 이상 신뢰하지 못하고 즉시 교체 및 폐기해야합니다.

### 3.2 사후 조치 및 재발 방지

#### 외부 공유용 저장소 분리

먼저, 외부에 공개할 AI 모델이나 데이터셋은 내부 데이터와 같은 storage account에 두지 않아야 합니다. 만약 공개 AI 데이터셋을 전용 storage account로 분리했다면 과도한 token이 발급되더라도 노출 범위를 공개 데이터로 제한할 수 있었을 것입니다. 

#### Account SAS 사용 제한

AWS에서 외부에 파일을 공유할 때는 보통 **S3 Presigned URL**을 사용합니다. Presigned URL은 IAM 권한을 가진 주체가 특정 S3 객체에 대해 임시 접근 URL을 생성하는 방식입니다. 

반면 Azure의 **Account SAS**는 Storage Account Key를 기반으로 생성되는 SAS token입니다. 설정에 따라 단일 파일이 아니라 컨테이너 또는 전체 storage account 범위까지 접근 권한을 줄 수 있고, 읽기뿐만 아니라 쓰기, 삭제 권한까지 포함할 수 있습니다. 따라서 Account SAS가 과도한 범위와 권한으로 생성되면, AWS 기준으로는 **강한 권한을 가진 장기 공유 URL을 외부에 배포한 것과 비슷한 위험**이 발생한 것과 마찬가지라고 할 수 있습니다.
따라서 외부 공유 목적이라면 Account SAS 사용을 피하고, 아래에 설명되어 있는 Service SAS나 User Delegation SAS 등의 조금 더 제한적인 SAS 방식을 선택하는 것이 안전합니다.

> **[Service SAS](https://learn.microsoft.com/en-us/rest/api/storageservices/service-sas-examples)란?**  
> Service SAS는 Azure Storage의 특정 서비스 리소스에 대해 접근 권한을 부여하는 SAS token입니다. 예를 들어 Blob Storage의 특정 컨테이너나 특정 Blob (aws로 따지면 s3 object)에 대해서만 권한을 줄 수 있습니다. Stored Access Policy와 연결하면 서버 측 정책을 통해 만료 시간이나 권한을 더 중앙에서 관리할 수 있어, Account SAS보다 통제하기 쉽습니다.

> **[User Delegation SAS](https://learn.microsoft.com/ko-kr/rest/api/storageservices/create-user-delegation-sas)란?**  
> User Delegation SAS는 Storage Account Key가 아니라 Azure Active Directory 자격 증명을 기반으로 생성되는 SAS token입니다. 만료 시간이 최대 7일로 제한되며, token 생성 주체를 Azure AD 기반으로 통제할 수 있습니다. 따라서 외부 공유가 필요한 경우 Account SAS보다 User Delegation SAS를 사용하는 편이 관리와 추적 측면에서 더 안전합니다.

정리하자면 AWS에서 특정 객체를 짧은 시간 공유할 때 Presigned URL을 사용하는 것처럼 Azure에서도 외부 공유 권한은 가능한 한 좁은 범위와 짧은 시간으로 제한해야 합니다. Service SAS 또는 User Delegation SAS처럼 범위와 수명을 더 제한할 수 있는 방식을 Account SAS와 같은 위험한 방식보다 우선적으로 고려해야 합니다.

#### 권한 범위와 만료 시간 최소화

그리고 SAS token을 사용할 수밖에 없는 경우에는 접근 범위와 권한, 만료 시간을 최소화해야 합니다. 공개 모델 파일 다운로드가 목적이라면 전체 storage account가 아니라 특정 파일 또는 특정 컨테이너에 대한 읽기 권한만 부여해야 합니다. 또한 full control 권한은 외부 공유 목적에 부적절합니다. 

만료 시간도 짧게 설정해야 합니다. 이 사건의 token은 2051년까지 유효하도록 설정되어 있었던 것과 같이, SAS token에는 만료 시간 상한이 없기 때문에 장기 또는 무기한 token이 만들어질 수 있습니다. 따라서 외부 공유 token은 필요한 기간이 지나면 자동으로 만료되도록 구성해야 합니다. 

#### 지속적 탐지 통제 구축

SAS token은 중앙에서 발급 목록을 관리하기 어렵기 때문에, 생성 이후의 지속적인 탐지 체계가 필요합니다. 특히 앞서 언급했듯 Account SAS token은 Azure 리소스 객체로 등록되지 않으므로, 관리자가 포털에서 현재 발급된 token 목록을 한 번에 확인하기 어렵습니다. 따라서 storage account가 private으로 설정되어 있더라도, 외부에 공개된 SAS token이 있다면 실제로는 접근 가능한 상태가 될 수 있습니다.

이를 보완하기 위해 storage account별 로그와 메트릭을 활성화해야 합니다. [Azure Metrics](https://learn.microsoft.com/ko-kr/azure/azure-monitor/reference/supported-metrics/microsoft-classicstorage-storageaccounts-metrics)를 사용하면 SAS 인증 요청이 발생한 storage account를 확인할 수 있고, [Storage Analytics logs](https://learn.microsoft.com/ko-kr/azure/storage/common/storage-analytics-logging)를 활성화하면 SAS token을 이용한 접근 기록을 더 구체적으로 분석할 수 있습니다. 평소 사용하지 않던 storage account에서 SAS 요청이 발생하거나, 비정상적으로 많은 요청이 발생하면 이상 징후로 탐지해야 합니다.

또한 GitHub 저장소, 문서, 웹사이트, 모바일 앱, 빌드 산출물 등 외부에 노출될 수 있는 자산에 대해 secret scanning을 수행해야 합니다. 공개 저장소에 장기 SAS token이 포함되면 검색 엔진이나 공격자에게 발견될 수 있으므로, 커밋 전 단계와 배포 전 단계에서 github secrets scanning 등의 자동 검사를 수행하고, token이 발견되면 병합이나 배포를 차단해야 합니다.

---

## ref

[https://www.wiz.io/blog/38-terabytes-of-private-data-accidentally-exposed-by-microsoft-ai-researchers](https://www.wiz.io/blog/38-terabytes-of-private-data-accidentally-exposed-by-microsoft-ai-researchers)
[https://www.microsoft.com/msrc/blog/2023/09/microsoft-mitigated-exposure-of-internal-information-in-a-storage-account-due-to-overly-permissive-sas-token](https://www.microsoft.com/msrc/blog/2023/09/microsoft-mitigated-exposure-of-internal-information-in-a-storage-account-due-to-overly-permissive-sas-token)
