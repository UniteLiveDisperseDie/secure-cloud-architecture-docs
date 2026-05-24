<p align="center"><img src="https://velog.velcdn.com/images/hvvup/post/aa8d2564-903c-4eed-9709-1f1d36c8344f/image.png" width="50%"></p>

## 1. 개요

### 1.1 사고 배경

2018년 초, 클라우드 보안 업체 RedLock이 Tesla의 AWS 클라우드 환경에서 발생한 침해사고를 발견하였습니다. Tesla는 당시 차량 텔레메트리, 머신러닝 파이프라인, 인프라 운영 등 다수의 워크로드를 Kubernetes 기반으로 AWS 위에서 운영하고 있었습니다.

해당 시기는 Kubernetes가 엔터프라이즈 환경에 본격적으로 도입되던 초기 단계로, 보안 설정 미흡 및 운영 미숙이 광범위하게 존재하던 시기였습니다. Kubernetes Dashboard라는 웹 기반 관리 UI가 기본 설치 구성요소로 제공되었으나, 인증 설정 없이 외부에 노출되는 사례가 빈번했다고 합니다. 이로 인해 발생한 대표적인 사례로 볼 수 있습니다.

### 1.2 사고 요약

| 항목 | 내용 |
|---|---|
| 발생 시기 | 2018년 2월 |
| 발견 주체 | RedLock 보안 연구팀 |
| 침해 경로 | 인증 없이 공개된 Kubernetes Dashboard |
| 탈취 자산 | AWS IAM Credentials (Access Key / Secret) |
| 피해 범위 | AWS S3 버킷 접근, 암호화폐 채굴(Cryptojacking) |
| 근본 원인 | IAM 과다 권한 부여 + Workload Identity 보호 실패 |

공격자는 Tesla의 Kubernetes Dashboard에 인증 없이 접근한 뒤, 실행 중인 Pod 내부에 평문으로 저장된 AWS Credentials를 탈취하였습니다. 탈취한 자격증명에 연결된 IAM Role은 필요 이상의 광범위한 권한을 보유하고 있었으며, 공격자는 이를 활용해 S3 버킷 내 민감 데이터에 접근하고, AWS 컴퓨팅 자원을 암호화폐 채굴에 악용하였습니다.

<p align="center"><img src="https://i0.wp.com/securityaffairs.com/wp-content/uploads/2018/02/Tesla-data-breach.png?resize=768%2C483&ssl=1" width="70%"></p>

---

## 2. 공격 분석

### 2.1 공격 흐름 (Attack Flow)

```
[인터넷]
    │
    ▼
[Kubernetes Dashboard]  ← 인증 없이 외부 공개 (0.0.0.0:80)
    │
    ▼
[Pod 내부 접근]  ← kubectl exec / Dashboard UI를 통한 컨테이너 조회
    │
    ▼
[AWS Credentials 탈취]  ← 환경변수 또는 파일에 평문 저장된 Access Key
    │
    ▼
[IAM Role 권한 행사]  ← AssumeRole 또는 직접 API 호출
    │
    ├──▶ [S3 버킷 열람 / 데이터 탈취]
    │
    └──▶ [EC2 자원 악용 → Cryptojacking]
```

- Kubernetes Dashboard가 인증 없이 외부에 공개되었습니다.
- Pod 내부에 AWS Credentials가 노출되었습니다.
- 공격자가 S3 버킷에 접근하였습니다.
- Monero cryptojacking이 이루어졌습니다.
- 채굴 트래픽을 CloudFront 뒤에 숨겨 탐지를 우회하였습니다.
- CPU 사용률을 낮게 유지하는 방식으로 은닉하였습니다.

<br>

### 2.2 단계별 공격 프로세스

**1. 초기 접근**

공격자는 인터넷에 노출된 Kubernetes Dashboard UI에 인증 없이 접속하였습니다. 해당 Dashboard는 클러스터 내 모든 네임스페이스의 Pod, Deployment, Secret 등을 조회하고 조작할 수 있는 관리자급 인터페이스였으나, `--enable-skip-login` 옵션 또는 RBAC 미설정으로 인해 누구나 접근 가능한 상태였습니다.

**2. 내부 탐색 및 자격증명 수집**

Dashboard를 통해 실행 중인 Pod 목록을 확인하고, Pod의 환경변수(env) 및 마운트된 파일 시스템을 열람하였습니다.

일부 Pod에는 AWS 계정의 access key ID와 secret access key가 환경변수 혹은 설정 파일 형태로 평문 저장되어 있었으며, 공격자는 이를 열람 및 사용할 수 있었습니다.

**3. 권한 상승 및 횡적 이동**

탈취한 Credentials에 정확히 어떤 권한이 있었는지 알 수는 없지만, 계정 자체의 권한이나 연결된 IAM Role은 `Action: "*"`, `Resource: "*"` 형태의 과도한 권한 정책이 붙어 있었을 수 있습니다.

혹은 해당 S3에 대한 권한이 추가되어 있는 등 해당 role로 S3에 접근할 수 있었던 것은 분명합니다.

**4. 결과**

- **데이터 접근**: S3 버킷 목록 열람 및 Tesla 내부 데이터(텔레메트리, 모델 학습 데이터 등 추정) 접근
- **자원 악용**: 탐지를 피하기 위해 채굴 풀(Mining Pool) 트래픽을 CloudFront 뒤에 숨기고, CPU 사용률을 낮게 유지하는 방식으로 Monero(모네로) 암호화폐 채굴을 수행하였습니다.
- **은닉**: 비정상적인 트래픽 패턴을 숨기기 위해 커스텀 마이닝 소프트웨어를 사용하여 일반적인 모니터링 탐지를 우회하였습니다.

Tesla는 이 사고에서 고객의 개인 정보나 보안에 직결된 정보는 해커에게 노출되지 않았다고 밝혔으며, 해커 접근 1시간 이후에 바로 조치를 했다고 합니다. 또한 해당 사고는 회사 내부에서 사용하는 테스트 자동차들에 대한 데이터에 한정되었다고 공개하였습니다.

다만 회사의 특성상 테스트 데이터 유출 또한 치명적일 수 있을 것입니다.

---

## 3. 대응 방안

### 3.1 즉각 대응 절차

사고 발생 또는 탐지 직후 아래 순서로 긴급 조치를 수행해야 합니다.

**[1단계] 자격증명 즉시 무효화**
- 탈취된 것으로 의심되는 AWS Access Key를 즉시 비활성화 및 삭제합니다.
- 해당 IAM User/Role에 연결된 모든 세션 토큰을 강제 만료합니다. (`aws iam delete-access-key`)
- CloudTrail 로그를 통해 해당 Credentials로 수행된 모든 API 호출 이력을 수집합니다.

**[2단계] 피해 범위 파악**
- CloudTrail, S3 Access Log, VPC Flow Log를 기준으로 비정상 접근 IP 및 행위를 식별합니다.
- 접근된 S3 버킷 목록과 오브젝트별 접근 기록을 확인합니다.
- 공격자가 생성한 IAM User, Role, Access Key 등 백도어 리소스를 전수 조사합니다.

**[3단계] Kubernetes 클러스터 격리**
- Dashboard 서비스를 즉시 내부망으로 제한하거나 서비스를 중단합니다.
- 침해된 Pod 및 Node를 클러스터에서 격리(Cordon & Drain)한 후 포렌식 이미지를 확보합니다.
- 해당 네임스페이스의 ServiceAccount Token을 전면 교체합니다.

**[4단계] 외부 공유 및 보고**
- 내부 CISO 및 법무팀에 보고하고, 필요 시 관련 규제 기관에 신고합니다. (개인정보 포함 여부 확인)
- AWS 측에 침해 사실을 통보하고 어뷰징 리소스 정리를 요청합니다.

<br>

### 3.2 사후 조치 및 재발 방지

**1. IAM 최소 권한 원칙 적용**

모든 IAM Role 및 Policy를 전면 재검토하여 `*:*` 형태의 와일드카드 권한을 제거합니다. 각 워크로드에는 해당 기능 수행에 필요한 최소한의 Action과 Resource만 허용하며, IAM Permission Boundary를 설정하여 역할이 위임되더라도 권한 범위가 초과되지 않도록 합니다.

이 사건에서는 특히 S3에 접근하는 것 외에도 공격자가 권한을 가지고 cryptojacking 수행이 가능하였습니다. 이는 어느 단계에서든 최소 권한을 제대로 적용하지 않았다는 의미이기도 합니다.

해당 사건이 정확히 탈취된 크레덴셜이 가진 과/오권한 때문에 발생하였는지, privilege escalation을 통해 추가적인 권한을 획득한 것인지는 알 수 없습니다. 하지만 Pod이 가지고 있는 권한의 경계를 명확히 분리하고, 필요한 권한 외에는 할당하지 않도록 해야 합니다.

**2. Kubernetes Workload Identity 보호**

Pod에 AWS Credentials를 직접 저장하는 방식을 우선적으로 대체해야 합니다. IRSA 또는 EKS Pod Identity를 사용하여 Pod 수준의 임시 자격증명을 발급받는 구조로 전환합니다. 이를 통해 자격증명 유출 자체를 원천 차단할 수 있습니다.

| 방식 | 위험도 | 권장 여부 |
|---|---|---|
| 환경변수에 Access Key 직접 저장 | 매우 높음 | 사용 금지 |
| Kubernetes Secret으로 저장 | 높음 | 비권장 |
| **IRSA / Pod Identity (임시 토큰)** | 낮음 | 권장 |

**3. Kubernetes Dashboard 및 관리 UI 접근 통제**

- Dashboard는 기본적으로 외부 노출을 차단하고, 반드시 내부망 또는 VPN 경유 접근만 허용합니다.
- RBAC을 통해 최소 권한의 ServiceAccount만 Dashboard에 바인딩합니다.
- `--enable-skip-login` 옵션을 비활성화하고 강제 인증을 적용합니다.

**4. 시크릿 관리 체계 도입**

Pod에 직접적으로 크레덴셜이 노출되어 있다는 점이 이 사건의 시작이었습니다. 계정 정보와 같은 경우 인프라에 직접적으로 접근할 수 있도록 하므로, AWS Secrets Manager 또는 HashiCorp Vault를 도입하여 모든 자격증명을 중앙 관리하고, 애플리케이션은 런타임에 API를 통해 동적으로 시크릿을 주입받는 구조로 전환하는 등의 조치가 필요합니다.

코드 및 컨테이너 이미지 내 하드코딩된 자격증명은 Trufflehog, Gitleaks 등의 도구로 정기 스캔하는 방법도 고려해볼 수 있습니다.

**5. 지속적 탐지 및 모니터링 강화**

- AWS GuardDuty를 사용하여 비정상 API 호출, 자격증명 유출, 암호화폐 채굴 트래픽을 자동 탐지합니다.
- CloudTrail + SIEM 연동으로 IAM 고위험 행위(대량 S3 조회, 신규 IAM 생성 등)에 대한 실시간 Alert를 설정합니다.
- Kubernetes Audit Log를 활성화하여 Dashboard 접근, exec 명령, Secret 조회 등의 이벤트를 기록 및 분석합니다.

<br>

---

# 기타

## K8S Dashboard 노출

이 사고의 시초는 **노출된 k8s dashboard를 통한 접근**이었습니다. 그렇다면 일반적으로 어떤 방법을 통해 k8s dashboard를 사용하고, 각각이 가지는 보안적 한계점은 무엇인지 살펴보겠습니다.

### 1. kubectl proxy

```
kubectl proxy --port=8001
# → localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/...
```

**동작 원리**

- 로컬 머신과 Kubernetes API Server 사이에 터널을 생성합니다.
- Dashboard가 외부에 노출되지 않고 localhost에서만 접근 가능합니다.
- 인증은 kubeconfig에 의존합니다.

**한계**

- 접근하려면 반드시 kubectl이 설치되어 있어야 하고, kubeconfig도 있어야 합니다.
- PM, QA, 기획자 등 비개발 직군이 사용하려면 현실적으로 어렵습니다.

=> 보안은 좋지만 운용 불편으로 인해 다른 방법을 찾게 만드는 원인이 됩니다.

### 2. LoadBalancer/NodePort

```yaml
# 이런 식으로 서비스 타입을 설정하면 외부에 바로 노출됩니다.
spec:
  type: LoadBalancer  # 또는 NodePort
```

**동작 원리**

- Dashboard 서비스를 클러스터 외부에서 직접 접근 가능하도록 노출합니다.
- LoadBalancer는 클라우드의 외부 IP를 발급받아 인터넷에서 접근 가능합니다.
- NodePort는 노드의 특정 포트를 통해 외부 접근이 가능합니다.

**한계**

**이 글에서 분석한 Tesla 케이스의 Dashboard 접근 방식입니다.**

- 방화벽 규칙 하나를 잘못 설정하면 인터넷에 Dashboard가 그대로 노출됩니다.
- 네트워크 경계 보안(Perimeter Security)에만 의존하는 구조인데, 이는 단일 실패 지점(Single Point of Failure)이 됩니다.
- 매우 위험한 방식으로, 지양해야 합니다.

### 3. OAuth Proxy

```
[사용자] → [OAuth2-Proxy] → [Kubernetes Dashboard]
                ↑
         Google/GitHub 등 OIDC 인증
```

**동작 원리**

- Dashboard 앞단에 인증 프록시(OAuth2-Proxy 등)를 배치합니다.
- 사용자는 Google, GitHub 등 외부 IdP로부터 인증 후 토큰을 받아 접근합니다.
- 인증된 사용자만 접근 가능합니다.

**한계**

- 구성 요소가 많아 복잡도가 높습니다.
   - Ingress Controller 설정
   - TLS 인증서 관리
   - OIDC 연동 설정
- 설정 및 유지보수 부담이 크지만, 세 가지 방법 중 가장 현실적으로 안전한 방법으로 평가됩니다.

<br>

## 개념 정리

### IRSA(IAM Roles for Service Accounts)

- Kubernetes의 ServiceAccount와 AWS의 IAM Role을 연결하는 방식입니다.
- Pod은 ServiceAccount를 통해 IAM Role을 위임받아 임시 자격증명을 얻습니다.

**동작 방식**
```
[Pod 실행]
    │
    ▼
Kubernetes가 ServiceAccount Token 발급 (OIDC JWT)
    │
    ▼
Pod 내부 AWS SDK가 Token을 STS에 전달
(AssumeRoleWithWebIdentity 호출)
    │
    ▼
STS가 IAM Role의 임시 자격증명 반환
(Access Key + Secret + Session Token, 만료 시간 존재)
    │
    ▼
Pod이 임시 자격증명으로 AWS API 호출
```

**한계**

- OIDC Provider 설정, Trust Policy, ServiceAccount 어노테이션 등 설정 단계가 많고 복잡합니다.
- IAM Role과 ServiceAccount 간 연결이 클러스터 OIDC endpoint URL에 강하게 결합되어 있어, 클러스터 재생성 시 Trust Policy를 전부 수정해야 합니다.
- OIDC Token이 Pod 내 파일시스템에 마운트되므로, 토큰 자체가 탈취될 경우 STS 호출이 가능합니다.

### EKS Pod Identity

IRSA의 복잡성을 해결하기 위해 출시한 새로운 방식으로, OIDC 없이 EKS 자체가 자격증명 발급을 중개합니다.

**동작 방식**
```
[Pod 실행]
    │
    ▼
EKS Pod Identity Agent (DaemonSet)가 노드에서 실행 중
    │
    ▼
Pod 내 AWS SDK가 Agent에 자격증명 요청
(169.254.170.23 로컬 엔드포인트 호출)
    │
    ▼
Agent가 EKS Auth API를 통해 임시 자격증명 반환
    │
    ▼
Pod이 임시 자격증명으로 AWS API 호출
```

위에서 살펴본 IRSA의 OIDC, STS AssumeRoleWithWebIdentity 과정 없이 EKS가 내부적으로 처리합니다.

<br>

---

참고:

- [Hack Brief: Hackers Enlisted Tesla's Public Cloud to Mine Cryptocurrency](https://www.wired.com/story/cryptojacking-tesla-amazon-cloud/)
- [K8s-Dashboard-Manager: Zero Trust Access with Teleport](https://medium.com/@omarnoor_5895/k8s-dashboard-manager-zero-trust-access-with-teleport-f38b0ad8b206)
- [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Hackers compromised a Tesla Internal Servers with a Cryptocurrency miner](https://securityaffairs.com/69413/data-breach/tesla-servers-hacked.html)