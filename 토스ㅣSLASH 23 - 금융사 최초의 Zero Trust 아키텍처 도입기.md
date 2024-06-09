#  토스ㅣSLASH 23 - 금융사 최초의 Zero Trust 아키텍처 도입기 
[ 토스ㅣSLASH 23 - 금융사 최초의 Zero Trust 아키텍처 도입기 ](https://youtu.be/2V2xYeqUsWw?si=9uLvEEmkFVih57VM)


## 기존의 로그인 방식의 불편함
- 많은 Saas, On-prem, Public Cloud 같은 시스템의 계정들은 관리하는 방식도 다르고 계정들도 분리되어있었음
- 매번 ID/PW + OTP MFA를 이용해야함
- 각기 다른 ID/PW, 3개월마다 변경이 필요
- PWD 분실로 인한 업무지연
- 매번 찾아서 입력해야하는 OTP
- 보안팀 입장에서의 불편함 : Active Directory 취약점, 보안사고의 위험성
  - 취약점을 주기적으로 패치
 
### 기존의 네트워크 접근 방식
- 직원들 :
  - 재택근무를 하기 위해서는 매번 SSL-VPN에 접속해야함
  - Office환경과 재택근무 환경 각각 방화벽을 다르게 허용해야함
- 보안팀 : 
  - 재택근무 환경과 Office환경의 관리포인트 이원화
  - IP기반의 접근제어 방식 관리 및 추적의 어려움 
  - 허용이되면 계속 접근할 수 있는 시스템 (경계기반 보안의 한계) -> IP를 이후 입사자가 승계받으면 받아야하는 권한보다 더 많이 받게 될수도
- 무수히 많은 보안솔루션, 개발도구와의 충돌
- MacOS디바이스의 키체인과 충돌, 동기화 오류로 인한 업무 지연

`보안성이 높아지면 왜 더 일하기 불편해질까...?`

# 요구사항
1. ID/PWD 통합관리
2. SSO + 생체인증
3. Active Directory 대체 수단 (관리가 더 편하고
4. 사용자 기반 네트워크 접근 통제 + 암호화 트래픽 가시성 확보
5. PC 보안 솔루션 다이어트 + 접근 통제시스템과의 연동을 통한 자산 및 보안 식별


## 새로운 보안 모델 : Zero Trust
Zero trust란? 고정된 네트워크 경계 방어 -> 사용자/자산/자원 중심의 방어로 변경하는 발전적인 사이버 보안 패러다임
- 자산 또는 사용자 계정에 부여된 암묵적인 신뢰는 없다고 가정

- IAM (Identity Access Management)
  - 기업에서 적합한 사용자, 디바이스가 필요할 때 원하는 시스템에 액세스할 수 있도록 허용하는 프레임워크
  - SSO는 SAML, OIDC와 같은 인증 프로토콜을 사용하여 회사의 시스템과 로그인을 통합
  - RBAC & ABAC
    - Role Based Access Control : 인사DB와 연계하여 직군, 팀단위로 접근제어
    - Attribute Based Access Control : 권한이 있더라도 접근할때 회사에서 정의한 네트워크, 자산, 보안은 준수하는지 검증
  - Policy : 어플리케이션에 접근할때의 Factor(ID/PW, MFA)등을 정의, RBAC/ABAC 검증을 통한 제어, Role/Network/자산/보안 등을 정의하고 검증
  - Security : Threat Intelligence 분석을 통해 IAM Provider와 회사의 위협을 미리 파악, SIEM연동을 통해 이상행위 분석
- SASE (Secure Access Service Edge)
  -  사용자, 시스템, 엔드포인트 및 원격 네트워크 앱 및 리소스를 안전하게 연결하는 클라우드 전달 플랫폼으로 SD-WAN(Software Defined Wide-Area Network) 및 제로 트러스트 보안을 집중하는 보안 프레임워크
  -  Cloud Delivery Platform
  -  Policy : SSL Inspecton을 통한 HTTPS 암호화된 패킷을 통제, 분석하기 용이, URL/APP Control을 통해 피싱 차단
  -  Data Protection : DLP(Data Loss Protection)을 통해 데이터 유출방지, CASB(Cloud Access Security Broker)를 통한 Cloud에 보관된 중요 데이터 유출 방지
  -  Threat Management : 멀웨어, ATP, Sandbox
- ZTNA (Zero Trust Network Access)
  - 명확하게 정의된 액세스 제어 정책을 기반으로 조직의 리소스에 대한 보안 원격 액세스를 제공하는 서비스
  - VPN은 Network 단위 ZITA는 어플리케이션, 서비스 단위로 보안
  - Role : IAM 연동을 통해 사용자, 그룹 정의
  - Segment : 내부 네트워크의 자산을 어플리케이션 단위로 정의하고 분석을 통한 어플리케이션 식별
  - Policy : Role + Segment -> Role이 있는 사용자여도 보안 정책을 준수하지 않으면 차단
- UEM (Unified Endpoint Management)
  - PC와 모바일 디바이스를 중앙에서 통합관리하는 솔루션
  - 디바이스를 분실했을 때 잠금, 원격 처리
  - OS, 필수 프로그램을 자동 설치 및 버전 관리
- EPP (Endpoint Protection Platform)
  - PC< 디바이스에서 발생할 수 있는 다양한 보안 기능 통합으로 제공
  - Anti 멀웨어, EDR


## Zero Trust 전환 절차 
1. IAM 온보딩 : 전 직원 계정 재생성 & MFA 등록
2. PC에 조인된 AD제거 & UEM으로 전환
3. UEM에 접속시 자동으로 Profile 다운로드 & 불필요한 어플리케이션 삭제 & 새로운 인증서&앱 설치
4. SASE&ZITNA 로그인
