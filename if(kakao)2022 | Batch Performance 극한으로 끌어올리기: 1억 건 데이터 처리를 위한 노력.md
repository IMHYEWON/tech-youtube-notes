# Batch Performance 극한으로 끌어올리기: 1억 건 데이터 처리를 위한 노력 / if(kakao)2022 

1. 대량 데이터 READ
2. 데이터 Aggregation 처리
3. 대량 데이터 WRITE
4. Batch 구동 환경
5. 대량 데이터 처리 방식 총정리

## When? 개발자들은 언제로 처리할까요??? 
ex. 오후 4시에 상품 주문 배송정보를 고객들에게 문자로 일괄전송해야하는 구나.. Batch로 개발해서 스케쥴 걸어놔야지!   
→ 서버개발자들이 자주 사용하는 방식으로, 부담 감소

### Batch 개발 케이스
1. 일괄 생성 (READ → CREATE → WRITE) : 주문 정보 조인&조합 → 새로운 정보 생성
2. 일괄 수정 (READ → UPDATE → WRITE) : 기존 정보 참고해서 이미 있는 정보 
3. 통계 (SUM READ → CREATE → WRITE) : Group By, SUM READ → 

## How? Batch Performance 개선
- 개발환경 : SPring Batch, MySQL, SCDF, Redis, 쿠버네티스

## 1. Batch 성능 개선의 첫 걸음 = Reader 개선
- 10억개중 환불이 발생한 100만개 검색..

### Chunk Processing -> Zero Offset Reader
- 대량 배치에서는 항상 Chunk Processing
- 이와 함께 Pagination Reader를 자주 사용했으나.. (limit, offset) → *대량 처리에 매우 부적합..*
- ✅ MySQL에는 offset이 커질때 부담스러워한다 (5천만번째 데이터를 찾을때)
- 💡 이런 부담을 해소하기 위해 매번 Offset을 0으로 유지하는 Reader 개발
- `select * from orders where category='BOOK' and id > 50000 limit 0, 100
<img width="607" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/60f42fee-a861-4b96-80b6-79457d0bad81">

### Cursor 방식 사용
- 데이터가 없을 때까지 일정 개수의 데이터를 반복해서 제공
- SPring Cursor 지원하는 ItemReader 두가지
  - JpaCursorItemReader : MySQL 커서방식이 아니고, 데이터를 모두 읽어서 서버에서 직접 커서하는 방식 -> *OOM*유발
  - JdbcCursorItemReader, HibernateCursorItemReader : MySQL 커서방식처럼 일정개수만크 Fetch하는 방식 → 안전
    - → 그러나 HQL, Native Query 쌩쿼리를 사용해야 함
    - → **Exposed** 사용
      - Kotlin 기반 ORM 프레임워크
      - **쿼리는 ExposedDSL + 동장방식 JdbcCursorItemReader**
- 100만개, 300만개 부터 ItemReader 성능이 크게 차이
  - JpaCursorItemReader 는 갑자기 시간이 늘어나 예측하기 어려움
  - JdbcCursorItemReader는 선형적으로 늘어나 예측이 쉬움
 
## 2. 데이터 Aggregatoion 처리
- SUM 쿼리에 의존하는 기존 Batch의 문제점
  - 여러 테이블 조인, group by
  - JOIN + GROUP BY + SUM 의 문제점
    - 연산과정이 쿼리에 의존적 → DB 부하증가
    - 데이터 누적 → 카디널리티 변경, 쿼리 실행계획 변경, 쿼리 튜닝 난이도 상승
  - 쿼리 튜닝을 위한 과도한 인덱스 추가
    - INSERT, UPDATE 성능 저하
    - 저장 용량 차지  

### GROUP BY를 포기하자 ⇒ 새로운 아키텍처, Redis를 활용한 SUM
<img width="584" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/868d966e-13ed-4791-907c-2a62fc27ad70">

- GROUP BY 사용하지 말고 직접 Aggregation하자 ! 
- **이게 가능해?** ⇒ 📌새로운 아키텍처 만들자
<img width="664" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/e02538ab-4966-4ab2-bece-4b85f9a8fb8a">

### 그러나 REDIS를 도입해도 해결되지 않는 문제
- Redis 연산은 빠른데, I/O가 너무 많아
- *Redis PipleLine* 으로 처리
  - 다수의 command를 한 번에 묶어서 처리
  - Batch App전용 Redis Pipeline 대량 처리 라이브러리를 별도 개발
  - Spring data redis로는 어려움

## 3. WRITER 개선
- **Batch Insert** > 일괄로 쿼리 요청
- **명시적 쿼리** > 필요한 컬럼만 Update, 영속성 컨텍스트를 사용하지 않음 → JPA 안녕 ~

### Batch에서 JPA WRITE에 대한 고찰 : BAtch 환경에서 JPA가 과연 잘 맞는가?
- Dirty Checking과 영속성 관리 -> Dirty Chekcing과 영속성을 사용하지 않도록 JPA버리거나 Projection사용
- UPDATE할 때 불필요한 컬럼도 UPDATE -> JPA Dynamic UPDATE가 있긴 하지만 성능이 안좋음
- JPA Batch Insert 지원이 어려움 -> I/O가 너무 많아 큰 성능 저하

## 4. Batch 구동
- crontab, 에어플로우, 젠킨스.. -> 스케줄 관리 모니터링이 맘에 안듬
- Spring Cloud Data Flow 도입 : Stream, Task 지원
  - K8s와 완벽한 연동으로 Batch 실행 오케스트레이션
    - 다수의 Batch가 상호 간섭없이 컨테이너에서 러닝
  - 스프링 배치와 완벽한 호환, 시각적으로 모니터링 가능
    - 자체 대시보드, 그라파나 연동 가능 


