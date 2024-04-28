# Java의 미래, Virtual Thread
[ [4월 우아한테크세미나] ‘Java의 미래, Virtual Thread’ ](https://www.youtube.com/watch?v=BZMZIM-n4C0)

1. Virtual Thread란?
2. Virtual Thread 동작 원리
3. 기존 스레드 모델 서버와 비교
4. 성능테스트
5. 서비스 적용 시 주의사항

# Virtual Thread란?
### 배경 : 전사보급을 목표로하는 GateWay 시스템 개발 계획
- 인증과 같은 전처리를 먼저 처리하고 뒷단의 다른 서버로 보냄
-  → 안정성, 처리량에 대한 고민
-  선택지 : 코틀린 코루틴 & 자바의 프로젝트룸
-  → 당시에 프로젝트룸이 정식 피쳐가 아니었기 때문에 코틀린 코루틴을 선택

### Virtual Thread
- 스레드 생성 및 스케쥴링 비용이 기존 스레드보다 저렴
- 스레드 스케줄링을 통해 논블로킹 I/O지원
- 기존 스레드를 상속하여 코드 호환

1. 스레드 생성 및 스케쥴링 비용이 기존 스레드보다 저렴
  - 기존 자바 스레드는 생성 비용이 크다
    - 스레드 풀의 존재 이유
    - 사용 메모리 크기가 크다. 최대 2M까지 차지
    - OS에 의해 스케쥴링 됨 = 항상 OS와 통신하기 때문에 시스템 콜 발생 : 시스템 콜 오버헤드 존재
  - VT는 생성 비용이 작다
    - 스레드 풀 개념이 없고 필요할때마다 생성 및 종료
    - 사용 메모리 크기가 작다 (최대 50KB)
    - OS가 아닌 JVM내 스케쥴링

  - 진짜 빠른가? 실험 해보자!
    - 아무 작업도 하지 않는 스레드 100만개를 기존 스레드 vs 버츄얼 스레드로 생성해봄
    - 기존 스레드 : 31초 정도 VS 버츄얼 스레드 : 0.375초정도
2. Non Blocking I/O 를 지원한다
- 블로킹 타임을 획기적으로 줄여준다. -> 요즘과 같은 MSA 환경에서 중요
  - cf. Spring WebFlus & Netty : 이벤트루프를 통해 블로킹 타임을 줄임
- VT에서 Non Blocking I/O가 가능하게 된 개념
  - JVM 스레드 스케쥴링
  - Continuation 활용
3. 기존 스레드를 상속한다.
- `VirtuaThread`는 `BaseVirtualThread`를 상속하고, `BaseVirtualThread`는 `Thread`를 상속한다.
- ExecutorService도 그대로 사용했던 것처럼 사용하면 됨 → 런닝커브가 낮음

# Virtual Thread 동작 원리
- 일반 자바 스레드 특징
  - 플랫폼 스레드 (유저 스레드)
  - OS가 스케쥴링
  - OS의 커널 스레드와 1:1로 매핑
  - 작업단위 : Runnable

- 어플리케이션은
  - 커널영역(OS) + 유저영역(JVM)으로 이루어짐
  - 유저영역에서 커널영역을 사용하려면 JNI(Java Native Interface)를 통함
  - 커널 스레드 : 플랫폼 스레드 =  1:1
  - 플랫폼 스레드 생성시에 유저 영역에 객체 생성 후 스타트 할 때 JNI를 통해 커널스레드 생성을 요청함
 
### Virtual Thread의 특징
- 가상 스레드
- JVM에 의해 스케쥴링
  - Thread.start() -> submitRunContination() : 작업 스케쥴링
    - `scheduler`, `runContination` 필드 존재
    - scheduler : 가상 스케쥴러 -> 모든 버츄얼 스레드가 동일한 스케쥴러를 공유
    - 프로세서 수의 캐리어 스레드
    - Work Stealing방식으로 작업을 수행 : 본인의 워크 큐가 비어있으면 남의 워크큐를 뺏어와서 작업
- **JVM 스케쥴링의 이유?**
  - VT는 커널영역 접근 없이 단순 자바 객체 생성, *생성/실행 시 시스템 콜이 발생하지 않는다. * -> 실행시간이 빠른 이유
- 새로운 작업단위 `Continuation`
  - 코틀린 코루틴도 Continuation를 활용함, Caller가 코루틴을 호출할 때 function과 다르게 모든 라인을 수행하는것이 아니라 Suspend지점을 만나면 중단하고 돌아갔다가 그 지점으로 다시 돌아옴
  - **Continuation**
    - 실행가능한 작업흐름
    - 중단가능
      - 실행하다가 중단지점을 만나면 작업을 중단하고, Stack메모리에 기억 -> heap으로 이동시킴
    - 중단 지점으로부터 재실행 가능
      - 다시 실행할 때 heap -> stack -> 스택 포인터를 기준으로 다시 실행하게 됨
  - 
