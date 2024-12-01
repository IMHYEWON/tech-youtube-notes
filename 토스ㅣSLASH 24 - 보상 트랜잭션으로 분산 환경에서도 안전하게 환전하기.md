# 토스 | SLASH 24 - 보상 트랜잭션으로 분산 환경에서도 안전하게 환전하기
[토스 | SLASH 24 - 보상 트랜잭션으로 분산 환경에서도 안전하게 환전하기](https://youtu.be/xpwRTu47fqY?si=6MTMK3KdaUEwSL08)

## 1. 분산 환경이 만들어진 이유
- ASIS : 토스의 코어뱅킹서버는 Monolithic 서버로, 서버1-DB1이 었음, 이를 조금씩 분리 시작
  - 원화계좌 서버의 경우 같은 DB를 바라보지만 서버 분리
  - 외화계좌와 같은 경우 신규 서버 였기 때문에 DB와 서버 모두 분리된 MSA

- **환전**은 `원화`와 `외화`를 트레이딩 하는 것인데, 단일 서버면 매우 동일 트랜잭션 내부에서 간단하겠지만 분산되어 있기 때문에 쉽지 않음
<img width="359" alt="image" src="https://github.com/user-attachments/assets/5b55ae05-35ec-47df-922f-9f93d708a645">
    
👉 환전의 원자성을 위해 분산 트랜잭션이 구현되어야 하는 이유

## 2. 2PC vs SAGA 분산 트랜잭션 비교
- 2PC (Two Phase Commit)
  - 2단계로 나누어 커밋하는 방법론
    - 1. Coordinator가 각 트랜잭션 참여자들에게 (원화서버, 외화서버) Commit가능 여부를 질의
    - 2. 참여자 모두가 가능이라고 하는 경우 commit, 그 외에는 롤백
  - **강한 일관성**, but *낮은 가용성, 확장성*
- SAGA Pattern
  - 각 서비스의 작은 트랜잭션들을 실행하면서 진행하다가, 특정 단계에서 실패하면 `보상 트랜잭션` 실행
  - **높은 가용성, 확장성** but *중간 상태 노출, 보상 트랜잭션 구현 필요*
👉 높은 트랜잭션을 견뎌야하고, 이후 서비스가 추가될 수 있다는 점을 고려해 SAGA패턴으로 선택

- 2가지 SAGA Pattern
<img width="639" alt="image" src="https://github.com/user-attachments/assets/e2283bf5-c319-4073-a661-d7f713c05cac">
  - Choreography Saga
    - 중앙 제어 없이, 메시지 브로커를 통해
    - 단일 장애점이 없음, 느슨함
    - 그러나 현재 진행중인 트랜잭션 및 상태 추적이 어려움
  - Orchestration Saga
    - Orchestrator가 중앙에서 제어
    - 단일 장애점이 있으나, 상태 추적이 용이
👉 환전 서버가 Orchestrator가 될 수 있고, 둘다 추적해야했기 때문에 Orchestration 방식으로 선택

## 3. SAGA를 이용한 환전 구현
- 입출금 실패 케이스
  - 정상적인 실패 (비즈니스 적)
    - 잔액 부족, 계좌 해지
  - 비정상적인 실패 (에러)
    - 서버 에러, 타임아웃, 메시지 produce/consume 실패

- 환전의 단계 :
  - 1. 원화 출금 요청
    2. 외화 입금
   
- 환전 실패 케이스
  - 출금 실패 : 추가적인 입금 없이 실패로 마무리
  - 입금 실패 : 이미 원화가  출금됐으므로, 다시 원화를 돌려주는 보상 트랜잭션이 필요함

- 입출금 요청에서 사용한 통신 방식 (HTTP vs Messaging-Kafka)
  - 입금, 출금은 HTTP 를 사용
    - 출금 결과를 알고 입금으로 넘어가야하기 때문 (동기)
    - 유저는 환전이 즉시 완료되길 기대함
  - **보상트랜잭션 (출금취소)**는 메시징으로 구현
    - 출금취소는 마지감 과정이며, 유저가 기다릴 필요없음 (비동기)

## 4. 에러 핸들링
- HTTP Error Handling
  - 출금 요청 시 5xx에러/타임아웃이 발생했을 때 -> **출금 성공/실패 재확인** 
    - 성공 확인 : 보상 트랜잭션 발생 (출금 취소)
    - 실패 확인 : 추가적인 처리 없이 환전 실패 
  - 그러나 네트워크 문제로 재확인도 어려울 때? -> **Kafka Message Scheduler**
    - 프로듀서에서 바로 메시지 발행하지 않고, 지연시간을 추가해서 메시지 발행 -> **Kafka Message Scheduler**
    - **Kafka Message Scheduler**는 지연시간 이후 메시지 발행해서 지연 이후에 컨슈머가 메시지를 가져갈 수 있도록 함
  - 그러나, 환전 서버의 문제로 환전 지연 이벤트를 발행하지도 못하고 서버가 죽은 경우 -> **Batch를 통해 재실행**
    - 환전을 재시작
- Eventually Consistency : Consumer Dead Letter (CDL)
  - 원화계좌 (Consumer)가 출금 취소 메시지를 처리하다가 에러가 발생한 경우? -> **CDL**을 사용
  - **CDL Message Broker**가 **DL**서버로 발행, DL서버가 재처리
- Transactinal Messaging : Production Dead Letter
  - 환전 서버로 부터 출금 취소 메시지 발행을 실패하는 경우
  - <img width="419" alt="image" src="https://github.com/user-attachments/assets/79ee8839-368c-4e20-b5b5-dd1186547462">

## 5. 모니터링
<img width="784" alt="image" src="https://github.com/user-attachments/assets/92eebb15-8c6b-4ad5-aaf3-8d8ad7fb0f83">    
- 환전 로그 테이블을 생성해 모든 단계별로 추적이 가능하도록 설계
- 일정 시간 이후에도 환전이 안 끝난 트랜잭션이 있을 경우 Alert
- 원화, 외화 계좌에서는 ETL을 통해 꾸준히 정보계로 보내고 이를 모니터링

## 6. 결론 및 성과
- SAGA패턴의 장점 : 가용성
  - 회계서비스를 쉽게 추가 가능
  - 부족한 돈 자동환전 결제 서비스를 쉽게 추가 가능
