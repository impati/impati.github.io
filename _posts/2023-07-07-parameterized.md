---
layout: post
date: 2023-07-07
categories: [test]
title:  "@ParameterizedTest 사용을 고려해보자"
img_path: /img/path
---

# 들어가며

---

테스트를 진행하다 보면 하나의 메서드에 여러 테스트를 수행해야 할 때가 있었다. 

예를 들어 `“1,2,3”`  과 같은 문자열이 입력으로 들어오고  `,` 혹은 `:` 으로 구분한 뒤에 값을 모두 더하는 메서드가 있다고 했을 때  가볍게 예상해 볼 수 있는 케이스는 다음과 같다.

 

- `“1,2”` ⇒ `3`
- `“1,2,3”` ⇒ `6`
- `“1,2:3”` ⇒ `6`
- `1:2:3:4` ⇒ `10`

이를 테스트하기 위해서는 다음과 같이 테스트 코드를 작성해 볼 수 있다.

```java
@Test
@DisplayName("1,2을 입력으로 3을 반환한다.")
void split_and_sum_case1(String input) {
	
    String input = "1,2";
    
    int result = splitAndsum(input);
    
    assertThat(result).isEqualTo(3);

}

@Test
@DisplayName("1,2,3을 입력으로 6을 반환한다.")
void split_and_sum_case2(String input) {
    
    String input = "1,2,3";

    int result = splitAndSum(input);

    assertThat(result).isEqualTo(6);

}

@Test
@DisplayName("1,2:3을 입력으로 6을 반환한다.")
void split_and_sum_case3(String input) {
		
    String input = "1,2:3";
    
    int result = splitAndSum(input);
    
    assertThat(result).isEqualTo(6);

}

@Test
@DisplayName("1:2:3:4을 입력으로 10을 반환한다.")
void split_and_sum_case4(String input) {
    
    String input = "1:2:3:4";
    
    int result = splitAndSum(input);
    
    assertThat(result).isEqualTo(10);

}
..
```

# 문제점

---

**splitAndSum**  테스트에서 문제점을 찾아보자. 여러 케이스 입력에 대한 결과가 다를 뿐 나머지 코드가 모두 중복된다는 문제점이 있다.  테스트 해야 하는 케이스가 더욱 많아지는 경우에는 중복 코드가 더 많아지며 테스트 코드에 대한 유지보수가 떨어질 수 있다.

이런 문제를 어떻게 해결할 수 있을까?


# @ParameterizedTest 로 문제 해결

---

Junit 은 @ParameterizedTest 으로 단일 테스트 메서드를 여러 입력 조합으로 실행할 수 있도록 도와준다.

@ParameterizedTest 을 사용하면 **splitAndSum**  테스트의 중복 코드 문제점과 케이스를 추가해야 하는 경우 유지보수하기 어렵다는 문제를 하나의 테스트 메서드만을 사용함으로써 해결할 수 있다.

@ParameterizedTest 는 테스트하고자 하는 메서드가 매개 변수가 있는 테스트임을 알리는 데 사용하고 적어도 하나의 @ArgumentSource 또는 ArgumentProvier 를 지정해야 한다.

> ArgumentSource 는 @ArgumentsSource 주석이 달린 테스트 메서드에 대한 인수 공급자를 등록하는 데 사용 되는 반복 가능한 주석입니다.
> 

```java

@DisplayName("쉼표 또는 콜론을 구분자로 분리한 각 숫자의 합을 반환.")
@ParameterizedTest
@MethodSource("splitAndSumArgumentFactory")
void split_and_sum(String input,int expected) {
    int result = splitAndSum(input);
    
    assertThat(result).isEqualTo(expected);
}

static Stream<Arguments> splitAndSumArgumentFactory() {
        return Stream.of(
                Arguments.of("1,2", 3),
                Arguments.of("1,2,3", 6),
                Arguments.of("1,2:3", 6),
                Arguments.of("1:2:3:4", 10)
        );
}
```

- @MethodSource는 팩토리 메서드에서 반환된 값에 대한 엑세스를 제공하는 ArgumentSource이고 인수 스트림을 생성하여 반환하는 팩토리 메서드를 사용하여 @ParameterizedTest 메서드의 개별 호출에 대해 인수로 사용할 수 있다.
- @MethodSource(”…”) 에는 ArgumentSource 로 사용할 외부 클래스 내 팩토리 메서드 이름을 명시할 수 있지만 보통 테스트를 수행할 때에는 현재 테스트 클래스 내에서만 사용되므로 테스트 클래스 내 팩토리 메서드를 정의하고 지정 해주는 것이 좋아보인다.

위 테스트에서는 `splitAndSumArgumentFactory` 팩토리 메서드를 정의하였고 `Arguments.of("...",..)` 인 순서에 맞게 `split_and_sum(String input,int expected)` 에 인수로 받아 테스트를 진행하였다.

@ParameterizedTest 를 사용함으로써 여러 케이스에 대한 중복 코드가 없어졌고 케이스를 추가하기 위해 팩토리 메서드에 `Arguments.of(...)` 만을 추가하면 된다는 장점을 가져갈 수 있다.

입력에 대한 결과 값이 달라지는 경우가 대부분이므로 @MethodSource를 통해서 문제를 해결하지만 @ParameterizedTest 를 사용하며 지정할 수 있는 다른 ArgumentSource 도 알아보자.

## @ValueSource

valueSource 는 **리터럴 값**에 대한 엑세스를 제공하는 ArgumentSource 이다 

지원되는 타입으로는 자바 primitive 타입과 String 이다 . ints, floats  … strings 등등

```java
@ParameterizedTest
@ValueSource(ints = {4, 5, 6, 7, 8, 9})
void move(int number) {
	...
	...
}
```

- ints = { … } 에 설정한 값이 `void move(int number)` 로 전달되어 테스트를 진행할 수 있다.

## @CsvSource

 value 속성이나 textBlock 속성을 통해 제공된 레코드에서 (기본적으로) 쉼표로 구분된 값을 읽은 후 @ParameterizedTest 메서드에 대한 인수로 제공한다.

@ValueSource 같은 경우 입력에 대한 결과 테스트에 적합하지 않지만 @CsvSource 의 경우 delimiter 로 구분할 수 있기 때문에 입력에 대한 결과 구성이 가능해진다.

```java
@ParameterizedTest
@CsvSource(value = {"1,2=3" , "1:2:3=6"} , delimiter = '=')
void split_and_sum(String input , int expecedNumber) {
	...
	...
}
```

- CsvSource 이 제공하는 기능이 꽤 많아보인다. [문서](https://junit.org/junit5/docs/5.9.1/api/org.junit.jupiter.params/org/junit/jupiter/params/provider/CsvSource.html)를 읽어보고 어떻게 응용할 수 있을지 생각해보자.

## @NullSource

@ParameterizedTest 메서드에 **단일** null 인수를 제공한다.

```java
@ParameterizedTest
@NullSource
public void null_source(Integer integer) {
    assertThat(integer).isNull();
}
```

## @**EmptySource**

@ParameterizedTest 메서드에 **단일** 빈 String 값 , 빈 List ,Set,Map 을 인수로 제공한다.

```java
@ParameterizedTest
@EmptySource
void empty_source(List<String> container) {
    assertThat(container.isEmpty()).isTrue();
}
```

## @**NullAndEmptySource**

@ParameterizedTest 메서드에 **단일** null 값과 빈 값을 인수로 제공한다. 

@NullSource 와 @EmptySource 의 기능을 결합하여 구성한 어노테이션이라고 한다.

```java
@ParameterizedTest
@NullAndEmptySource
void null_and_empty_source(String input) {
    assertThat(input).isBlank();
}
```

- String class 나 Collection 프레임워크에서 null 체크와 빈 값 체크를 자주 같이 테스트하기 때문에 제공하는 것으로 보인다.

# **@EnumSource**

Enum 상수를 인수로 제공한다. 

```java
@ParameterizedTest
@EnumSource(value = Season.class)
void enum_source(Season season) {
	  // season 인수로 모든 상수가 제공됨
		...
		...
}

enum Season {
	SPRING,
	SUMMER,
	FALL,
	WINTER
}
```

- `names` 옵션을 통해 Enum 상수의 이름에 따라 인수를 제공받을 수 있다.

```java
@ParameterizedTest
@EnumSource(value = Season.class, names = {"SPRING"}, mode = Mode.EXCLUDE)
void enum_source(Season season) {
		// season 인수로 SPRING 만 제공되지 않음.
		...
		...
}

enum Season {
	SPRING,
	SUMMER,
	FALL,
	WINTER
}
```

- `mode` 옵션을 통해 names 에 지정한 Enum 상수 이름만을 포함시킬지 제외 시킬지 지정할 수 있다.
    
    기본적인 값은 포함이며 names 에 지정한 상수 이름을 패턴을 통해서 매칭하는 `MATCH_ALL` , `MATCH_ANY` 옵션도 있다.
    

## @MethodSource

내부 또는 외부 테스트 클래스의 하나 이상의 팩토리 메서드를 참조할 수 있다.

각 팩토리 메서드는 `static` 으로 선언해야하며 (내부 클래스인 경우`@TestInstance(TestInstance.Lifecycle.*PER_CLASS*)` 어노테이션을 사용하면 static 으로 선언하지 않아도 된다)

각 팩토리 메서드는 인수 스트림을 생성하여 @ParameterizedTest 주석이 있는 메서드의 인수로 제공한다. 

```java
@ParameterizedTest
@MethodSource("stringProvider")
@DisplayName("빈 스트링 , null 값이 들어오면 0을 반환한다.")
void split_and_sum_case5(String input) throws Exception {
    
    String input = "1:2:3:4";
    
    int result = splitAndSum(input);
    
    assertThat(result).isEqualTo(0);

}

static Stream<String> stringProvider() {
    return Stream.of("", null);
}
```

- 단일 매개변수만 필요한 경우 위와 같이 작성할 수 있으며 팩토리 메서드 이름을 @MethodSource(”팩토리 메서드 이름”) value 로 적어주면 된다. 설정하지 않는다면 현재 메서드와 동일한 이름을 가진 팩토리 메서드를 찾는다.

```java
@ParameterizedTest
@MethodSource("range")
void testWithRangeMethodSource(int integer) {
    // integer 를 0 부터 20까지 인수로 받음
}

static IntStream range() {
    return IntStream.range(0, 20);
}
```

- 특정 범위의 int 타입의 여러 값이 필요한 경우 위와 같이 구성할 수 있다.

```java
@ParameterizedTest
@MethodSource("stringIntAndListProvider")
void testWithMultiArgMethodSource(String str, int num, List<String> list) {
    assertEquals(5, str.length());
    assertTrue(num >=1 && num <=2);
    assertEquals(2, list.size());
}

static Stream<Arguments> stringIntAndListProvider() {
    return Stream.of(
        arguments("apple", 1, Arrays.asList("a", "b")),
        arguments("lemon", 2, Arrays.asList("x", "y"))
    );
}
```

- 여러 매개변수가 필요한 경우 위와 같이 작성해줄 수 있으며 `Argument.of()` 를 사용하거나 `static import` 시 `Argument.arguments()` 를 사용할 수 있다.
    
    (`Argument` 의 `of` 메서드와 `arguments()` 메서드는 차이 없다 `arguments()` 가 static import 용)


# @ParameterzedTest에 DisplayName 부여하기

---

@ParameterizedTest 의 name 속성에 표현하고 싶은 DisplayName 을 설정할 수 있다. 
이때 다음의 `placeholders` 에 대해서 index 라던지 argument 이라던지 지정할 수 있다.

| Placeholder | Description |
| --- | --- |
| {displayName} | 테스트 메서드의 DisplayName |
| {index} | 호출 인덱스 |
| {arguments} | 쉼표로 구분된 Argument 목록 |
| {0} , {1} , {2} …  | 순서에 맞는 각 Argument  |

```java
@DisplayName("쉼표 또는 콜론을 구분자로 분리한 각 숫자의 합을 반환.")
@ParameterizedTest(name = {"[{index}] input = {0}, expected = {1}")
@MethodSource("splitAndSumArgumentFactory")
void split_and_sum(String input,int expected) throws Exception {
    
    int result = splitAndSum(input);
    
    assertThat(result).isEqualTo(expected);
}

static Stream<Arguments> splitAndSumArgumentFactory() {
        return Stream.of(
                Arguments.of("1,2", 3),
                Arguments.of("1,2,3", 6),
                Arguments.of("1,2:3", 6),
                Arguments.of("1:2:3:4", 10)
        );
}
```

- 필요한 `placeholders`  는 index 혹은  {0} , {1}  , … 선에서 마무리 지을 수 있을 것 같다.

![](/param/param.png)


# 마치며
---

무언가 반복적인 테스트 작업을 하고 있다면 @ParameterizedTest 사용을 고려하여 케이스 확장성과 가독성 유지 보수성을 높혀보자.


# Reference

---

- <a href="https://junit.org/junit5/docs/current/user-guide/#writing-tests-parameterized-tests" target="_blank">https://junit.org/junit5/docs/current/user-guide/#writing-tests-parameterized-tests</a>
