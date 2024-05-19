#  [NHN FORWARD 22] 로그인에 사용하는 OAuth : 과거, 현재 그리고 미래 
[ [NHN FORWARD 22] 로그인에 사용하는 OAuth : 과거, 현재 그리고 미래 ](https://www.youtube.com/watch?v=DQFv0AxTEgM)

1. 웹 환경에서 인증과 인가
2. OAuth 1.0
3. OAuth 2.0
4. OAuth 2.1
5. OPenID Connect
6. GNAP


## 웹 환경에서 인증과 인가
- 인증 : 인터넷에서 사용자의 신원을 확인하는 행위
- 인가 :

## Oauth 1.0
- Resource Owner : 일반 사용자
- OAuth Client : 리소스 오너의 리소스에 접근해서 이것을 활용하려고 하는 어플리케이션 (ex. SNS 사진 에디터)
- OAuth Server : 리소스 오너의 신분을 확인해 인증하고 리소스오너와 상호작용해서 Oauth클라이언트에게 인가
- **Oauth 1.0의 흐름**
    1. 리소스 오너는 클라이언트(ex. 사진에디터)를 이용해서 기능을 수행하고 싶음
    2. 클라이언트는 이런 요구사항을 들어주기 위해, Oauth Server에게 '나 대신 리소스 오너에게 인가를 받고 이 주소로 인가 정보를 보내주세요' 라는 요청을 하게 됨
    3. 클라이언트는 Oauth서버대신 리소스오너에게 나를 이용하기 위해서는 Oauth 서버에서 인증을받고오도록 하라는 의미로 로그인 페이지를 리다이렉트
    4. 리소스 오너는 인증을 거치고 'SNS 사진 에디터가 (리소스 오너)의 사진에 접근하는것을 동의하십니까?' 와 같은 인가 후 Oauth 클라이언트에게 인가코드 전달
    5. 클라이언트는 이 임시 인가코드를 가지고 Oauth서버로부터 토큰을 받고 이후에 이 토큰으로 리소스에 접근하게 됨
  -> 클라이언트에게 ID, PWD정보 전달 없이 리소스를 안전하게 줄 수 있다?!
- *Oauth 1.0의 문제점*
  - scope 개념 없음
  - 토큰 유효기간 문제
  - 역할이 확실히 나누어지지 않음
  - CLient 구현 복잡성
  - 제한적인 사용 환경

## Oauth 2.0 인가
- 현재 가장 널리 쓰이는 인가 프로토콜과 버전
- Oauth 1.0은 토큰만 있으면 모든 리소스에 접근할 수 있고 접근 범위를 제한할 수 없었으나, 2.0부터는 scope개념이 생겨서 제한 할 수 있음
- Client 복잡성 간소화
  - 1.0에서는 서명을 구현하기 위해, 파라미터 순서 정렬/엔코딩 등의 복잡한 서명 본문을 만들어야만 스펙 구현 가능
  - -> Bearer Token + TLS로
- 토큰 탈취 문제 개선 -> Access Token + Refresh Token
- 제한적인 사용 환경 개선 : **Grant**개념의 등장
  - Authorization Code, Implicit, Resource Owner Password credentials,
  - Authorization Code : 서버 투 서버 통신을 통해
  - Implicit :
  -  Resource Owner Password credentials : 직접적으로 ID, PWD를 받아
  -  Client Credentials : 사실상 리소스오너와 Oauth Client가 동일한 객체일때 복잡한 플로우보다 직접적으로 API를 통해 발급

## Oauth 2.1 인가
- 더 보안적으로 민감한 사용처들의 프로토콜 채택 (ex. 금융권, 의료계..)
- 1.0 > 2.1 에서 구현을 쉽게 하기위해 trade off 로 일부 보안 리스크 업
- 보완하는 PKCE, BCP 같은 RFC가 나오게 됨
- 직접적으로 유저의 패스워드를 받거나 유저에게 직접 토큰을 보내는 Implicit, Resource Owner Password credentials 폐지
- *Device Authorization Grant*가 등장
  - IOT기기에 적합
- 검증 값 탈취를 막기 위한 PKCE
  - 클라이언트가 어떤 랜덤값에 대한 hash를 서버를 보냄
  - 마지막 토큰 값 요청시에는 Hash의 원문을 보내서 첫 요청과 마지막 요청의 검증을 받을 수 있게 됨
- BCP : 리프레시 토큰 회전방식의 등장


## OpenID Connect 인증 계층
- Oauth로도 인증을 할 수 있지 않나요??
- Oauth 2.0 스펙기준으로 확장한 인증 방식
- OpenID 인증방식을 통해 인증하면 ID Token을 응답으로 받을 수 있음
- ID Token에는 토큰 발급자, 발급 시간, 토큰사용자(클라이언트 식별자), 사용자 식별자, 토큰 만료시간 등이 포함되어있음
- 이 토큰을 사용하면 기존처럼 토큰을 가지고 API CALL을 하지 않고 ID Token으로 사용자 정보를 파싱할 수 있음
- 트래픽이 늘어나는 환경에서 인기가 많아짐

