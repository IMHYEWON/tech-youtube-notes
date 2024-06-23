# 서킷브레이커 사용 방식 개선하기
[ 서킷브레이커 사용 방식 개선하기 | 당근 SERVER 밋업 2회 ](https://youtu.be/ThLfHtoEe1I?si=65vkq867rq3TkS-s)

## Circuit Breaker
- 요청 대상이 정상(CLOSE)일 때 계속 요청을 흘려보내지만 비정상일 땐 빠른 실패를 받기 위해 사용함

## Spring Cloud Circuit Breaker
- Resillience4j
- Spring Retry

### 사용방법
1. circuitBreaker Interface factory로 생성해서 구현 
- main 블록을 감싸서 실행  
- fallback 함수 정의  
<img width="336" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/5f8351a9-ea11-454b-b691-8304d7b53d35">
  
2. Spring AOP 활용
<img width="438" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/f75c388a-4aee-44ef-931d-7d85e08534ec">

- @CircuitBreaker annotation으로 이름과 fallback메소드 이름 지정  
- fallback 메소드로 넘어오는 case
  - main 블록에서 exception이 발 생한 경우 -> fallback 메소드로 그대로 전달
  - exception을 확인해서 default 처리를 하거나 클라이언트로 그대로 내리김
  - Circuit이 OPEN 됐을 때 -> main블록을 수행하지 않고 바로 fallback 메소드 수행
    - 서킷이 오픈된 경우만 디폴트처리를 해주고 그 외에는 그대로 exception을 내려주는 등의 처리 가능
<img width="420" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/eb5f639f-49e2-4570-827b-c390cf9ef338">

## Spring AOP를 활용했을때의 문제점
1. 런타임 예외 가능성 -> fallbackmethod를 String으로 이름으로 적기 때문에 컴파일 단계에서 알기 어려움
2. 낮은 응집도 -> main block의 fallback 메소드를 이름으로 찾아야하는 번거로움
3. Open Fallback 처리의 번거로움 -> 일반적인 예외상황 외에 서킷 오픈 케이스만 fallback처리하고 싶은데 일반 예외상황과 분리해야 하는 번거로움
4. 구현체를 알아야함 -> 이 방식을 활용하기 위하여 Resillience4j라는 구현체의 활용법을 알아야 함
5. 내부함수 호출 불가 -> Spring AOP가 가지고 있는 기본적인 문제
<img width="420" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/f1f3b696-5747-41ec-b508-ad957f061326">

### Circuit Breaker를 동작하기 위해 필요한 것을 다시 확인해보자
1. 메인 블록을 서킷 브레이커로 감싸줄 주체,
2. 서킷 브레이커의 이름
3. fallback 메소드

## 문제점을 해결하기 위해 구현하기 원하는 최종 모습 (디자인)
<img width="503" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/ce85fd9b-d0e3-4d98-9db2-f27b48e7510b">
<img width="503" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/48e404b4-4d9b-440d-abc6-add34ec08050">

- **메인블록을 서킷 브레이커로 감싸주는 주체가 어노테이션이 아닌, 코틀린 function**
- 서킷 브레이커 이름은 함수로 전달
- **fallback메소드는 별도로 선언하는 것이 아닌 메인함수의 체이닝메소드**로
- 서킷브레이커가 오픈됐을때만 실행되는 함수도 있으면 좋을듯
- 모든 API마다 서킷 브레이커가 달려있었기 때문에, 메소드명을 명시하지 않으면 API명을 서킷브레이커이름의 디폴트로
- 어노테이션을 사용하지 않는 김에, 여러 서킷브레이커 블록을 한 메소드 내부에서 같이 실행

## 구현 시작
1. circuit 구현
CircuitBreaker 인터페이스 & 구현체 구현 (SpringCloudCircuitBreakerFactory 한번 래핑)

- 기본 구현체
``` kotlin
class StandardCircuitBreaker (
val factory: SpringcloudCircuitBreakerFactory
): CircuitBreaker {
  override fun <> run (name: String, block: () -> T): T {
    val circuitBreaker = factory.create (name)
    return circuitBreaker.run (block)
}
```

- Circuit Open됐을때
``` kotlin
class StandardCircuitBreaker (
  override fun <T> run (name: String, block: () -> T): T {
    try {
      val circuitBreaker = factory.create (name )
      return circuitBreaker.run (block)
    } catch (e: Exception) {
      if (e is CallNotPermittedException) throw CircuitOpenException ( )
      else throw e
    }
  }
}
```

- CircuitBreaker Interface를 이용한 circuit 함수 (name을 전달받음)
``` kotlin
fun <> circuit(
  name: String = apiPath(), // default값 설정
  circuitBreaker: CircuitBreaker = DefaultCircuitBreakerProvider.get(), //디폴트 서킷브레이커
  block: () -> T,
): T {
   return circuitBreaker.run(name, block) 
}
```

- DefaultCircuitBreakerProvider Class : 스프링 뜰 때 싱글톤 서킷하나 만들어서 가지고 있다가 필요할 때 전달
``` java
class DefaultCircuitBreakerProvider (
  val circuitBreaker: CircuitBreaker
) {
  // singleton
  fun get () = circuitBreaker
}
```

2. fallback 구현
- 처음 Circuit 메소드 구현은 제네릭 T를 리턴해줬는데, fallback 메소드 체이닝을 위해 수정이 필요
- Kotlin 내장 라이브러리인 Result를 이용
  - 내고 싶은 응답이 정상이면 Result(), 그렇지 않으면 Result(Failure(Exception)) 리턴
  - isFailure, isSuccess 같은 헬퍼 메소드를 제공
  - circuit 의 응답을 Result클래스로 래핑해서 반환
  
``` kotlin
fun <T> circuit (
  //...
): Result<T> {
  try {
    val result = circuitBreaker.run(name, block)
    return Result(result)
  } catch (e: Exception) {
    return Result(Failure(e))
  }
}

// 아래처럼 해도 같은 결과
``` kotlin
fun <T> circuit ()
// ..
): Result<T> {
  return runCatching {
    circuitBreaker.run (name, block)
  }
}
```

- **fallback 함수 구현**
  - Result의 확장함수로 구현
  - = Result클래스의 함수를 하나 더 구현 -> Result 값 뒤에 **체이닝**형태로 사용
``` kotlin
fun <T> Result<T>.fallback (block: () -> T): Result<T> {
  when (this.isSuccess ()) {
    true -> return this // 성공인 경우 그대로 응답
    false -> return runCatching { block() } // 실패인 경우에 fallback용 블록 실행 & 똑같이 Result로 감싸서 반환
  }
}
```


3. fallbackIfOpen 구현
``` kotlin
fun <T> Result<T>. fallbackIf0pen (block: () -> T) : Result<I> {
  when (this.exceptionOrNall()) {
    // Result에서 전달받은 exception이 cirtcuitOpenException인 경우에만 전달받은 fallback 블록 실행
    is CircuitOpenException -> return runCatching { block() }
    else -> return this
}
```

### 완성 & 비교
<img width="599" alt="image" src="https://github.com/IMHYEWON/tech-youtube-notes/assets/37797830/26056316-693e-4af5-be2c-7394f7f93559">

#### Result 껍질 없애기
- 응답이 클라이언트로 나가기 위해 Result에서 실제 응답을 꺼내는 과정이 하나 추가되었음.
- KotlinResultAdvice로 해결 -> 응답이 나가는 순간에 응답을 캐치해서 응답 조작
  - → 잘 동작하지 않았음 → *왜?*
  - → 실제로는 Result가 사라지고 Group만 남았음
  - → fail인 경우에도 실제로는 Result 벗겨지고 Failure만 남음
  - Result는 value 클래스임 → 코틀린에서 Value 클래스는 컴파일과정에서 사라짐
  - Value 클래스가 있는  이유? 가독성을 높이고, 실수를 줄이기 위함 그러나 컴파일 과정에서 사라져 실제로 생성되는 비용을 줄임
  - Result를 두번 감싸서 해결 -> Result(body).getOrThrow()

