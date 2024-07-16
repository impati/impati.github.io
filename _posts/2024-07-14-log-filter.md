---
layout: post
date: 2024-07-14
categories: [java,spring,logback]
title:  "로그 필터링하기"
img_path: /img/path
---

## 들어가며

로깅은 개발할 때 의도한 대로 동작하는지 확인할 수 있는 좋은 방법이다. 나아가 운영할 때 요청이 어떻게 왔는지 그 요청에서 데이터들이 어떤 상호작용을 했는지 확인할 수 있다. 그렇다고 로깅을 너무 많이 남기면 로그파일에서 원하는 데이터를 검색하는데 시간이 아주 오래 걸릴 수 있다.

## 의도하지 않은 로깅

이러한 사실을 잘 알고 있다고 하더라도 개발을 하다보면 모듈 재사용은 자주 일어난다. 재사용하는 모듈에서 하드?하게 로깅을하고 있다면 새로운 API 개발에서 로깅이 필요하지 않았음에도 많은 양의 로그를 남기게 될 가능성이 높다. 

이 문제를 해결하기 위해서 짧게 생각해본다면 모듈을 재사용하지 않거나 재사용 모듈에서 로깅을 제거하고 기존 로깅하는 위치를 변경하는 것이다. 로그를 너무 많이 남아서 로그 검색이 느리다는 이유로 기존 설계 및 코드를 변경하는 것은 부가 기능이 비지니스 로직에 영향을 주는 것을 의미하기에 원하는 사람은 없을 것이다.

처음부터 이러한 문제를 고려해서 개발했다면 좋았을까? 개발하면서 매번 로깅을 고려하면서 개발하기에는 쉽지 않다.

## 로깅 필터링하기

특정 조건에 만족할 때 로깅을 비활성화할 수 있다면 로깅이 필요하지 않은 API 에 대해서만 로깅을 하지 않는 방법으로 기존 설계 및 코드를 변경하지 않고 대응할 수 있다. 

Logback  의 Logger 에서는 로그 메시지를 필터링한 뒤 기록하는 모습을 볼 수 있었는데 FilterReply 값이 DENY 이면 로깅을 하지 않고 ,  NEUTRAL 이면 레벨을 판단 후 기록 , ACCEPT 면 로그를 무조건 기록한다.

```java
    private void filterAndLog_0_Or3Plus(final String localFQCN, final Marker marker, final Level level, final String msg, final Object[] params,
                    final Throwable t) {

        final FilterReply decision = loggerContext.getTurboFilterChainDecision_0_3OrMore(marker, this, level, msg, params, t);

        if (decision == FilterReply.NEUTRAL) {
            if (effectiveLevelInt > level.levelInt) {
                return;
            }
        } else if (decision == FilterReply.DENY) {
            return;
        }

        buildLoggingEventAndAppend(localFQCN, marker, level, msg, params, t);
    }
```

로깅 요청을 필터링하는 클래스는 TurboFilter(추상 클래스) 이며 로깅 컨텍스트와 연결이 되어 있다고한다. 스프링에서는  LogbackLoggingSystem 가 logback-spring.xml 파일을 읽어 LoggerContext 를 구성하기 때문에 logback-spring.xml 에 특정 조건에 로깅하지 않는 TurboFilter 를 등록한다면 특정 API 에서 로깅을 비활성화 할 수 있다.

TurboFilter 는 추상클래스이므로 하위 구현 클래스인 MDCFilter 를 이용하면 스레드 단위로 로깅을 필터링 할 수 있을것으로 보인다.
MDCFilter 의 MDCKey 의 value 가 일치하면 onMatch 를 반환하는데 onMatch 가 FilterReply 타입이다.

```java
public class MDCFilter extends MatchingFilter {

    String MDCKey;
    String value;

    ...

    @Override
    public FilterReply decide(Marker marker, Logger logger, Level level, String format, Object[] params, Throwable t) {

        if (!isStarted()) {
            return FilterReply.NEUTRAL;
        }

        String value = MDC.get(MDCKey);
        if (this.value.equals(value)) {
            return onMatch;
        }
        return onMismatch;
    }
}
```


logback-spring.xml  에 다음과 같이 DISABLE_LOGGING 값으로 true 로 매치되는 경우 DENY 를 반환하도록 하면 true 일 때 필터링시킬 수 있다.

```xml
    <turboFilter class="ch.qos.logback.classic.turbo.MDCFilter">
      <MDCKey>DISABLE_LOGGING</MDCKey>
      <value>true</value>
      <OnMatch>DENY</OnMatch>
      <OnMismatch>NEUTRAL</OnMismatch>
    </turboFilter>
```

MDC 의 키 값으로 DISABLE_LOGGING value 로 “true” 를 넣어주면 아래 demoService 의 호출에서의 모든 로깅은 필터링된다. MDC 가 스레드 로컬을 활용하기 때문에  다른 요청에 대해서 로깅이 필터링 되는 경우를 우려할 필요도 없다.

```java

@RestController
public class DemoController {

    private final DemoService demoService;

    @PostMapping("/demo")
    public String demo() {
    
        MDC.put("DISABLE_LOGGING", "true");
        
        demoService.doSomething();
        
        MDC.remove("DISABLE_LOGGING");

        return "ok";
    }
}
```

다만, MDC는 스레드 로컬을 이용하므로 마지막에는 반드시 remove 를 호출하여 초기화해줘야한다.


## 개선해보기

로깅을 비활성화하는 구조를 보면 스프링의 AOP 기능을 이용하기에 정말 좋다. 다음과 같이 어노테이션과 Aspect 를 구현해서 빈으로 등록해주면 로깅을 비활성화하는 부가 기능을 분리할 수 있다.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DisabledLog {

}
```

```java
@Aspect
@Component
public class DisabledLogAspect {

    @Before("@annotation(com.baemin.coupon.log.DisabledLog)")
    public void before() {
        MDC.put("DISABLE_LOGGING", "true");
    }

    @After("@annotation(com.baemin.coupon.log.DisabledLog)")
    public void after() {
        MDC.remove("DISABLE_LOGGING");
    }
}
```

```java
@RestController
public class DemoController {

    private final DemoService demoService;

    @PostMapping("/demo")
    @DisabledLog
    public String demo() {
      
        demoService.doSomething();
        return "ok";
    }
}
```

## 마무리하며

로깅이 불필요한 API가 주문 트래픽을 모두 받고 있는데 하필 이 API 에서 로그를 너무 많이 찍고 있어서 피크 시간대에 원하는 로그 데이터를 검색하는데 불편함이 있었다.불필요한 로깅을 하고 있지는 않은지 되돌아 볼 수 있는 시간이었고 ,  로깅 필터링 기능이 있을 것 같았는데 못 찾다가 마침내 찾아서 기분이 아주 좋았다.