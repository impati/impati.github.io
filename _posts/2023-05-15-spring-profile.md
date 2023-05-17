---
layout: post
date: 2023-05-16
categories: [spring]
title:  "@Profile 로 설정에 따라 다른 빈 객체 주입받기"
img_path: /img/path
---


# 들어가며
---

진행하고 있는 프로젝트에서 동기 VS 비동기 호출, 로컬 캐싱 VS Redis, RestTemplate VS webClient를 비교하며 성능을 측정할 계획을 가지고 있다.

현재 프로젝트에서 특정 로직이 실패하게 되면 이메일을 보내는 기능이 있는데 성능 테스트를 수행하면서 수많은 요청과 함수 호출이 이루어질 것이므로 특정 로직이 실패할 확률은 높아진다.

**그러면 성능 테스트를 수행하며 불필요한 이메일을 무수히 많이 발송하는 문제가 발생한다.**


# 어떻게 해결하지 ?
---
가장 간단하게 생각해 보자 성능 테스트를 수행하면서 이메일을 보내는 로직을 주석 처리하면 되지 않은가?

아주 잠깐은 사용해 볼 만한 방법이지만 오랜 기간 성능 측정할 계획을 가진 나에게는 적합해 보이지는 않는다.

![example](/profile/profile1.png)

AlarmSender 인터페이스를 구현한 MailAlarmSender 가 빈에 등록되어 있고 

성능 테스트를 진행하다 SomeoneElse 가 AlarmSender 기능을 호출했을 때 MailAlarmSender 가 호출되는 구조에서

![example](/profile/profile2.png)

성능 테스트를 진행할 때 AlarmSender 인터페이스를 구현한 실제로는 아무 액션도 취하지 않는 StubAlarmSender 구현하여 MailAlarmSender 대신 빈에 등록하면 의도한 대로 동작을 기대할 수 있다.

그런데 운영 환경에서 MailAlarmSender를 빈에 등록하는 것을 까먹었다면?

혹은 MailAlarmSender을 빈에 등록했지만 StubAlarmSender 도 등록되어 있었다면?

부트가 실행하면서 오류가 발생하거나 메일을 보내야 할 때 보내지 않는 문제가 발생할 수 있다.


# @Profile 을 활용한 문제 해결
---
이러한 빈 등록 실수는 개발자의 실수로 인해 충분히 발생할 수 있는 문제이며 개발 단계에서 환경에 따라 빈을 등록해주는 기능을 사용하면 이러한 실수를 미연에 방지할 수 있다.

@Profile 어노테이션을 사용하면 spring.profiles.active 설정에 따라 빈을 주입해준다.

```java
@Bean
@Profile("prod")
public AlarmSender mailAlarmSender(){
    return new MailAlarmSender(javaMailSender,messageMaker());
}

@Bean
@Profile("local")
public AlarmSender stubAlarmSender(){
    return new StubAlarmSender();
}
```

- StubAlarmSender 는 Spring.profiles.active 값이 `local` 인 경우 빈으로 등록되고
- AlarmSender 는 Spring.profiles.active 값이 `prod` 인 경우 빈으로 등록된다.


# 왜 그럴까?
---
먼저 , 스프링 부트가 여러가지 방법(OS 환경 변수 , JVM 자바 시스템 속성 , JAVA 커맨드 라인 인수 , 설정 파일)으로 설정된  spring.profiles.active 정보를 읽는다.

그리고 빈으로 등록하기 전에  BeanFactoryPostProccessor 인 `ConfigurationClassPostProcessor` 클래스에서 @Conditional 어노테이션이 설정된 클래스의 빈 정보를 구성한다.

> @Conditional 은 지정된 조건이 일치하는 경우 구성요소가 될 수 있음을 나타내는 어노테이션이다.


`ConfigurationClassPostProcessor` -> \
`ConfigurationClassParser`  -> \
`ConditionEvaluator`

그리고 BeanPostProccessor 가 BeanFactoryPostProccessor 에서 정의된 빈들에 대해 인스턴스화 , 실제 빈으로 스프링 IOC컨테이너에 등록하는 것이다.

spring.profiles.active  값과 @Profile 설정으로 어떻게 다른 빈을 등록할 수 있는지 설명을 하다말고 이런 이야기를 한 이유는 @Profile 의 비밀에 있다.

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(ProfileCondition.class)
public @interface Profile {

	/**
	 * The set of profiles for which the annotated component should be registered.
	 */
	String[] value();

}
```
@Profile 이 @Conditional 어노테이션을 가지고 있다. 즉, ProfileCondition.class 의 반환 값에 의해서 빈으로 구성되거나 되지 않거나 결정되는 것이다.

ProfileCondition 파일을 읽어보면 @Profile(”…”) 에서 “…” 값을 읽고 설정된 spring.profiles.active 값과 비교하여 참/거짓을 반환한다.

```java
class ProfileCondition implements Condition {

	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
		if (attrs != null) {
			for (Object value : attrs.get("value")) {
				if (context.getEnvironment().acceptsProfiles(Profiles.of((String[]) value))) {
					return true;
				}
			}
			return false;
		}
		return true;
	}

}
```

정리하면 @Profile(”local”)의 local 값을 읽고 설정된 spring.profiles.active 값과 비교한 후

true / false를 반환하고 빈 팩토리 후처리기에서 구성될지 말지가 결정되고 최종적으로 일치한다면 빈으로 등록되고 그렇지 않다면 빈으로 등록되지 않는다.


# 테스트
---


```java
@SpringBootTest
public class ProfileTest {

    @Autowired
    private AlarmSender alarmSender;

    @Test
    @DisplayName("profiles.active = [local] 인 경우 클래스명이 StubAlarmSender")
    public void profiles_active_local() throws Exception{
        String className = alarmSender.getClass().getName();
        assertThat(className.contains("StubAlarmSender")).isTrue();
    }
    
}
```
- spring.profiles.active = local 설정 후 주입 받은 객체의 클래스 명이 StubAlarmSender 인지 테스트

```java
@SpringBootTest
public class ProfileTest {

    @Autowired
    private AlarmSender alarmSender;

    @Test
    @DisplayName("profiles.active = [prod] 인 경우 클래스명이 MailAlarmSender")
    public void profiles_active_prod() throws Exception{
        String className = alarmSender.getClass().getName();
        assertThat(className.contains("MailAlarmSender")).isTrue();
    }
    
}
```
- spring.profiles.active = prod 설정 후 주입 받은 객체의 클래스 명이 MailAlarmSender 인지 테스트

만약 spring.profiles.active = prod , local 으로 여러개 설정한다면 ? NoUniqueBeanDefinitionException 이 발생한다.



# 마무리하며
---

빈으로 등록하는 것을 주석 처리하며 성능 테스트를 진행해왔는데, 이게 맞나 싶어 어렴풋이 알고 있던 개념에 대해 다시 공부해 보았다.

결국 @Profile 은 @Conditional 을 사용하기 때문에 @Conditional를 사용하고 커스텀 한 Class를 구현해 준다면 보다 복잡한 상황에서 빈 주입을 컨트롤할 수 있을 것으로 기대된다.