
# AWS Community AMI 악성코드 삽입 


## 1. 개요

### 1.1 사고 배경

AWS는 EC2 인스턴스를 실행하기 위한 기반 이미지인 AMI(Amazon Machine Image)를 여러 유형으로 제공하고 있습니다. 그 중 **Community AMI**는 불특정 사용자가 자유롭게 업로드하고 공유할 수 있는 이미지로, 별도의 신원 검증이나 보안 심사 없이 누구든 게시할 수 있습니다.

| 유형 | 제공 주체 | 신뢰 수준 |
|------|-----------|-----------|
| AWS 소유 AMI | Amazon 직접 관리 | 높음 |
| 마켓플레이스 AMI | 검증된 파트너 벤더 | 높음 |
| Community AMI | 불특정 커뮤니티 사용자 | 낮음 |
| Private AMI | 조직 내부 또는 외주 개발자 | 조직 의존 |

이러한 개방성은 다양한 이미지를 손쉽게 활용할 수 있다는 장점이 있지만, 동시에 악의적인 행위자가 악성코드를 사전 삽입한 이미지를 배포하는 공격 벡터로 악용될 수 있다는 심각한 위험을 내포하고 있습니다.

### 1.2 사고 요약

2020년, 보안 사고 대응 전문 기업 **Mitiga**의 연구팀은 한 금융기관의 인시던트를 조사하던 중 AWS Community AMI에 **Monero(XMR) 크립토마이너**가 사전 삽입되어 있던 사실을 발견했습니다.

| 항목 | 내용 |
|------|------|
| 피해 기관 | 금융기관 (익명) |
| 발견 경위 | Mitiga 연구팀의 인시던트 조사 |
| 악성 AMI | AWS Community AMI `Microsoft Windows Server 2008` |
| 악성코드 종류 | Monero(XMR) 크립토마이너 |
| 피해 지속 기간 | 약 **5년** (미탐지 상태로 운영) |

피해 기관은 이미 해당 AMI를 EC2에서 제거하여 더 이상 사용하지 않고 있다고 밝혔지만, 채굴기는 제거 후에도 여전히 시스템 내에 잔존하고 있었습니다. 이는 단순히 AMI를 교체하는 것만으로는 이미 감염된 인스턴스를 치료할 수 없음을 시사합니다.

---

## 2. 공격 분석

### 2.1 공격 흐름 (Attack Flow)

이 사고의 전체 흐름은 **공급망 공격(Supply Chain Attack)** 의 전형적인 패턴을 따릅니다. 공격자는 피해 기관의 인프라를 직접 침투하는 대신, 피해 기관이 신뢰하고 사용하는 자원 자체에 악성코드를 심어두는 방식을 택했습니다.

```
[공격자]
    │
    │ 악성 AMI 업로드
    ▼
[AWS Community AMI 마켓플레이스]
    │
    │ 피해 기관이 검증 없이 사용
    ▼
[EC2 인스턴스 생성]
    │
    │ 부팅 시 크립토마이너 자동 실행
    ▼
[Monero 채굴 → 공격자 지갑으로 전송]
    │
    │ 5년간 미탐지
    ▼
[Mitiga 인시던트 조사 중 발견]
```

### 2.2 단계별 공격 프로세스

**① 악성 AMI 제작 및 업로드**

공격자는 정상적인 `Microsoft Windows Server 2008` 이미지를 기반으로, 부팅 시 자동으로 실행되는 크립토마이너를 내부에 삽입했습니다. AWS Community AMI는 게시자 신원 검증, 자동 악성코드 스캔, 사용자 리뷰 등 어떠한 안전장치도 제공하지 않으므로, 악성 AMI는 정상 이미지와 외형상 구별이 거의 불가능한 상태로 마켓플레이스에 등록될 수 있었습니다.

다른 공개 플랫폼과 비교하면 그 차이가 명확합니다.

| 플랫폼 | 콘텐츠 유형 | 안전장치 |
|--------|-------------|----------|
| GitHub | 오픈소스 코드 | 자동 취약점 스캔, 신고 시스템, 평판 기반 지표 |
| Docker Hub | 컨테이너 이미지 | Official Image 인증, 스캔 결과 공개, 다운로드 수·평점 |
| PyPI / npm | 패키지 | 악성코드 자동 탐지, 신고 시스템, 다운로드 통계 |
| AWS Community AMI | VM 이미지 | **없음** |

**② EC2 인스턴스 생성 및 악성코드 실행**

피해 기관은 해당 AMI를 의심 없이 EC2 인스턴스 생성에 사용했습니다. 인스턴스가 부팅되는 순간 사전 삽입된 크립토마이너가 자동으로 실행되었으며, 채굴된 Monero는 공격자의 지갑으로 지속적으로 전송되었습니다.

**③ 5년간의 미탐지 운영**

크립토마이너는 CPU·네트워크 자원을 소모하지만, 모니터링 체계가 갖춰져 있지 않았기 때문에 5년이라는 긴 기간 동안 탐지되지 않았습니다. 크립토마이너는 공격 결과물 중 비교적 가벼운 피해에 속하지만, 공격자가 의도에 따라 아래와 같은 훨씬 심각한 공격도 가능했습니다.

- **백도어 설치** → EC2 인프라 전체 접근 및 Lateral Movement
- **랜섬웨어 지연 트리거** → 특정 시점에 데이터 암호화 및 협박
- **자격증명 탈취** → IAM 키, DB 패스워드 등 민감 정보 수집
- **내부 침투** → EC2를 거점으로 VPC 내 다른 리소스로 확산

**④ 사고 발견 및 AMI 제거 후에도 잔존**

Mitiga의 인시던트 조사를 통해 악성 AMI가 발견되었고, 피해 기관은 해당 AMI를 제거했습니다. 그러나 이미 실행 중이던 인스턴스 내에는 악성코드가 여전히 남아 있었으며, 이는 AMI 교체만으로 감염 인스턴스를 치료할 수 없다는 중요한 교훈을 남겼습니다.

---

## 3. 대응 방안

### 3.1 즉각 대응 절차

악성 AMI로 생성된 인스턴스가 식별된 경우, 다음 순서로 대응을 진행해야 합니다.

**① 격리(Isolation)**

감염된 인스턴스를 즉시 네트워크에서 격리하여 추가적인 채굴 통신 및 Lateral Movement를 차단합니다.

```bash
# Security Group을 통해 모든 인바운드/아웃바운드 트래픽 차단
aws ec2 modify-instance-attribute \
  --instance-id <instance-id> \
  --groups <isolation-sg-id>
```

**② 스냅샷 보존**

포렌식 분석을 위해 인스턴스 종료 전 EBS 볼륨의 스냅샷을 반드시 보존합니다.

```bash
aws ec2 create-snapshot \
  --volume-id <volume-id> \
  --description "Forensic snapshot - malicious AMI incident"
```

**③ 영향 범위 파악**

동일한 AMI로 생성된 모든 인스턴스를 식별하고, 해당 인스턴스가 접근했던 IAM 자격증명, S3 버킷, RDS 등 연관 리소스를 목록화합니다.

```bash
# 특정 AMI로 실행된 인스턴스 조회
aws ec2 describe-instances \
  --filters "Name=image-id,Values=<malicious-ami-id>"
```

**④ 자격증명 교체**

감염 인스턴스에 연결된 IAM Role, 액세스 키, DB 패스워드 등 모든 자격증명을 즉시 교체하고, 기존 세션을 무효화합니다.

**⑤ 인스턴스 종료 및 재생성**

감염 인스턴스는 치료하지 않고 종료 후 검증된 AMI로 새롭게 생성하는 것을 원칙으로 합니다.

### 3.2 사후 조치 및 재발 방지

**① AMI 사용 정책 강화 (SCP 적용)**

SCP(Service Control Policy)를 활용하여 조직 내에서 허용된 AMI 소유자의 이미지만 사용할 수 있도록 강제합니다. 이를 통해 Community AMI 사용 자체를 정책 레벨에서 차단할 수 있습니다.

```json
{
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:image/*",
  "Condition": {
    "StringNotEquals": {
      "ec2:Owner": [
        "amazon",
        "aws-marketplace",
        "YOUR_TRUSTED_ACCOUNT_ID"
      ]
    }
  }
}
```

**② Golden AMI 파이프라인 구축**

Golden AMI란 보안 검증과 컴플라이언스 테스트를 통과한 신뢰할 수 있는 기준선 이미지입니다. AWS EC2 Image Builder를 활용하면 아래 프로세스를 완전 자동화할 수 있습니다.

<p align="center"><img src="https://d2908q01vomqb2.cloudfront.net/761f22b2c1593d0bb87e0b606f990ba4974706de/2018/05/16/GAP-process.png" width="80%"></p>

AWS 역시 공식적으로 Golden AMI 파이프라인 구성을 권장하고 있습니다.

**③ 런타임 탐지 시스템 구축**

이 사고에서 크립토마이너가 5년간 발견되지 않았던 핵심 원인은 **런타임 모니터링 체계의 부재**입니다. 검증되지 않은 AMI를 사용했더라도 모니터링이 갖춰져 있었다면 CPU·네트워크 이상 징후를 통해 훨씬 빨리 발견할 수 있었을 것입니다.

| 서비스 | 역할 |
|--------|------|
| GuardDuty | 크립토마이너, 비정상 네트워크 통신, 자격증명 탈취 탐지 |
| Inspector | EC2 인스턴스 및 컨테이너 이미지의 CVE 취약점 스캔 |
| Security Hub | 보안 이벤트 통합 대시보드 및 자동 조치 |
| CloudWatch Anomaly Detection | CPU·네트워크 사용량 이상 감지 및 자동 알림 |

**④ 최소 권한 원칙 적용**

EC2 인스턴스 자체가 침해되더라도, 최소 권한 원칙이 적용되어 있다면 피해 범위를 효과적으로 제한할 수 있습니다.

- **IAM Role 최소 권한**: EC2에 필요 이상의 IAM 권한 부여 금지
- **VPC 세분화**: Security Group 최소화 및 불필요한 인터넷 직접 통신 차단
- **IMDSv2 강제 적용**: 메타데이터 서비스를 통한 자격증명 탈취 방어
- **Private Subnet 운영**: 인터넷 게이트웨이 직접 연결 최소화

**⑤ 공동 책임 모델에 대한 이해**

AWS는 Community AMI 사용에 대해 다음과 같이 명시하고 있습니다.

> "You use a shared AMI at your own risk. Amazon can't vouch for the integrity or security of AMIs shared by other Amazon EC2 users."

즉, Community AMI 사용으로 인해 발생하는 보안 사고의 책임은 전적으로 사용자에게 있습니다. Mitiga는 AWS가 다른 플랫폼(GitHub, Docker Hub 등)과 달리 Community AMI에 대한 어떠한 안전장치도 제공하지 않는다는 점을 "충분하지 않다"고 지적했지만, 현재 정책상 이에 대한 조직의 자체적인 검증 체계와 명확한 AMI 사용 정책 수립이 필수적입니다.

---

**참고 자료**

- [Security Advisory: Mitiga Recommends All AWS Customers Running Community AMIs to Verify Them for Malicious Code](https://www.businesswire.com/news/home/20200821005011/en/Security-Advisory-Mitiga-Recommends-All-AWS-Customers-Running-Community-AMIs-to-Verify-Them-for-Malicious-Code)
- [Cryptominer Found Embedded in AWS Community AMI – Dark Reading](https://www.darkreading.com/cloud-security/cryptominer-found-embedded-in-aws-community-ami)
- [Researchers Sound Alarm Over Malicious AWS Community AMIs – Threatpost](https://threatpost.com/malicious-aws-community-amis/158555/)
- [Malicious AMI Detection: Securing Amazon Machine Images – Infosec Institute](https://www.infosecinstitute.com/resources/cloud/malicious-amazon-machine-images-amis/)
- [Announcing the Golden AMI Pipeline – AWS Blog](https://aws.amazon.com/ko/blogs/awsmarketplace/announcing-the-golden-ami-pipeline/)