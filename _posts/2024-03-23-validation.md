---
layout: post
date: 2024-03-23
categories: [validation]
title:  "Jakarta Validation(Bean Validation) , Spring Validation , Spring MVC 에서 유효성 검증"
img_path: /img/path
---

## 들어가며
컨트롤러에서 요청에 대한 validation 을 할 때는 다음과 같이 요청 객체 필드에 `jakarta.validation.constraints` 어노테이션을 추가해주고 메서드 파라미터 요청 객체에 @Valid 어노테이션을 사용해주는 방법을 사용한다.
```java
@Getter
@AllArgsConstructor
@ToString
public class PersonRequest {

    @NotNull
    @Length(min = 2, max = 30)
    private final String name;

    @NotNull
    @Min(18)
    private final Integer age;
}
```

```java
@RestController
public class DemoController {

    @PostMapping("/my")
    public String home(@Valid @RequestBody PersonRequest request) {

        return request.toString();
    }
}
```

이 경우 request 의 age 가 17세 인경우 예외가 발생한다. 즉 , 요청에 대한 검증이 동작했다고 볼 수 있다.

## 검증이 제대로 동작하지 않을 때
```java
@Slf4j
@RestController
public class DemoController {

    @PostMapping("/company")
    public String company(@Valid @RequestParam @Min(18) Integer age) {

        return age.toString();
    }
}
```

마찬가지로 @Valid 어노테이션을 적용하고 @RequestParam 에 `jakarta.validation.constraints` 을 추가해주면  `company?age=17` 로 요청이 들어왔을 때 검증하여 예외가 발생할 것으로 기대된다.

하지만 실제로 위와 같은 코드에서는 벨리데이션이 동작하지 않아 예외가 발생하지 않고 정상적으로 컨트롤러가 호출된다.

왜 그럴까? [스프링 Web MVC 문서](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-validation.html)를 살펴보면  @ModelAttribute, @RequestBody, @RequestPart 가 있는 메서드 파라미터에 대해 @Valid 또는 @Validated 가 있는 경우 유효성을 검사한다고 언급하고 있다.  즉 , @RequestParam 인 경우에는 유효성 검사를 진행하지 않아서 위 요청에 대한 예외가 발생하지 않았던 것이다.

왜 @ModelAttribute, @RequestBody, @RequestPart 가 있는 메서드 파라미터에 대한 유효성 검증만을 수행하며 @Valid 는 왜 붙여줘야 했는지 그리고 @Validated 는 또 무엇인지 알아보기 위해서는 Jakarta Validation(Bean Validation) , Spring validation , Spring MVC 에서 유효성 검증에 대해서 알아볼 필요가 있다.

## Jakarta Validation 이란 ?

먼저 Jakarta Validation(Bean Validation) 는 Java에서 데이터의 유효성 검증을 위한 표준으로  이 표준은 객체의 데이터가 사전에 정의된 제약 조건을 충족하는지 검증하기 위한 어노테이션 기반의 API를 제공한다.  주요 인터페이스 및 클래스를 알아보자.

- `@Constraint` 는 검증을 위한 어노테이션을 정의할 때 사용되며 검증 로직을 수행하는 **ConstraintValidator** 구현체를 지정하는 데 사용된다. @Size, @NotNull 과 같은**jakarta.validation.constraints** 도 내부적으로 @Constraint 를 포함하고 있다.
- `ConstraintValidator` 는 주어진 객체에 대해 @Constraint 으로 지정한 제약 조건을 검증하는 역할을 가지고 있으며 @Constraint 마다 구현체를 가지고 있다. 예를 들어 @Size 어노테이션을 resolve 하는 **ConstraintValidator 는 SizeValidatorForArraysOfInt , SizeValidatorForCharSequence 등 이 있다.**
- `@Valid` 는 검증 대상임을 지정하는 어노테이션으로 객체의 재귀적인 검증을 위해 필요하다고 한다. 또한 이 어노테이션이 있는지 없는지에 따라 Spring MVC 에서 검증 대상인지를 판별하기 때문에 검증 대상의 메타데이터라고 생각하면 편하다.
- `Validator` 는 실제 검증을 수행하는 인터페이스로 object 에 대한 모든 제약 조건을 검증한다. ValidatorFactory 를 통해 생성되며 내부적으로 **ConstraintValidator 를 사용하여** object 의 검증을 수행한다.

정리하자면 Jakarta Validation 는 검증을 위한 API 를 제공하는 라이브러리로  Validator 가 내부적으로 여러 **ConstraintValidator 를 가지고 있어 객체에서 @Constraint이 있는 객체에 대한 검증을 수행하는 방식으로 동작한다고 볼 수 있다.**

```java
@Getter
@AllArgsConstructor
@ToString
public class Person {

    @NotNull
    @Length(min = 2, max = 30)
    private final String name;

    @NotNull
    @Min(18)
    private final Integer age;
}
```

```java
public class DemoService {

    private final Validator validator;

    public DemoService() {
        ValidatorFactory validatorFactory = Validation.buildDefaultValidatorFactory();
        this.validator = validatorFactory.getValidator();
    }

    public void validateUser(Person person) {
        Set<ConstraintViolation<Person>> violations = validator.validate(person);

        if (!violations.isEmpty()) {
            throw new IllegalArgumentException();
        }
    }
}
```

Validation 에서 디폴트 ValidatorFactory 를 가지고와 Validator 를 생성할 수 있고 Validator 로 Person 객체를 검증할 수 있다.

```java
class DemoServiceTest {

    DemoService service = new DemoService();
        
    @Test
    void validatePerson() {
        Person person = new Person("hi", 21);

				assertThatCode(() -> service.validateUser(person))
                .doesNotThrowAnyException();
    }

    @Test
    void validatePersonFail() {
        Person person = new Person("h", 21);

        assertThatThrownBy(() -> service.validateUser(person))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void validatePersonFail2() {
        Person person = new Person("hi", 2);

        assertThatThrownBy(() -> service.validateUser(person))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```
테스트까지 잘 동작하는 것을 볼 수 있고 Bean Validation 이 어떻게 검증을 수행하는지 간략하게 알아보았다.

## Spring Validation 이란?

bean Validation 을  포함하여 Spring 의 자체 유효성 검사 기능으로 통합된 검증 방법을 지원한다. Spring Validation 에도 Validator 가 있으며 **supports** 메소드에서는 검증할 객체의 타입을 확인하고, **validate** 메소드에서는 실제 검증 하는 방식으로 동작 및 지원한다.

**LocalValidatorFactoryBean 설정으로 Bean Validation 을 설정 및 사용할 수 있도록 지원하는데 이를 이용해**  @Validated 어노테이션을 클래스 단위로 지정하면 클래스의 메서드에 @Valid 가 있는 객체에 대해 검증을 수행한다. 이 패턴은 스프링에서 많이 사용하는 AOP 로 앞서 본 DemoService 의 기능을 다음과 같은 코드 수준에서 정리할 수 있다.

```java
@Service
@Validated
public class DemoSpringService {

    public void validateUser(@Valid Person person) {
    
        //DemoSpringService$$SpringCGLIB$$0
        System.out.println(person.getClass().getName());
        System.out.println(person);
    }
}
```

@Validated 는 스프링의 @Transactional, @Asyc 와 같이 프록시 패턴을 이용하여 Person 객체에 대한 검증을 validateUser 메서드 호출 전에 수행하는 스프링 빈을 등록할 수 있게 해주는 어노테이션이라고 볼 수 있다. 이외에도 Spring Validation 은 여러가지 지원하지만 깊게 알고 싶지는 않으니 넘어가자.

## Spring MVC 에서 유효성 검증

그렇다면 앞서 본 컨트롤러는 어떻게 동작하는 걸까?

```java
@RestController
public class DemoController {

    @PostMapping("/my")
    public String home(@Valid @RequestBody PersonRequest request) {

        return request.toString();
    }
}
```

이 컨트롤러에 @Validated 어노테이션이 있는 것도 아니면서 Validator 를 알고 있는 것도 아니다. 하지만 이 경우는 요청에 대한 검증이 이루어지는 것을 볼 수 있었다. 바로 이 부분이 Spring MVC 가 지원하는 영역이다.  앞서 언급한 @ModelAttribute, @RequestBody, @RequestPart 의 HandlerMethodArgumentResolver 에는 @Valid ,@Validated 어노테이션이 있는지 확인하고 벨리데이션을 진행하지만 @RequestParam 의 경우에는 그러한 로직을 찾아 볼 수 없다. 

즉, 컨트롤러의 파라미터의 HandlerMethodArgumentResolver 가 @Valid ,@Validated 어노테이션이 있다면 검증을 진행하는 방식으로 동작해온 것이다. 동시에  RequestParamMethodArgumentResolver 에는 검증하지 않기에 다음과 같이`@Valid @RequestParam @Min(18) Integer age` 을 해줘도 검증이 이루어지지 않았던 것이다.

```java
@Slf4j
@RestController
public class DemoController {

    @PostMapping("/company")
    public String company(@Valid @RequestParam @Min(18) Integer age) {

        return age.toString();
    }
}
```
그렇다면 위 API 에 age 가 18세 이상이길 검증하기 위해서는 어떻게 해야할까 ? 

- DemoController 에 @Validated 를 설정하거나
- 함수 내부에서 검증 로직을 작성하거나
- RequestParam 에 대한 커스텀한 HandlerMethodArgumentResolver 를 구현하여 등록해주면 될 것 같다.


## 결론

`@Valid` 는 bean Validation 의 검증 대상임을 나타내는 메타데이터 어노테이션이며 `@Validated` 는  프록시 패턴을 사용하여 Bean Validation 의 Validator 로 @Valid 가 적용된 객체에 대해 검증을 수행해주는 AOP 메타데이터 어노테이션이다. 그리고 @ModelAttribute, @RequestBody, @RequestPart  를 사용한 파라미터의 검증은 HandlerMethodArgumentResolver 에서 @Valid, @Validated 어노테이션이 있는지 확인하고 검증을 한다.

Jakarta Validation(Bean Validation) , Spring Validation , Spring MVC 에서 유효성 검증을 분리해서 알아보았고 컨트롤러에 어떤 식으로 검증이 이루어져왔는지 알게된 시간이었다.

## Reference
- <a href="https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-validation.html" target="_blank">https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-validation.html</a>
- <a href="https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html" target="_blank">https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html</a>
- <a href="https://jakarta.ee/specifications/bean-validation/3.0/jakarta-bean-validation-spec-3.0.html#constraintsdefinitionimplementation-constraintfactory" target="_blank">https://jakarta.ee/specifications/bean-validation/3.0/jakarta-bean-validation-spec-3.0.html#constraintsdefinitionimplementation-constraintfactory</a>
