## 토스ㅣSLASH 23 - 토스는 Gateway 이렇게 씁니다 
[ 토스ㅣSLASH 23 - 토스는 Gateway 이렇게 씁니다 ](https://www.youtube.com/watch?v=Zs3jVelp0L8&t=4s)


### ❗️API Gateway가 없다면?
- 인증 서비스같은 공통 부분에 업데이트가 생기면 모든 클라이언트 서비스들이 업데이트해야 되고 휴먼에러, 장애의 가능성이 높아짐
- 매우 비효율적

|AS IS|TO BE|
|---|---|
|<img width="489" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/7a5fe684-c0a2-4a65-a33a-99b57ebe4229">|<img width="613" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/15bb0f97-835e-4018-8c19-34224a707e98">|

### GateWay란?
라우팅 및 프로토콜 변환을 담당하며, 마이크로 서비스 사이의 중개자 역할을 해 클라이언트가 독립적으로 추가될 수 있도록 한다. 
- 모든 서비스에서 필요한 유저정보, 보안정책들을 처리
- 동작
  - 요청이 오면 정의된 설정에 따라 라우팅
  - 설정된 필터들을 작동
  - 필터들을 통과하면 업스트림 서버로 요청 프록시
- **구성요소**
  - `Route` : 설정 구성 단위
  - `Predicate` : 요청을 구분할 때 사용하는 값, path, method, host 등으로 요청을 매칭
  - `filter` : 매칭된 요청에 대한 전처리, 서비스 응답에 대한 후처리 구현
    - 하나의 라우터 안에서 여러개의 필터가 있을 수 있으며, 순서에 따라 필터를 처리함

### BFF (Backend For Frontend)
- API Gateway가 하나의 거대한 Monolithic 서비스가 되었음
  - ex. Web, App에서 필터 로직이 각각 다르지만 GateWay는 이 두가지로직을 모두 가지고 있어야 한다.
  - 👉 이를 해결하기 위해 BFF 패턴 적용
 
#### BFF패턴이란?
Client에 맞는 하나의 백엔드를 사용하는 패턴
- client별로 **관심사를 분리**하고 클라이언트에 맞는 로직을 처리
- 요청 방향에 맞게 분리하는 **Ingress/Egress** 패턴
  - `Ingress` : 클러스터에 들어오는 요청을 처리하는 Ingress GateWay
  - `Egress` : 클러스터에 나가는 요청을 처리하는 Egress Gateway

## 토스팀의 GateWay
 <img width="529" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/71b49000-5a01-4841-ad23-8166b7cc4868">
 
#### 아키텍처
- APP Gateway : App에서의 요청을 처리하는 Gateway
- Public Gateway : Web요청을 처리
- Secure Gateway : mTLS 인증서 검증
- SSR Gateway : Server Side Rendering 처리
- Internal Service/Secure Gateway : 사내 요청 처리 (망분리)
- Egress Gateway : 각 계열사별 OutBound 트래픽 처리를 담당하는 Egress-Gateway 

#### 개발 스택
- Spring Cloud Gateway, Spring Webflux
  - 내부적으로 Reactor-Netty를 사용하여 비동기 처리를 지원
- Kotlin Coroutine : 필터 개발
- Istio : Ingress/Engress - Envoy 필터

#### 공통로직 개발
- **Request 처리**, **유저**, **보안**, **서비스 안정화**
1. Request 처리
  - Request Sanitize : 클라이언트로부터 올바르지 않은 요청이 올 경우 이를 정제 해줌, 헤더에 올바르지 않은 값이 들어오더라도 이를 변환
  - Trace Id 생성 : 모든 요청의 진입점이기 때문에 Trace Id를 생성하여 전파함,
    - 생성된 Trace Id는 각자의 서비스 내에서 MDC 컨텍스트에 저장되고 로깅에 사용 (MDC 컨텍스트가 뭐지?)
    - 로그에 Trace ID가 남게 되고 어떤 요청이 이어졌는지 릴레이를 알 수 있음
    - *우리 회사 Gateway에서도 지금 Trace ID를 남기고 분산 추적이 되고 있는걸까?*
  - Relay Header : 하나의 트랜잭션 안에 여러 서비스에서 공통 정보가 필요한 경우
    - 기존에는 모든 서비스가 공통 API를 요청하였고, 불필요한 중복 요청이 발생함
    - -> Gateway에서는 이러한 공통정보를 Internal Header로 주입하면 서비스에서는 헤더에서 필요한 값을 가져와 사용
    - *서비스가 필요하다는 정보인 걸 어떻게 알지..*
      - 각 서비스에서 사용자 동의 여부 확인이 필요한 경우 Gateway에서 동의여부 API 요청&응답을 받아 Internal Header에 넣어주고 있구나
      - 각 서비스는 약관 API를 요청할 필요가 없어짐 
2. 유저
  - 기존에서는 모든 서비스에서 유저 정보가 필요할때 각각 유저정보API 호출 -> 불필요한 중복 요청
  - **넷플릭스의 Passport 구조 참고**
  - 넷플릭스 : 유저 인증시에 Passport라는 Id 토큰을 트랜잭션 내로 전파하는 방법 사용 -> 토스 Passport
  - *토스 Passport*
    - 사용자 기기 정보와 유저정보를 담은 하나의 토큰
    - 1) App에서 유저 식별 키와 함께 API 요청
      2) Gateway에서 이 키를 토대로 Auth 서버에 Passport 요청
      3) Gateway에서는 발급받은 Passport를 Serialize해서 전파
      4) 유저정보가 필요한 서비스는 유저정보 호출 없이 Passport 정보를 통하여 유저에 대한 정보 사용 가능
  - [Edge Authentication and Token-Agnostic Identity Propagation](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)
  - [[MSA] Netfilx는 MSA에서 어떻게 인증/인가를 처리할까? : Netfilx Passport](https://velog.io/@sheltonwon/MSA-Netfilx%EB%8A%94-MSA%EC%97%90%EC%84%9C-%EC%96%B4%EB%96%BB%EA%B2%8C-%EC%9D%B8%EC%A6%9D%EC%9D%B8%EA%B0%80%EB%A5%BC-%EC%B2%98%EB%A6%AC%ED%95%A0%EA%B9%8C-Netfilx-Passport)
3. 보안
  - **E2EE (End to End Encryption) **토스에서 사용하는 대부분의 API는 종단간 암호화를 통해 패킷 분석의 허들을 높여 안전한 정보 던잘
  - 1) App에서 암호화 키를 이용하여 요청 바디 암호화
    2) Gateway에서 이를 복호화하여 서비스에 전달
  - 복호화 과정에서 인증, 인가 처리  & 복호화된 데이터와 유저 정보를 서비스로 전달
  - **프론트에서 SSR과 같이 종단간 암호화를 사용하기 어려운 스펙을 개발하는 경우**
    - JWT를 통한 API 요청 기능도 지원
    - 1) 클라이언트는 매 요청마다 새로운 토큰 생성
      2) 토큰+요청을 Public Gateway에 전달 : 게이트웨이에서는 토큰의 서명값, 유효기간, 중복 사용여부, 만료여부를 검증하여 Replay Attacck, 토큰 변조를 방지
      3) 검증된 토큰을 프론트의 Node서버(SSR 서버)에 전달
      4) 전달받은 Node서버는 SSR Gateway를 통해 업스트림 서비스에 접근
      5) SSR Gateway : 토큰을 파싱 & 유저 서버에서 유저정보 가져와서 & 알맞은 서비스에 전달
  - **외부에서 토스 서비스를 접근하는 경우를 위한 Oauth2를 횔용한 인증/인가처리 지원**
    - 1) 허가된 클라이언트는 ClientId, ClientSecret을 가지고 있고 요청 헤더에 담아서 보냄
      2) Gateway에서는 접근 요청이 유효한지 내부 Oauth서버에 질의, 요청이 클라이언트가 가진 scope에 포함되는지 확인 후
      3) 서비스로 요청 전달
    - *갑자기 궁금한 점, 각 BFB서버에서는 지금 사용자 정보 & 세션에 어떻게 접근하고 있는거지? 그냥 클라이언트에 있는 토큰만 함께 보내기?*
    - 토스 App은 내부적으로 짧은 기간을 가진 키값 + 변조되지 않은 토스 앱에서만 알 수 있는 정보를 활용해서 각 요청을 서명하고 이를 Gateway로 보냄
      - 실제로 initialize 되었는지? 위변조된 토큰인지? 만료된 토큰인지? 재사용되었는지?
      - **의심스러운 요청이 발견되면 FDS(Fraud Detection System)을 통해 계정 비활성화 block**
        - 유저의 모든 행위 기반 로그 분석을 통한 의심스러운 막는 기능을 제공 -> block id, user, ip Redis에 저장해서 정교한 차단
        - FDS는 유저의 행위를 분석하여 브루트포스를 이용한 정보 획득, App분석 시도 등 감지
  - **서비스 개발을 위해 외부 서비스나 내부 개발자들의 요청을 위한 클라이언트 인증서를 이용한 mTLS API 호출 지원**
  - <img width="632" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/4d3258d9-59a2-465b-9578-db5c834cf209">  
  - Istio에서 제공하는 mTLS flow + 자체 인증/인가 처리 레이어
  - Istio 외에 추가적인 인증/인가 어플리케이션 gateway 구현 이유
    - 코드 베이스로 더 정교한 Auditing
    - 카나리 배포 가능
  - Secure Gateway 인증 프로세스
    - 1) Edge에서 Istio가 클라이언트 인증서의 CA 유효성 확인
      2) 해당 인증서 정보를 헤더에 실어 모든 트래픽을 게이트웨이에 전달
      3) 받은 인증서를 Decode & X.509인증서 중 Subject Alternate Name을 활용하여 인증서로부터 사용자 정보를 얻게 됨
      4) 사용자, 요청에 대한 인증/인가 Auditing
    - 내부 개발자의 Staging 환경에서도 IP/MAC 망 주소 등이 아닌 Secure Gateway를 사용
      - 단순한 인가만 가능하기 때문에 실제 사용자 정보 획득은 어려움
      - 디버깅을 위한 호환성 높음 : 크롬, java, curl, postman 등  
   - **사내 웹서비스 호출을 위한 OIDC Provider를 활용한 Internal Service Gateway**
   - <img width="615" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/82a6989d-f0a5-462f-aa3a-04bba5b69c97">  
     - OIDC Flow를 통해 Provider에서 Access Token을 가져오고 이를 Redis Session Store을 통해 관리
     - 사내 웹서비스 중 Internal Service Gateway를 프록시로 거치는 서비스는 큰 공수 없이 사내 SSO 로그인을 사용하여 페이지 별 인증/인가 처리를 할 수 있음
     - 핀포인트, 키바나 같은 오픈소스 플랫폼에서도 게이트웨이를 활용한 인증 가능
     
4. 서비스안정화

