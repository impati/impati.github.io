---
layout: post
date: 2024-02-29
categories: [feign]
title:  "Feign Client 도입"
img_path: /img/path
---

## 들어가며
여차저차해서 Feign Client 를 도입하기로 결정했다면 어떻게 사용하고 어떤 설정들이 있는지 알아보자.

## Feign Client 란
Feign 은 선언적 웹 서비스 클라이언트 라이브러리로 HTTP 서비스 호출을 인터페이스와 애노테이션을 통해 정의할 수 있으며, Feign이 이를 구현하여 실제 서비스 호출을 처리하는 방식으로 동작한다. RestTemplate 이나 WebClient 를 사용해서 API 호출하는 방법보다 간단하게 API 호출을  할 수 있다는 점이 장점이다.  동기 방식으로 동작하며 비동기 방식으로 동작을 원하는 경우 설정을 따로 해줘야한다고 한다. (아마`AsynchronousMethodHandler` 를 사용하도록 해야하는 것 같다) 아니면 WebClient 를 사용하는 것도 방법이겠다. 하지만 그렇게 성능이 필요하지 않은 경우 트랜잭션 안에서 외부 API 호출 하지 않는 등의 주의만 해주면 Feign Client 사용은 적은 힘으로 큰 효율을 낼 수 있는 좋은 선택이라고 생각한다.

## Feign Quick Start
간단히 시작 및 사용하는 법을 알아보자.

- gradle 파일에 다음과 같이 종속성을 추가해준다.

```
dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.0")
    }
}

dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
}

```
이때 스프링 부트 버전에 맞는 ${springCloudVersion} 를 지정해줘야하는데 이 버전은  [이 곳](https://spring.io/projects/spring-cloud)에서 확인할 수 있다.
- `@EnableFeignClients` 를 설정해준다. 이 어노테이션을 사용하면 `@FeignClient` 가 선언되어 있는 인터페이스를 검색한다고 한다. 

```kotlin
@Configuration
@EnableFeignClients
class FeignConfig {
}
```

- `@FeignClient` 을 인터페이스에 선언해주면 인터페이스에 선언한 내용에 맞게 REST 클라이언트를 사용하여 구현체를 동적으로 생성해서 호출해준다.

```kotlin
@FeignClient(
    name = "feign-test",
    url = "https://github.com"
)
interface TestClient {

    @GetMapping("/impati")
    fun getMyGitHubPage(): String
}
```

이제 github 계정을 GET 요청하는 API 를 호출해서 실제로 제대로 동작하는지 확인해보자.

```kotlin
@SpringBootTest(
    classes = [FeignConfig::class]
)
class TestClientTest @Autowired constructor(
    val testClient: TestClient
) {

    @Test
    fun test() {
        val response = testClient.getMyGitHubPage()

        println(response)
    }
}

```
테스트를 실행하면 HTML 파일이 응답으로 오는데 아주 제대로 동작하고 있음을 의미한다. 3단계만으로 외부 API 호출을 완료했다. 이것이 feign Client 의 장점이다.

## 외부 API와의 통신 시 고려해야할 여러가지 요소

외부 API 와의 통신 시 고려해야할 요소들이 많은데 일일이 전부 기억이 나지 않아서 몇가지 예시를 던져주고 ChatGPT 4 에게 물어보았다.
![](/feign/first.png)

그리고 ChatGPT 4 의 답변은 다음과 같았다.

> **인증과 권한 부여** : 대부분 외부 API 는 인증을 요구합니다. API 키 , OAuth 토큰 등 다양한 인증 방식이 있으며 이러한 매커니즘을 이해하고 구현해야합니다.
>
> **연결 및 읽기 타임아웃 설정** : 네트워크 지연이나 외부 서비스의 다운타임은 애플리케이션의 성능에 영향을 줄 수 있습니다. 연결(connection timeout) 및 읽기(read timeout) 타임아웃을 설정하여, 이러한 문제가 발생했을 때 적절한 시간 내에 실패로 처리하고 대응할 수 있어야 합니다.
>
> **에러 처리** : API 호출 결과로 다양한 에러가 발생할 수 있습니다. 이러한 에러를 적절히 처리하고, 필요한 경우 사용자에게 의미 있는 메시지를 제공해야 합니다.
>
> **재시도 전략** : 일시적인 네트워크 문제나 외부 API의 장애로 인해 요청이 실패할 수 있습니다. 이런 경우를 대비해, 재시도 전략을 구현해야 합니다. 지수 백오프(exponential backoff)와 같은 전략을 사용하여, 실패한 요청을 지정된 시간 간격으로 재시도할 수 있습니다.
>
> **로깅 및 모니터링** : 요청과 응답을 로깅하여 문제가 발생했을 때 원인을 분석할 수 있도록 해야 합니다. 또한, API 사용량, 응답 시간, 실패율과 같은 지표를 모니터링하여, 문제를 조기에 발견하고 대응할 수 있도록 해야 합니다.


ChatGPT 4 가 알려준 이 내용을 Feign Client 를 사용하여 모두 고려해보자.

### 인증과 권한 부여

만약 호출하고자 하는 외부 API 에 api 토큰이 필요한 경우 Feign Client 에서는 어떻게 해야할까?
Feign Client 의 `RequestInterceptor` 를 사용하면 요청 헤더를 추가할 수 있다.

- 요청 헤더에 추가할 정보를 정의한 RequestInterceptor 를 정의한다.

```kotlin
class BearerAuthRequestInterceptor(
    private val bearerToken: String
) : RequestInterceptor {

    override fun apply(template: RequestTemplate?) {
        template!!.header("Authorization", bearerToken)
    }
}

```

- @FeignClient 의 Configuration 에 사용할 Configuration 을 정의해준다.

```kotlin
class TestHeaderConfiguration(
    @Value("\${test.api-key}")
    val apiKey: String
) {

    @Bean
    fun bearerToken(): BearerAuthRequestInterceptor {
        return BearerAuthRequestInterceptor(apiKey)
    }
}
```

- @FeignClient 의 Configuration 에 사용할 정의한 Configuration 을 지정해준다.

```kotlin
@FeignClient(
    name = "feign-test",
    url = "https://github.com",
    configuration = [TestHeaderConfiguration::class]
)
interface TestClient {

    @GetMapping("/impati")
    fun getMyGitHubPage(): String
}
```

이러한 방법으로 요청 헤더에  api-key 값과 같은 원하는 값을 넣을 수 있다. 

### 연결 및 읽기 타임아웃 설정

연결 및 읽기 타임아웃 설정은  다음과 같이 할 수 있다.

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          ${feignName}
            connect-timeout: 2000
            read-timeout: 2000
```

이 설정 정보는 `FeignClientProperties` 클래스에 바인딩되며 설정하지 않는 경우 기본값으로 설정된다. 버전에 따라 feign.client.config… 로 설정해주어야할 수 있으니 버전에 맞는 설정을 하자.

이 설정 정보로 url, loggerlevel , errorDecoder , retryer , defaultQueryParamater 등등 설정할 수 있다.  자세한 정보는 `FeignClientProperties` 를 확인해보자.

### 에러 처리

외부 API 호출시 발생하는 에러를 내부적인 예외로 변환하거나 로깅하는 등 핸들링 하여 처리하는 역할은 ErrorDecoder 를 통해 할 수 있다. 응답 요청이 2XX 가 아닌 경우 decode 메서드가 호출되며 요청에 대한 응답이 5XX 인 경우에 재시도하고 싶은 경우 RetryableException Exception 을 throw 할 수 있고 4XX 인 경우 로깅을 하는 등 자유롭게 응답에 대한 액션을 취할 수 있다. 별도의 등록 없이 ErrorDecoder 를 사용하는 경우 Default 를 사용하게 되는데 이 경우 응답 상태에 대한 적절한 FeignException 을 응답하고 응답 헤더에 `Retry-After` 가 있는 경우 RetryableException 던져 재시도 매커니즘이 시작되는 구조로 동작한다.

응답이 5XX 인 경우 RetryableException Exception 을 던지며 4XX 인 경우 로그를 남기는 ErrorDecoder 를 구현해보자.

```kotlin
class MyErrorDecoder : ErrorDecoder {

    private val log: Logger = LoggerFactory.getLogger(MyErrorDecoder::class.java)

    override fun decode(methodKey: String?, response: Response?): Exception {
        val request = response!!.request()
        val exception = FeignClientException.errorStatus(methodKey, response)

        if (exception.status() in 400..499) {
            log.warn("Client Error")
        }

        if (exception.status() in 500..599) {
            return RetryableException(
                exception.status(),
                "서버 오류로 실패했습니다",
                request.httpMethod(),
                currentTimeMillis() + RETRY_AFTER_MS,
                request
            )
        }

        return exception;
    }

    companion object {
        private const val RETRY_AFTER_MS = 100L
    }
}
```

이 코드는 200 응답이 오지 않았을 때 실행되며 4XX 인 경우에는 로그만을 남기고 5XX 인 경우 RetryableException 을 리턴하여 재시도 매커니즘을 실행하도록 한다. 이때 RETRY_AFTER_MS 는 재시도하기 전 대기하는 밀리초를 의미한다.

### 재시도 전략

Feign Client 라이브러리에  `Retryer`  인터페이스가 존재하는데  Retryer.NEVER_RETRY 를 사용할 경우 재시도를 하지 않겠다는 의미이고 , Retryer.Default 를 사용할 경우 재시도 전략을 사용하겠다는 의미이다.

재시도 전략을 사용하지 않는 feign 인 경우 configuration 에 다음과 같이 Retryer.NEVER_RETRY 를 빈으로 등록하는 configuration 을 지정해주면 된다.

```kotlin
class NoRetryConfiguration {

    @Bean
    fun noRetry(): Retryer {
        return Retryer.NEVER_RETRY
    }
}
```

재시도 전략을 사용하려는 feign 인 경우 Retryer.Default 빈으로 등록하는 것으로 끝나는 것이 아니라 `ErrorDecoder` 에서`RetryableException` 를 응답해줘야  `retryer.continueOrPropagate(e)` 이 호출되어 재시도 액션을 시작하게 된다. 이러한 이유에서 ErrorDecoder 와 Retryer 는 같이 등록해주는 것이 좋아보인다.

```kotlin
class RetryConfiguration {

    @Bean
    fun defaultRetry(): Retryer {
        return Retryer.Default(
            100, // period
            1000, // maxPeriod
            3 // maxAttempts
        )
    }

    @Bean
    fun myErrorDecoder(): ErrorDecoder {
        return MyErrorDecoder()
    }
}
```

Retryer.Default 는 지수 백오프(exponential backoff) 전략을 사용하고 지수 백오프 대기 시간이maxPeriod 시간보다는 길지 않게 대기하도록 설정할 수 있다.


### **로깅 및 모니터링**

우선 feign 에 대한 로깅 레벨을 지정하는 것이 우선되어야 한다. 로깅 레벨은 다음과 같이 네가지가 있으며 개발 단계 및 필요에 따라 알맞게 설정하여 사용하면 된다.

> **NONE**: 로깅을 수행하지 않는다.
>
> **BASIC**: 요청 방법과 URL, 응답 상태 코드, 실행 시간만 로깅한다.
>
> **HEADERS**: BASIC 레벨에 추가로 요청과 응답의 헤더를 로깅한다.
>
> **FULL**: 요청과 응답의 헤더, 바디, 메타데이터를 포함하여 모든 정보를 로깅한다


feign 에 대한 로깅 레벨은 yml 파일로 설정해줘도 되고 bean 으로 등록해줘도 된다.

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          ${feignName}
            connect-timeout: 2000
            read-timeout: 2000
            logger-level: basic
```

또는 

```kotlin
@Configuration
class FeignClientLoggerConfiguration {

    @Bean
    fun feignLoggerLevel(): Logger.Level {
        return Logger.Level.FULL
    }
    
    
    @Bean
    fun feignLogger(): Logger {
        return object : Logger() {
            override fun log(configKey: String, format: String, vararg args: Any?) {
                // 커스텀 마이징
            }
        }
    }
}
```

FeignClientLoggerConfiguration 를 빈으로 등록해 전역 설정을 할 수도 있고 아니면 @Configuration 를 사용하지 않고 feign 의 configuration 에  FeignClientLoggerConfiguration 를 지정해주는 방법으로 원하는 feign 에만 로깅을 설정할 수 있다.(이러한 설정은 사실 재시작 설정이나 intercepor 과 같은 다른 설정들도 마찬가지이다.)

Logger 클래스를 보면 request , response 에대한 로깅 메서드를 오버라이드하여 커스텀하게 로깅할 수도 있다. 기본으로 제공되는 로그도 꽤나 괜찮아보여 로깅 레벨 설정만 해줘도 괜찮을 것 같다.

## 마무리 하며
외부 API와의 통신 시 고려해야할 여러가지 요소 설정만 초기에 잘 하고 나면 이후에 사용하기에 정말 편해보인다. 특히 base 가 되는 코드가 있다면 러닝 커브가 거의 존재하지 않는다고 봐도 무방할 정도로 사용하기가 편하다고 생각한다.