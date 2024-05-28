## Server-driven UI로 토스팀의 마지막 어드민 만들기
[토스 SLASH 23 - Server-driven UI로 토스팀의 마지막 어드민 만들기](https://youtu.be/3wxG1WLDONI?si=CEob0VxVGnSYxN-l)

## 사람에 대한 의존성
하이테크 기술이어도, 사람의 수작업에 의존성이 높은 경우가 많음
ex. 토스앱의 소비탭에서 보여주는 카테고리도 일부 자동화 + 수작업으로 관리중
*⇒ 생산성을 위해 완벽한 자동화를 꿈꾸지만, 높은 사용성을 위해서는 휴먼터치를 피할 수 없음*
⇒ 이런 순간들을 위해 비즈니스 운영, 관리, 고객대응을 하기 위한 Admin, Back Office가 필요함

## 어드민을 개발하면서 마주한 문제
1. 어드민이 너무 많음
  - 프로덕트 어드민, FDS 어드민, AML 어드민, ..등등
  - 시간이 흐르면서 다양한 기능들이 분산 되어 어떤 기능을 어디에서 관리할 수 있는지 찾기 어려움
  - 필요성에 따라 최소 비용으로만 개발된 불편한 사용성
  - 일관성 없이 파편화된 UI/UX : 어드민 사용법을 매번 새로 숙지해야 함
2. 엄격한 보안규칙을 지키기 위해 개발의 복잡도가 높아짐
  - ex. 어드민에 로그인할 수 있다고 해서 모든 정보에 제한 없이 접근할 수 있어서는 안 됨 = 개인의 업무 영역에만 접근 권한 필요, 권한에 대한 정보 및 기록
  - 어드민 개발하는 구성원들이 이런 보안 정책을 모두 숙지하기 위한 노력과 시간이 필요
3. 문화와의 호환성
  - 작은 목적 조직이 각 제품에 대한 온전한 권한을 갖기에 어드민도 각 목적 조직에서 개발하여 **중복 개발** 발생
  - 목적을 다하면 해체하는 조직이 있음에도 불구하고 남아있는 어드민..

> 토스팀의 조직과 문화를 유지하면서 안전하고 편리한 어드민을 쉽게 만들 수 있는 방법?   
= 어드민 개발을 위한 Platform as a Service(PaaS)를 개발

## Admin Platform as a Service
![IMG_12AA03E9F031-1](https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/bdadff7c-4c35-40a2-b869-b2d854452c62)

**서버 개발자가 입력한 UI DSL을 실제 화면으로 렌더링 = Server Driven UI**
- Server Driven UI란?
  - UI를 기술하는 DSL + DSL을 렌더링하는 프론트엔드 엔진

### UI 기술 DSL
#### 테이블 형태의 DSL 및 UI 예시
![image](https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/46ef732f-a898-4b23-b417-f7139c230bbd)

#### input form 구현 DSL 및 UI 예시
![image](https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/a7b1bc8f-891a-41d0-85e7-25a3868c8f22)

### DSL을 렌더링하는 프론트엔드 엔진
- props를 받아 리액트 node를 반환
- 지원하는 dsl.type이 추가될 때마다 이를 리턴해주는 함수를 수정해야할 가능성
  - 의존성 주입으로 UI엔진 코어 로직 / 개별 컴포넌트 분리
- 모노레포로 하나의 코드베이스안에서 효율적으로 구현
