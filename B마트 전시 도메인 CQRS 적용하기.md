## B마트 전시 도메인 CQRS 적용하기 (#우아콘 2021)
[B마트 전시 도메인 CQRS 적용하기](https://youtu.be/fg5xbs59Lro?si=4Uhb4i0PjjmzkBIF)

## B마트 들어가기
### B마트 상품의 데이터구조
<img width="564" height="377" alt="스크린샷 2025-07-20 오전 1 47 43" src="https://github.com/user-attachments/assets/3fa080d5-77b2-4a56-9899-b0aabe93d3cf" />
<img width="598" height="395" alt="스크린샷 2025-07-20 오후 11 50 55" src="https://github.com/user-attachments/assets/f3dbf5df-074b-48bc-ba12-5eda5af9adf9" />

- 일반적으로 정규화된 DB스키마와 전시 도메인에서 사용자에게 노출되는 정보의 구조는 매우 다름
  - B마트의 경우도 지점, 카탈로그, 상품 정보를 각각 입력 및 수정하고 이를 매핑하는 테이블에 최종적으로 업데이트함
  - 반면, 전시 도메인에서는 이를 복합적으로 보여주기 위해 다시 비정규화하는 과정이 필요함

## CQRS란?
- Command and Query Responsibility Segregation
  - 명령과 조회의 책임을 분리한다.
- 서비스 초기의 단순한 아키텍처에서 명령하는 부분(어드민, 배치 등을 통해 C/U/D)과 조회하는 SELECT 부분은 비즈니스 로직은 분리되긴 해도 DB에 접근하는 도메인 모델은 보통 하나의 모델이 같이 수행하게 됨
- 그러나 서비스가 성장해가면서, 실제 사용자 조회시에는 사용되지 않는 **내부 관리용 데이터**들이 생기게 됨
  - 관리용 이름, 디폴트 상품맥, 사은품 정보, 관리용 카테고리 및 코드 등
- 또한, 노출정책/재고정보/위치정보 등 외부로부터 전달받는 데이터들도 생김
<img width="490" height="291" alt="image" src="https://github.com/user-attachments/assets/2cf523f6-dcb2-4ebb-ac47-8fa0ae9b5d71" />

- 조회로직과 명령로직이 같은 모델을 사용하며 영향을 주는 일이 생김
- 히스토리를 알지 않는 경우 자칫 중요한 도메인을 건드리게됨
<img width="541" height="299" alt="image" src="https://github.com/user-attachments/assets/3cf1214f-4636-492f-bafa-d8e2ad4220a8" />

- 따라서 명령 도메인과 조회 도메인을 분리하는 것이 적절한 CQRS의 방향

## CQRS구현하기
### 1. 모델 분리하기
<img width="491" height="288" alt="image" src="https://github.com/user-attachments/assets/fa3ca8da-0bf7-4e23-b520-d09b0865c7f0" />    


- 복잡한 구조를 분리하기 위해 공유하고 있던 교집합 모델을 분리한다 
  - 명령모델 & 조회모델로 구분하며, 명령모델은 일반적인 기존 도메인 모델
  - 명령모델 : 지점, 카탈로그, 할인, 정책, 상품, 아이템 등
  - 조회모델 : 지점, 카탈로그, 상품 → 전시 및 광고 도메인에서 사용하는 비정규화 모델
    - → 이 과정에서 *조회모델을 생성하는 별도의 로직*이 생기게 됨
    - → 최적화된 스키마를 비정규화하는 과정이 생긴다는 것 = **성능상 이슈**가 생기게 됨
  - **생성, 저장과 조회 시점을 분리**
    - → 명령모델을 통해 데이터 변경 이벤트 발생시 조회모델(비정규화된 데이터)을 생성해 저장
    - CQRS에서는 조회시 조인이나 연산을 권장하지 않고 그대로 사용하는 것을 권장하기 때문에 NoSQL 저장소를 사용하는 경우가 많음
    - 명령이 적고 조회가 많은 서비스에서 이점이 있음
    - DB에서는 레디스를 활용하고 있음
### 2. 책임과 성능에 대한 부담 해소
- 기존의 도메인 모델에서는 조회 모델의 생성과 저장의 책임도 떠안게 되는 것
<img width="497" height="319" alt="image" src="https://github.com/user-attachments/assets/91296759-7799-49eb-83ab-2a967471d001" />

- 이벤트 소싱 패턴을 통해 부담을 해소함
  - CQRS를 도와줌, 명령모델에 쌓이는 부하를 해소해줌

### 3. 언제 CQRS를 적용하는 것이 좋을까? 
- UX와 비지니스 요구 사항이 복잡해질 때
- 조회 성능을 보다 높이고 싶을 때
- 데이터를 관리하는 영역과 이를 뷰로 전달하는 영역의 책임이 나뉘어져야 할 때
- 시스템 확장성을 높이고 싶을 때

## B마트의 CQRS 적용
- 앱을 통해 고객에게 카탈로그와 상품을 전시하는 전시도메인에만 CQRS패턴을 적용함
  - CQRS는 필요한 일부 도메인에만 적용할 수 있음
- B마트 메뉴화면
  - 가장 복잡한 비정규화 로직을 갖고 있음
    - 노출 가능한 카탈로그만 보여주는것이 가장 중요한 로직이었음, 하위 매핑된 모든 카탈로그 및 카테고리와 상품에 대해 체크해야하는 전체 트리를 체크해야하는 로직이 생김
    - ASIS : 전체 트리를 레디스에서 캐시해서 사용했었음
   
    - 
