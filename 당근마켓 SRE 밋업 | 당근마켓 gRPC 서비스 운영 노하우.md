#  당근마켓 gRPC 서비스 운영 노하우 | 당근마켓 SRE 밋업 1회 
[ 당근마켓 gRPC 서비스 운영 노하우 | 당근마켓 SRE 밋업 1회 ](https://www.youtube.com/watch?v=igHrQPzLVRw)


## 당근 마켓의 과거/현재
- 현재 당근 마켓
  - http (non gRPC) 75k rps
  - gRPC 60k rps
  - 5개의 대표 MSA 구성 (Java, Ruby, Go, NodeJS, Python)
  - 점점 gRPC 서비스들이 늘어나는 중
 
- 레버리지를 잘 사용하자
  - 레버리지가 뭐지..?
  - *SRE(사이트 신뢰성 엔지니어링, Site Reliability Engineering)에서 "레버리지(Leverage)"라는 용어는 보통 효율성이나 효과성을 극대화하기 위한 도구나 방법론을 의미*
  - 효율적인 도구, 방법론, 자동화를 통해 생산성 향상 -> 자동화 스크립트, CI/CD 파이프라인, 모니터링 및 알림 시스템 구축
- 트래픽 플로우를 이해하려는 노력
  - Connection이 있는 ALB
  - Active Flow가 있는 NLB
  - 각 구간별로 어떻게 패킷이 오고가는지 알기
  - 사용할 프록시에 대해 이해 lptablesL ipvs, 사용할 cni 통신방식 이해
  - 쿠버네티스 네트워크 방식 이해
  - 각 구간별 메트릭 준비 (이슈 발생구간을 빨리!)
- 동일 환경을 가진 sre-test-server를 구축
  - 네트워크 이슈 디버깅 (디버깅 모드로 동작)
  - 변화 선제 적용
- gRPC 인프라 구성
  - haProxy -> NLB -> Istio Ingress/ Service Mesh
 
## gRPC 
- protobuf를 이용해 JSON 보다 성능
- latency
- 서비스 인터페이스 스키마
- 내장 기능 풍부 (인증, 암호화)

### gRPC 클라이언트 서버
- HTTP2를 이용한 gRPC 채널이라는 대표적인 컨셉
  - HTTP/2 엔드포인트에 대한 virtual connection을 나타냄
- 클라이언트가 gRPC 채널을 만들면 내부적으로 HTTP2 커넥션 생성됨
- RPC는 HTTP2의 스트림으로 처리 됨
- 메시지는 HTTP2의 프레임단위로 처리가 됨
- 채널에는 많은 RPC가 있을 수 있고, RPC에는 많은 메시지가 있을 수 있음
- HTTP2의 스트림은 Multi Concurrent Conversation을 지원하기 때문에 싱글 커넥션에서 가능
- 채널은 사용자가 표면적으로 손쉽게 메시지를 보낼수 있는 쉬운 인터페이스 제공
- **채널은 하나의 엔드포인트에 대한 virtual 커넥션을 표시하지만, 실제로는 많은 http2 커넥션에 의해 지원될 수 있음**
- gRPC 클라이언트는 resolver와 LB를 들고 있음
  - 리졸버는 주기적으로 타겟 DNS를 리졸브하면서 엔드포인트들을 갱신
  - 커넥션이 실패하면 로드밸런서는 직전에 사용했던 address 리스트들을 사용하여 재연결을 시작
  - -> 인프라 운영에 안정적인 복원력(레질리언스)을 지원
  - 커넥션 풀을 관리
- gRPC Keepalive
  - KeepALive HTTP/2 ping 프레임을 보내 연결상태를 주기적으로 확인하고, 커넥션을 살리는 역할을 함
  - ping frame 전달 시 응답이 제때 오지 않으면 연결 실패로 간주하고 커넥션 클로즈
  - 프록시 서버들은 보통 리소스를 절약하기 위해 idle connection을 종료시키기도 함, 타임아웃보다는 적은 시간으로 keepalive 세팅
  - gRPC가 연결에서 주기적으로 핑 프레임을 전송하면 해당 connection은 not idle 상태가 됨
- gRPC 요청과 응답 프레임 디버깅 모드에서 확인해보기
  - 요청 (헤더 + 바디)
  - 응답 (헤더 + 데이터바디:length prefixed Message + 트레일러 헤더)
    - 트레일러 : http 스펙중 일부로 grpc status, grpc-message를 포함함
    - 어플리케이션에 응답을 잘 주었는지 확인하기 위해 체크
    - **gRPC Status Code** : 어플리케이션 레벨, 인프라 레벨에서 핸들링해야하는 status가 분리되어 있음
      - http 응답으로 빈 리스트가 왔을때 자원이 없으니 404..? 정상이니 200...??? -> 이런 고민들이 보강됨
    - 프로메테우스의 다차원 메트릭을 사용해서 error 모니터링
    - 개발자 레이어에서는 커스텀하게 서비스에 맞는 대시보드를 세팅

### gRPC 로드밸런싱
- 스트림 베이스의 밸런싱, L7베이스의 밸런싱이 필요하나, *쿠버네티스는 L4*이기 때문에 gRPC 요청분배에 있어 잠재적 문제임
  - L4 : 커넥션 베이스드, L7 : 스트림 베이스드
- 요즘 proxy들은 k8s gRPC 서비스를 로드밸런싱할때, 서비스 객체를 엔드인트 주소 갱신용도로 사용함


### SRE로서 해야할..
- 모니터링 전역 쿼리 디자인
- 트래픽 플로우 관련 구간별 주요 메트릭 보강
- 인프라 구성을 고려한 gRPC 옵션 가이드 for 개발자

### 개발자로서 해야할..
- 서비스의 특성에 맞는 option 적용
  - keep-alive, max timeout,... 등
- Intercepter 패턴 활용, AOP 로깅, 모니터링
- 각 서비스를 서로 보호해줄 수 있는 DeadLine 설정

### 초기 세팅시
- Connection test하면서 Reflection 넣기
- k6, Ghz를 활용한 Load test

## 함께 일하기
- 실수 할 수 있는 환경을 만들기 -> 멋있는 마인드.. 나도 같이 일하고싶다.. 🥹
- 클라우드에서 rollback, resilience에 초점을 둔 환경설정


