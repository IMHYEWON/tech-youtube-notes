#  토스ㅣSLASH 23 - 새로운 은행을 위한 Modern 대외 연계 시스템 구축기 
[ 토스ㅣSLASH 23 - 새로운 은행을 위한 Modern 대외 연계 시스템 구축기 ](https://youtu.be/eS9tukmYBLI?si=CC6FV84c7rI3e-Pn)

## 은행시스템 구성
- 채널계 
  - MSA환경
  - 스프링, 카프카 오픈소스 사용
  - 앱과의 연계
 
- 계정계(Corebanking 시스템)
  - SI회사가 제작한 차세대 시스템을 기본으로 한 모놀리식 구조
  - 은행 고유의 여러 업무와 금원에 관한 트랜잭션을 안정적으로 처리하는 역할
 
- 대외기관
  - 금융결제원, 신용정보원
  - 계정계에서 금융관련 업무를 처리할때 이 대외기관과의 협력이 필요
  - 계정계 <- FEP (Front End Processor) -> 대외기관

 ## FEP (Front End Processor)
 - 대외기관과 메시지 통신
 - TCP
 - 연결 유지형, 세션관리 필요

### ASIS FEP의 아쉬운점
채널계 ←→ MCI ←→ 계정계 ←→ FEP ←→ 대외기관

1. 생산성  
   a. 간단한 기능을 만드려고 해도 대외기관을 통해야함 → 여러 담당자들이 협업해야함  
   b. 토스에서 자체적으로 만든 여러 시스템 (로깅, 모니터링, 배포환경) 과의 연동이 제한적임  
   c. HTTP 미지원 등 여러 제약  
2. 안정성  
  a. Blocking IO만을 지원, 요청이 많이 발생되면 스레드가 다 사용돼서 네트워크 지연 장애  
  b. FEP는 Active-standby형태로 유지중  
  c. 인스턴스가 제한적이라 하나의 인스턴스가 여러개의 대외업무를 함께 처리함 → 장애전파  
3. 유지보수성  
  a. 유지보수가 어렵고, 솔루션 툴은 FEP전문 인력만 유지보수 할 수 있다는 단점  

## Modern 대외연계 시스템 (Modern FEP) 처음부터 다시 만들
<img width="610" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/d29523c6-8e31-43e5-bfa8-b434d51a8b6e">

1. 데이터 형태 구현
- 데이터 형태
  - Json과 같은 타입이 아닌 바이트 형태로 통신
  - 항목 + 바이트 크기 + 타입 + 디코드할 캐릭터셋
  - 공통정보부 : 헤더와 유사
  - 데이터부 : 바디와 유사
  - → 코틀린 인터페이스로 변환

- 인터페이스
``` kotlin
interface TelegramInterface
@CommonPart (대외업무명 = SLASH23) // 공통정보부
@Telegramclass (전문명 = 발표자조회)
@TelegramField(순서 = 1, 크기 = 4, charset = ASCII)
@TelegramNameField
```

- 공통정보부 구현
``` kotlin
@CommonPart (대외업무명 = SLASH23)
data class 공통정보부(
  @TelegramField(순서 = 1, 크기 = 4, charset = ASCII)
  var 전문길이: String = "",
  @TelegramNameField
  @TelegramField(순서 = 2, 크기 = 4, charset = ASCII)
  var 전문명: String = "",
) : TelegramInterface
```
- 데이터부 구현
``` kotlin
@Telegramclass (전문명 = "1000")
data class 발표자조회(
  @TelegramField (순서 = 5, 크기 = 10, charset = KS_X_1001)
  var 고객명: String = "",
  @TelegramField(순서 = 6, 크기 = 20, charset = ASCII)
  var 계좌번호: String = "",
) : 공통정보부 ()

```

2. 전문 읽고 쓰기 (인코딩 & 디코딩) -  Modern FEP CODEC의 과정
  1. 인코딩
     1. 정렬된 필드 얻기 -> 만들어둔 어노테이션TelegramField 을 이용해 Java reflection으로 정보 얻기
     2. 필드별 Encode -> 전문 정보 습득 & 캐릭터셋에 따라 인코딩
     3. ByteArray 에 쓰기
  2. 디코딩
     1. byte -> 클래스 정보얻기
       - 공통정보부 클래스 얻기, 전문 클래스 얻기
       - 전문이 정의되어 있는 패키지의 정보를 Reflections 라이브러리라는 서드파티 라이브러리를 이용해 다 가져옴
       - 지금 처리하는 업무명으로 필터링
     2. 정렬된 필드 얻기
     3. 필드별 Decode 


3. 전문을 송수신
<img width="397" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/670786d3-857f-4c93-83e2-0b111e9fb390">

- Blocking IO 해결
  - TCP 통신 (FEP <-> 대외기관) 에는 Netty
  - HTTP통신에는 Spring WebFlux와 WebClient를 사용
 
- Active, Active한 구조에서의 응답
  - ASIS FEP는 FEP인스턴스가 2개인데, FEP1이 대외기관에 TCP요청을 보냄면 FEP2에 응답이 와서 유실됨
  - TCP프로토콜 자체에는 메시지의 요청과 응답이라는 개념이 없음
  - 대부분의 대외업무에서 요청세션과 응답세션을 분리하고 있었음

  - **TOBE 구조**
  <img width="678" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/56e77617-2d49-49b1-8d72-33fc1571858d">
  - 레디스를 활용함
    - 자원을 아끼기위해 Pubsub을 사용해도 되지만, 펍섭은 At-most-once로 동작하기 때문에 신뢰성을 위해 polling으로 활용함

- 대외업무별 분리
- <img width="538" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/7f2a9a0a-8021-4b0b-9326-416b192105d4">

  - 장애 도메인 분리, 최소화
  - 대외업무별 세부요건을 지원하기 위해 추상화된 인터페이스 개발
  - 대외업무를 맡은 서버개발자가 FEP를 몰라도 쉽게 개발할 수 있음










