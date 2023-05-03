---
layout: post
date: 2023-05-03 
categories: [spring]
title:  "Spring Retry 사용기"
img_path: /img/path
---

# 들어가며
---

최근에 클라우드 플랫폼에 서버를 올리면 종종 여러 가지 이유로 서버가 다운되어 응답 없는 경우가 발생했습니다.

직접 접속해 보기 전까지 이를 발견하지 못했으므로 직접 헬스 체크 기능을 구현하기로 마음을 먹었습니다.

사전에 약속된 앤드 포인트로 요청을 보내고 정상 응답을 기대하는 헬스 체크 기능을 구현한다고 했을 때

서버가 완전히 다운되지 않고 여러 이유들로 인해 순간적 실패의 경우.

곧바로 실패했다고 레코드를 저장하고 서버 주인에게 서버가 정상 동작하지 않는다는 이메일을 보내는 것은

재시도 호출을 수행하는 것보다 큰 비용이 발생할 수 있다고 생각했습니다.

대표적으로 순간적인 네트워크 오류일 경우에 서버가 다운되지 않았는데에도 잘못된 메일이 보내질 수도 있습니다.

하지만 재시도 기능을 구현한다고 하더라도 
충분한 지연시간을 두지 않고 여러 번의 재시도 요청을 보낸다면 네트워크에 더욱더 부담을 줄 가능성이 있습니다.


# 그래서 어떻게
---

이 문제를 해결하기 위해 헬스 체크 기능을 
루프문을 활용하여 3번의 재시도와  Thread API 를 활용하여 1초의 지연시간을 두어 구현해보겠습니다.


**헬스 체크 기능**

```java
public class HealChecker {
    private final RestTemplate restTemplate;
    ...

    public void check(String endPointUrl){
        restTemplate.getForObject(endPointUrl,String.class);
    }
    ...

}
```
  -  앤드포인트에 GET 방식으로 요청을 보내고 String 타입의 응답을 받습니다.



**헬스 체크 세번 시도 및 지연시간 구현**

```java
public class HealthChecker {

    private static final int THREE_TIMES = 3;
    private final RestTemplate restTemplate;
    ...

    public void checkUpToThreeTimes(String endPointUrl){
        int count = THREE_TIMES;
        while(count != 0){
            try {
                check(endPointUrl);
                return;
            }catch (RestClientException e){
                count -= 1;
                backOff(1000);
            }
        }
        throw new IllegalStateException("헬스 체크 시패!");
    }

    public void check(String endPointUrl){
        restTemplate.getForObject(endPointUrl,String.class);
    }

    private void backOff(int time){
        try {
            Thread.sleep(time);
        }catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
    ...

}
```
- 재시도 기능은 while 문을 사용해서 세번 호출하여 헬스체크를 수행할 수 있습니다.
- Thread.sleep() 을 활용하여 지연시간을 줄 수 있습니다.

하지만 이러한 재시도 기능이 다른 곳에서도 자주 사용된다면 매번 이런 식으로 코드를 작성해야 합니다.

결정적으로 3번 시도하는 기능은 **부가 기능**으로써 HealthChecker 가 직접 그 책임을 가지는 것보다

재시도라는 부가기능으로 분리하여 구현하는 것이 재사용 및 유지 보수 면에서 좋아 보입니다.


# AOP 를 활용해보자
---
어노테이션 기반의  Spring AOP 를 사용하여 재시도 부가 기능을 정의하고 헬스 체크 기능에 적용해보겠습니다.

**aop 의존성 추가**

```gradle
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

**@Retry 어노테이션 정의**
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Retry {
}
```

**어드바이저 구현**
```java
@Aspect
@Component
public class RetryAspect {

    private static final int THREE_TIMES = 3;

    @Around("@annotation(com.example.healtychecktest.aop.Retry)")
    public Object execute(ProceedingJoinPoint joinPoint) throws Throwable {
        int count = THREE_TIMES;
        while(count != 0){
            try {
                joinPoint.proceed();
                return null;
            }catch (RestClientException e){
                count -= 1;
                backOff(1000);
            }
        }
        throw new IllegalStateException("헬스 체크 실패!");
    }

    private void backOff(int time){
        try {
            Thread.sleep(time);
        }catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

}
```
- 앞서 구현했던 재시도 기능과 지연시간 기능을 가진 어드바이저를 생성해줍니다.


**헬스 체크 기능에 적용**
```java

@Service
public class HealthChecker {
    private final RestTemplate restTemplate;
    ...

    @Retry// 정의한 재시도 기능 
    public void check(String endPointUrl){
        restTemplate.getForObject(endPointUrl,String.class);
    }

    ...
}

```
- @Retry 어노테이션을 활용해서 손쉽게 재시도 기능을 헬스 체크에 적용할 수 있었습니다.



사실 스프링에는 AOP 를 활용하여 이미 재시도 기능을 제공하는 <a href="https://github.com/spring-projects/spring-retry" target="_blank">`spring-retry`</a> 기능이 있습니다.

직접 구현한 기능 이외에 재시도 실패시 복구 기능 등 여러가지 기능을 제공합니다.


# Spring  Retry로 문제 해결
---
Spring Retry 를 사용해서 3번까지 재시도하는 기능을 구현하도록 하겠습니다.


**의존성 추가**
```gradle
implementation 'org.springframework.retry:spring-retry'
```

**설정**
```java
@Configuration
@EnableRetry
public class RetryConfig {
}
```

**재시도 기능 도입**
```java
@Service
public class HealthChecker {

    private final RestTemplate restTemplate;

    ...


    @Retryable(
            retryFor = RestClientException.class, // 재시도할 예외
            maxAttempts = 3, // 재시도 횟수
            backoff = @Backoff(delay = 1000) // 지연 시간 1초
    )
    public void check(String endPointUrl){
        restTemplate.getForObject(endPointUrl,String.class);
    }

    ...
}
```
- 재시도 기능을 적용할 예외를 지정하고 재시도 횟수, 지연시간 등을 지정할 수 있습니다.


# 그런데 테스트는
---
성공하는 경우 , 실패하는 경우를 테스트한다면 어렵지 않을 것 같습니다. 

하지만 실패하는 경우 3번이나 시도를 했는지 혹은 2번의 시도에 마침내 성공하는 경우를 테스트하는 것은 어려워보입니다.

헬스 체크 기능은 외부서버에 요청을 보내 응답을 받는 기능이므로 외부서버를 모킹해서 테스트를 수행하도록 하겠습니다.

외부 서버를 모킹하는 테스트는 <a href="https://github.com/square/okhttp/tree/master/mockwebserver" target="_blank">MockWebServer</a>를  참고해주세요.


**의존성 추가**
```gradle
testImplementation("com.squareup.okhttp3:mockwebserver:4.10.0")
```

**테스트 설정**
```java
@SpringBootTest
class HealthCheckerTest {

    @Autowired
    private HealthChecker healthChecker;

    private MockWebServer mockWebServer;

    @BeforeEach
    void setup() throws IOException {
        mockWebServer = new MockWebServer();
        mockWebServer.start();
    }

    @AfterEach
    void cleanup() throws IOException {
        mockWebServer.shutdown();
    }	
}

```
- SpringBootTest 로 빈에 등록된 HealthChecker 를 주입 받습니다. \
  이때 주입 받은 HealthChecker 클래스가 직접 생성한 클래스가 아니라 \
  … HealthChecker\$\$SpringCGLIB$$0 이름을 가지고 있는 프록시 객체여야합니다.
  
**3번 모두 실패 테스트**
```java
@Test
@DisplayName("응답 없는 서버에 3번 시도하는 경우")
public void noResponseServerTest() throws Exception{
    String baseUrl = mockWebServer.url("/").toString();

    // 실패 응답 정의
    MockResponse failResponse = new MockResponse();
    failResponse.setResponseCode(500);

    // 3번 실패
    mockWebServer.enqueue(failResponse);
    mockWebServer.enqueue(failResponse);
    mockWebServer.enqueue(failResponse);

    // 결국에 실패하고 RestClientException 던진다.
    assertThatCode(() -> healthChecker.check(baseUrl))
                .isInstanceOf(RestClientException.class);

}
```
- 3번의 시도 모두 실패하는 테스트입니다. 
- check 기능의 RestTemplate 요청을 mockwebserver 로 보내고  500코드 를 응답으로 하는 실패 응답을 지정합니다
- 결국엔 실패하는 것으로 기대할 수 있고 이 테스트는 통과하게 됩니다.


**2번 실패 후 마지막에 성공 테스트**

```java
@Test
@DisplayName("3번째에 응답이 성공하는 경우")
public void sometimesResponseServerTest() throws Exception{
    String baseUrl = mockWebServer.url("/").toString();

    // 실패 응답 정의
    MockResponse failResponse = new MockResponse();
    failResponse.setResponseCode(500);

    // 성공 응답 정의
    MockResponse successResponse = new MockResponse();
    successResponse.setResponseCode(200);
    successResponse.setBody("OK");

    // 2번 실패 1번 성공
    mockWebServer.enqueue(failResponse);
    mockWebServer.enqueue(failResponse);
    mockWebServer.enqueue(successResponse);

    // 결국에 성공
    assertThatCode(() -> healthChecker.check(baseUrl))
            .doesNotThrowAnyException();

}
```
- 2번시도는 실패하고 마지막 시도일때 성공하는 테스트입니다.
- check 기능의 RestTemplate 요청을 mockwebserver 로 보내고  500코드 를 응답을 2번 정상 응답을 마지막으로 지정합니다.
- 결국엔 성공하는 것으로 기대할 수 있고 이 테스트는 통과하게 됩니다.


# 마무리
---
이렇게 해서 **헬스 체크 기능 ,실패하면 3번까지 시도하기** 기능 구현과 테스트까지 완료했습니다.

Spring Retry 기능은 AOP 를 사용하는 만큼 내부호출문제를 인지하고 사용해야합니다.


# Reference
---
- <a href="https://github.com/spring-projects/spring-retry" target="_blank">https://github.com/spring-projects/spring-retry</a>