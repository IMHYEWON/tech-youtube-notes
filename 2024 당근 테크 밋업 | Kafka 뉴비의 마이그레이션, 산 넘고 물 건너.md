# 2024 당근 테크 밋업 | Kafka 뉴비의 마이그레이션, 산 넘고 물 건너
[Kafka 뉴비의 마이그레이션, 산 넘고 물 건너 | 2024 당근 테크 밋업](https://youtu.be/zheA2qdyCnM?si=LW0Cb3zkcJe4M6Et)

1. Context
2. Static Broker Host
3. 점진적 전환
4. 메시지 동기화
5. Kick Off the Migration


# Context
<img width="513" alt="스크린샷 2024-12-15 오후 7 33 10" src="https://github.com/user-attachments/assets/e9d16171-e2a2-4903-b482-97ba52010854" />

- 전사 적으로 여러 이벤트를 처리하기 위한 메시지 큐로서 카프카를 사용하고 있었음
  - 사용 현황은 토픽 1100개, 컨슈머 멤버수 2000개, 일일 메시지 발행량 수백만, 13개 클러스터
  - 클러스터를 용도별로 (서비스) 별로 분리해서 사용함 → 서비스, 광고, 데이터
    - 분류 별로 사용된 세트 (서비스, 광고, 데이터)가 여러 글로벌 리전에도 있음
    - → 운영에 대한 부담감
- 별도 규칙 없이 사용되고 있어서 여러 문제점발생   
  - 1) 스키마 레지스트리가 없어 메시지 포맷이 다르게 발행될 때 컨슈밍시 호환이 안돼 문제
    2) 토픽 네이밍 컨벤션이 없는 문제
    3) 별도 토픽에 관한 권한 관리가 없음 → 내부 카프카 클라이언트라면 모든 토픽에 접근 가능
  - → 정책이 정리된 신큐카프카 클러스터 생성
    - Schema Registry를 정의해서 IDL 레포테 정의하게 함
    - Topic Naming Convention
    - SASL
  - 레거시 클러스터가 점점 fade out 되길 희망했으나 쉽지 않음 🥲
    - 각 팀에서 신규 클러스터로 마이그레이션이 어려움
    - → 결국 운영 비용이 2배
    - → `SEAMLESS MIGRATION` 목표 수립 - 클라이언트 변경을 최소화해서 클라이언트팀의 마이그레이션 비용을 최소화

# 첫번째 난관, Static Broker Host
- bootstrap.servers : kafka클러스터의 초기 연결을 설정하기 위해 사용하는 host:port의 리스트
  - ASIS : 이 config값을 정적인 Config로 서비스별로 주입해서 사용하고 있었음
    - 이 값을 변경시에 일일히 변경하는 작업비용 소모
  - 👉 *클라이언트 SDK에서 장소 투명성을 제공할 수 없을까?*
    - 하지만, 당근에서는 Go, Kotlin, Ruby, Typescript, Python등 여러 언어로 개발하고 있고 모두가 SDK가 다르기 때문에 어려움
  - 👉 **모든** **클라이언트를 지원할 수 있는 장소 투명성을 제공할 수 없을까?*
    - 카프카 프로토콜 (binary)이기 때문에 모든 언어의 SDK는 이에 맞춰야 함
  - **카프카 프로토콜** 
    - <img width="512" align="center" alt="image" src="https://github.com/user-attachments/assets/edaa2c38-0932-4d02-9811-624bd6b4112d" />    

  - Format : 메시지 형식 = [사이즈 필드 + (헤더+메시지) 필드]
    - 카프카 클라이언트와 브로커의 통신 프로세스 (초기 시퀀스 과정)
      - 1. 클 → 브 : API 리퀘스트 (버전 확인)
        2. 클 → 브 : 메타 데이터 리퀘스트 (클러스터의 토픽 정보 등)
        3. 클 → 브 : FindCoordinator 요청 (컨슈머 코디네이터)
        4. 이후 토픽 프로듀싱 & 컨슈밍
    - 👉 *이 초기 시퀀스 과정을 Kafka 브로커와 클라이언트가 직접 통신하는게 아닌 중간에 프록시가 있다면 어떨까?*
  - <img width="508" alt="image" src="https://github.com/user-attachments/assets/dd108967-738c-472e-be9d-d39fc6bafc6b" />
  - TOBE : 초기 연결에만 관여하는 카프카 와이어 프록시가 레거시/신규 클러스터를 결정해줌 
    - <img width="484" alt="image" src="https://github.com/user-attachments/assets/8c5cf95e-82a9-4743-97b6-f0e1a9efd10c" />


# 두번째 난관, 점진적 전환
- Issue : 하나의 토픽을 여러 컨슈머 그룹이 구독하기 때문에 여러 팀이 관리함. 토픽이 이전하면 모든 컨슈머도 이전해야되지만 한번에 전환하기엔 부담스러움
  - 👉 *점진적으로 전환할 수 없을끼?*
- 카프카 메시지 헤더에는 apiKey, apiVersion, **clientId**, correlationId, headerVersion 등 여러 버전을 가지고 있음
  - `client.id` : 컨슈머와 프로듀서를 식별하기 위한 메타정보로서, 환경변수로 주입할 수 있음
  - kafka client id 정책 수립
    - <img width="300" alt="image" src="https://github.com/user-attachments/assets/1b64815d-2d6f-41ee-8be5-e4921c3e5694" />

|Legacy|New|
|:---:|:---:|
|<img width="514" alt="image" src="https://github.com/user-attachments/assets/1459cad4-b5bb-4f3d-a488-a0d8f64feafd" />| <img width="498" alt="image" src="https://github.com/user-attachments/assets/129669f8-af1b-4f99-85a5-fcea18bf0d75" />|


# 세번째 난관, 메시지 동기화
- Issue #1 : Topic Configuration
  - 토픽 파티션 수, replication 수 를 Legacy와 new 클러스터가 일치해야 함
- Issue #2 : 메시지 & 오프셋 동기화
  - 카프카 메시지 동기화도 필요하지만 오프셋도 동기화가 필요함 !
  - <img width="507" alt="image" src="https://github.com/user-attachments/assets/0fc06555-aa8d-4f94-b917-1af28df8ceaa" />
- Issue #3 : 토픽이름의 변경
  - 토픽이름 정책에 맞게 토픽을 변경
- 👉 `카프카 미러메이커2` 사용
- **카프카 미러메이커2?**
  - 아파치 카프카 오픈소스
  - Kafka Connect 기반 -> 커넥터의 구현체
    - MirrorSourceConnector : 메시지 동기화
      - 신규 클러스터는 오프셋이 다르게 설정됨 (legacy : 300, new : 0)
      - → 미러메이커가 인터널 토픽으로 관리
    - MirrorCheckPointConnector : 소비자(클라이언트) 컨슈머 그룹의 오프셋 동기화를 위해 사용
      - 컨슈머 그룹은 기본적으로 `_consumer_offsets`를 확인해서 오프셋을 동기화 하기 때문에 클러스터간 오프셋 동기화를 하기위해서는 `checkpoints.internal` 를 확인해줘야함
      - → 이런불편함을 없애기 위해 `_consumer_offsets`로 바로 커밋하는 옵션 적용 `sync.group.offsets.enabled`
  - 클러스터간 데이터 복제 기반
- Renaming topic 이슈
  - 미러메이커2에서 renaming topic관리 정책 인터페이스를 사용가능 (ReplicationPolicy)
    - DefaultReplicationPolicy : Alias
    - IdentityReplicationPolicy : 매핑 관리(?)
  - → 이름을 바꾸는건 지원되지 않음
    - 삽질 #1 : RouterRegex - 토픽이름은 변경 가능하지만 offset 동기화 불가
      - <img width="517" alt="image" src="https://github.com/user-attachments/assets/181ab645-68a7-426c-a768-421983e1f238" />
      - 토픽 파티션 내부에서 사용되는 이름은 변경되지 않는것 ! 메시지 동기화를 하기 위해서는 내부파티션 토픽 네임도 변경해야함
    - 👉 ReplicationPolicy 인터페이스를 구현해서 별도의 구현체
      - 소스 토픽의 프로퍼티를 타겟 토픽의 프로퍼티에 주입함 → 이름 변경 가능
      -  👉 당근미러메이커extension이라는 도커 이미지를 빌드해서 미러메이커를 당근미러메이커로 사용하도록 변경
      -  <img width="485" alt="image" src="https://github.com/user-attachments/assets/4d8515b1-99c6-46c8-ad5d-949a243f4609" />

# Kick the Migration with Kafka Wire Proxy + Mirror Maker 2
- 마이그레이션 준비작업
  1. 마이그레이션 대상 토픽 & 컨슈머 그룹 선정
    - 제로 다운타임을 위해 변경시 과도기적인 시점 존재
    - **컨슈머에서 멱등성 보장될 것** - 중복메시지 발생 가능성 
    - **메시지 순서 보장이 필요하지 않은 경우** - 메시지 순서 영향 가능성
  2. 미러메이커를 사용하여 메시지 동기화 시작 
    - 레거시, 신규 클러스터 정보를 helm chart로 정리해두어서 쉽게 싱크
  3. Kafka Wire Proxy - 라우터 설정 정보 추가
    - <img width="467" alt="image" src="https://github.com/user-attachments/assets/be232a17-99d1-4ea6-8f80-b02843bc76c7" />
  4. 카프카 클라이언트 설정 변경 및 재시작
    - 부트스트랩 서버를 와이어 프록시 가리키도록 수정
    - 컨슈머의 경우 리네임 토픽 명추가
    - 클라이언트ID를 Naming 룰에 맞춰변경

# In the Future (이후 과제)
1. Migration 성공적으로 마무리 
2. Kafka Wire Proxy 고도화 
   - Disaster Ricovery
   - Major Upgrade
3. Kafka as a service 고도화
   - Topic Management
   - Managed Consumer
   - Easy Alert
  
  
