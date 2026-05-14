![](https://velog.velcdn.com/images/hvvup/post/1d981e8d-7609-449a-b15b-53b0a144a9f0/image.png)


# 1. 개요

## 1.1 사고 배경

2019년 말레이시아 저가항공사 말린도 에어(Malindo Air)에서 대규모 고객 데이터 유출 사고가 발생하였습니다.

이 사고는 클라우드 스토리지 오설정(misconfiguration)으로 인한 침해로 초기에 보도되었으나, 이후 조사에서 외부 공격이 아닌 내부자(Insider) 소행으로 밝혀졌습니다.

2022년에는 유출된 데이터가 다시 온라인 포럼에 게시되며 재조명되었습니다.

> - **발생 시점:** 2019년 3월
> - **공식 공개:** 2019년 9월 (다크웹 유포 발견 후)
> - **피해 항공사:** 말린도 에어(Malindo Air), 타이 라이언 에어(Thai Lion Air) — 모두 라이언 에어(Lion Air) 그룹 소속
> - **재유포 확인:** 2022년 9월, 약 4,500만 건의 탑승객 레코드가 온라인 포럼에 다시 게시됨
> - **현재 항공사 상태:** 말린도 에어는 이후 바틱 에어(Batik Air)로 리브랜딩

## 1.2 사고 요약

유출된 데이터 항목은 다음과 같습니다.

- 성명, 생년월일, 성별, 국적
- 이메일 주소, 전화번호, 실거주지 주소
- 여권 번호 및 여권 만료일
- 예약 번호, 탑승객 고유 ID, 비상 연락처
- 항공사 마일리지(로열티 프로그램) 정보

<p align="center"><img src="https://velog.velcdn.com/images/hvvup/post/53848475-5307-4685-9f67-6b2c37068a57/image.png" width="75%"></p>

PaymentGateway라는 이름으로 백업 파일이 올라왔습니다. 다음 이미지는 유출된 정보의 샘플입니다.

<p align="center"><img src="https://i0.wp.com/securityaffairs.com/wp-content/uploads/2019/09/Lion-Air-Leak.png?w=1443&ssl=1" width="80%"></p>


---

# 2. 공격 분석

## 2.1 공격 흐름 (Attack Flow)

<p align="center"><img src="https://velog.velcdn.com/images/hvvup/post/519dc4d8-fc1e-49f2-b2b0-26460926cbd0/image.svg" width="80%"></p>

내부자 위협이란 조직 내부에 있거나 내부 시스템에 대한 접근 권한을 가진 개인이 — 의도적이든 비의도적이든 — 조직의 정보자산, 시스템, 네트워크에 피해를 입히는 보안 위협을 말합니다. 외부 공격자와 달리 내부자는 이미 방화벽 안쪽에 있기 때문에, 탐지와 대응이 훨씬 어렵다는 점에서 특히 위험합니다.

## 2.2 단계별 공격 프로세스

본 사고는 외부 침투가 아닌 내부자의 권한 남용으로 발생하였습니다. 내부자는 이미 정상적인 접근 권한을 보유하고 있었으며, 이를 이용해 고객 데이터가 담긴 백업 파일을 외부로 유출하였습니다. 초기에는 클라우드 스토리지 오설정에 의한 침해로 보도되었으나, 이후 조사를 통해 내부자 소행임이 확인되었습니다. 유출된 데이터는 다크웹을 통해 최초 유포되었고, 2022년에 온라인 포럼에 재게시되며 피해가 재확산되었습니다.


---

# 3. 대응 방안

> "To protect customer data, organizations should employ continuous security validation tools to identify and prioritize gaps in security that need to be addressed first, and continuously assessing the viability of their security controls to make sure they are enabled, configured correctly and operating effectively at all times."

## 3.1 즉각 대응 절차

### 로깅·모니터링(이상행위 탐지)

- **필수 감사 로그**

  - Organization‑wide CloudTrail (모든 리전, 모든 계정) 활성화 후, 별도 로그 계정의 S3에 저장 + 버전관리 + Write‑only 권한을 적용합니다.

  - 주요 데이터 소스(RDS, Aurora, DynamoDB, Redshift, OpenSearch 등)에 Database Activity Streams / 감사 로그를 활성화합니다.

- **탐지 컨트롤**

  - Amazon GuardDuty를 통해 IAM 이상 행위(비정상 API, region, 시간대, 대량 데이터 조회 등)를 알림으로 수신합니다.

  - Amazon Macie를 통해 S3 내 민감정보(PII, 카드 정보 등)를 자동 식별·분류하고, 대량 다운로드/퍼블릭 버킷 같은 이벤트를 감지합니다.

  - UEBA/행위 분석(필요 시 써드파티)을 통해 동일 사용자가 갑자기 다른 리전에서 새 S3 버킷 전체를 복제하는 패턴 등 이상행위를 모델링합니다.

- **알림 및 자동 대응**

  - CloudWatch + EventBridge로 "고위험 이벤트"(예: s3:GetObject 대량, iam:CreateAccessKey, kms:DisableKey)를 감지할 경우
    - Security 채널(Slack/Teams/Email) 알림을 발송합니다.
    - 자동화 Lambda로 계정 비활성화/세션 revoke, 임시 네트워크 차단 등을 수행합니다.

<br>

## 3.2 사후 조치 및 재발 방지

### IAM/접근제어 설계

- **최소 권한(Least Privilege) 기본값**

  - 모든 IAM User/Role은 태스크 단위로 권한을 쪼개고, AdministratorAccess, s3:*, rds:* 같은 광범위 정책은 Security Hub / Config 룰로 탐지 및 금지합니다.

  - 정기적인 권한 리뷰와 unused permission 제거(Access Advisor, IAM Access Analyzer)를 수행합니다.

- **계정 구조와 Guardrail (AWS Organizations)**

  - Prod/Dev/Shared/Vendor 계정을 분리하고, Organization SCP로 "절대 하면 안 되는 행위"(예: s3:PutBucketAcl 로 public-read, iam:CreateUser, kms:DisableKey 등)를 상위에서 차단합니다.

  - 루트 계정 사용 금지·알림, 보안 계정(security tooling) 분리를 적용합니다.

- **벤더/운영자 계정 전략**

  - 개인별 federation(SAML/OIDC+AWS IAM Identity Center)으로 접근하게 하고, 장기 Access Key는 금지 또는 자동 회전·폐기합니다.

  - JIT(Just‑in‑Time) 접근: 평소에는 권한을 최소화하고, 필요 시에만 STS AssumeRole + 세션 정책으로 짧은 시간 Elevated 권한을 부여합니다.

  - 업무 분리(SoD): 같은 사람이 Dev+Prod, IAM+Billing, 보안툴+운영툴을 모두 가지지 않도록 역할을 분리합니다.

<br>

### 데이터 레벨 보호(유출 시 피해 최소화)

- **S3·백업 보안 (Malindo 유사 케이스 방지 포인트)**

   - S3 Block Public Access "계정 전체" 활성화 + public ACL/Policy 탐지 Config 룰을 적용합니다.

  - 민감 데이터가 저장된 버킷의 경우
     - KMS 암호화, VPC Endpoint+Bucket Policy로 내부 네트워크만 접근하도록 하며, Cross‑Account Access는 Role+조건부 정책으로만 허용합니다.

- **최소 수집·보존 정책**

  - 여권번호, 주소 같은 고위험 PII는 반드시 필요한 용도/보존기간을 정의하고, 그 이상은 수집·저장하지 않도록 애플리케이션/DB를 설계합니다.

  - 백업/스냅샷에 대해서도 별도의 Retention 정책과 삭제 Job을 두어 "영원히 남는 백업"이 없도록 합니다.

- **마스킹·토큰화**

  - 운영자/CS가 보게 되는 화면·쿼리에는 완전한 식별정보 대신 토큰/마스킹된 데이터만 제공합니다.

  - 결제 정보는 PCI‑DSS 호환 PG/토큰화 솔루션에 위임하며, 자체 저장은 금지합니다.

<br>

### 계정 라이프사이클·오프보딩

- **Joiner‑Mover‑Leaver 프로세스 자동화**

    - HR 시스템과 IdP를 연동하여, 직원/벤더 계약 종료 시 IdP 계정 비활성화 → AWS 접근을 자동으로 회수합니다.

    - Role 변경 시, 이전 권한을 자동으로 회수하고 중복 고권한 Role이 남지 않도록 Identity Governance를 도입합니다.

- **자격 증명 정기 점검**

    - 사용하지 않는 IAM User, Access Key, 콘솔 로그인 90일 이상 없음 등은 자동 비활성·삭제합니다.

    - Root, 보안관리자, CI/CD용 Role 등 높은 위험 계정은 별도 PAM(Privileged Access Management)이나 강제 MFA, session re‑auth를 적용합니다.

<br>

### 프로세스·조직 관점 (Insider Threat Program)

- **Insider Threat 프로그램 수립**

   - IT/보안 + HR + 법무가 함께 참여하는 공식 프로그램으로, 정책·교육·조사·징계 프로세스를 정의합니다.

    - 비정상 행위 발생 시 디지털 포렌식·법적 대응까지 염두에 둔 절차(증거 보존, 접근 제한, 내부 조사)를 마련합니다.

- **교육·심리적 요인 관리**

    - 운영자·벤더에게 "민감 데이터 취급 원칙, 로그가 남는다는 점, 위반 시 제재"를 주기적으로 교육합니다.

    - 과도한 압박·불만이 쌓이지 않도록 HR 차원의 관리도 Insider Threat 가이드에서 중요하게 다룹니다.


---

출처:
**Malindo Air 침해사고 사례**

- [Malindo Air Data Leak Reveals Info of 60 Million Passengers](https://vpnoverview.com/news/malindo-air-data-leak-reveals-info-of-60-million-passengers/)
- [MalindoAir Data Breach](https://haveibeenpwned.com/Breach/MalindoAir)
- [Backup files for Lion Air and parent airlines exposed and exchanged on forums](https://securityaffairs.com/91386/data-breach/lion-air-data-leak.html)
- [Lion Air Breach Hits Millions of Passengers](https://www.infosecurity-magazine.com/news/lion-air-breach-hits-millions-of/)

**내부자 위협**

- [Insider Threat Detection and Management on AWS: Strategies for Mitigating Internal Security Risks](https://just4cloud.com/insider-threat-detection-management-aws/)
- [Insider Threat Guide](https://www.dni.gov/files/NCSC/documents/nittf/20240926_NITTF-Insider-Threat-Guide.pdf)
- [Mitigating Insider Threats in Cloud Environments: Best Practices and Policies](https://aws.plainenglish.io/mitigating-insider-threats-in-cloud-environments-best-practices-and-policies-6c3a8e4c90ba)