---
layout: post
date: 2023-06-15 
categories: [test]
title:  "테스트 인스턴스를 랜덤 생성하여 테스트해보자"
---

# 들어가며
---
객체를 생성하는 테스트뿐만 아니라 조회, 삭제, 수정 등의 API를 테스트할 때에도 객체를 생성하는 작업이 선행된다.

특히, 객체가 여러 곳에서 사용되는 경우 테스트에서도 객체를 생성하는 로직이 중복으로 이어질 수 있다.

다음은 객체 생성이 얼마나 자주 일어나는지 게시글 Entity를 Builder로 매번 생성하는 코드를 보여준다.


### 수정 테스트
```java
@Test
@DisplayName("게시글 수정")
public void editArticleTest() throws Exception {
    // given
    Article article = Article.builder()
            .nickname("impati")
            .title("에러 발생!")
            .content("거짓말이에요")
            .boardType(BoardType.ERROR)
            .customerId(1L)
            .build();
	      
		...
        ...
}
```

### 삭제 테스트
```java
@Test
@DisplayName("게시글 삭제 테스트")
public void removeArticleTest() throws Exception {
    // given
    Article article = Article.builder()
            .nickname("impati")
            .title("에러 발생!")
            .content("거짓말이에요")
            .boardType(BoardType.ERROR)
            .customerId(1L)
            .build();
	      
		...
        ...
}
```

### 조회 테스트
```java
@Test
@DisplayName("게시글 제목으로만 검색")
public void searchOnlyTitleName() throws Exception {
    // given
    List<Article> articles = new ArrayList<>();

    articles.add(Article.builder()
            .nickname("impati")
            .title("안녕하세요")
            .content("안녕하세요")
            .boardType(BoardType.ETC)
            .customerId(1L)
            .build());

    articles.add(Article.builder()
            .nickname("impati")
            .title("반가워요")
            .content("반가워요")
            .boardType(BoardType.ERROR)
            .customerId(1L)
            .build());
    ...
    articleRepository.saveAll(articles);

    ...
	...

}
```
게시글 API 테스트에서만 객체를 생성하는 것이 아닐 수 있으므로 Article 객체를 생성 로직이 테스트 전역에 중복되는 문제가 발생한다. 이는 나중에 Builder를 사용하는 것이 아니라 정적 팩토리 메서드로 Article 을 생성하는 것으로 변경한다면 무수히 많은 테스트 코드를 고쳐야 하는 문제로 이어진다.
테스트에서 게시글 생성을 담당하는 클래스를 만들어 게시글 객체 생성을 한곳으로 모으는 것으로 이 문제를 해결할 수 있다.

```java
public class ArticleFixture {

    public static final BoardType DEFAULT_BOARD_TYPE = BoardType.ERROR;
    public static final Long DEFAULT_CUSTOMER_ID = 1L;
    public static final String DEFAULT_TITLE = "에러 발생했습니다";
    public static final String DEFAULT_CONTENT = "문세 상황 : ... ";
    public static final LocalDateTime DEFAULT_CREATED_AT = now();
    public static final String DEFAULT_NICKNAME = "impati";

    public static Article createDefaults() {
        return Article.builder()
                .customerId(DEFAULT_CUSTOMER_ID)
                .boardType(DEFAULT_BOARD_TYPE)
                .title(DEFAULT_TITLE)
                .content(DEFAULT_CONTENT)
                .nickname(DEFAULT_NICKNAME)
                .createAt(DEFAULT_CREATED_AT)
                .updatedAt(DEFAULT_CREATED_AT)
                .build();
    }

}
```
ArticleFixture 클래스를 만들고 DEFAULT 값으로  Article 객체를 생성한 후 이를 사용한다.

### 수정 테스트
```java
@Test
@DisplayName("게시글 수정")
public void editArticleTest() throws Exception {
    // given
    Article article = createDefaults();
        ...
}
```

### 삭제 테스트
```java
@Test
@DisplayName("게시글 삭제 테스트")
public void removeArticleTest() throws Exception {
    // given
    Article article = createDefaults();
        ...
}
```
ArticleFixture 에서 기본 Article 객체 생성을 캡슐화하여 제공하므로 Article 을 생성하는 방법에 대한 의존성을 없앴고 보다 간결하게 본래 테스트에 집중할 수 있게 되었다.
### 조회 테스트
```java
@Test
@DisplayName("게시글 제목으로만 검색")
public void searchOnlyTitleName() throws Exception {
    // given
    List<Article> articles = new ArrayList<>();

    articles.add(createDefaults());
    ...
    articleRepository.saveAll(articles);

    ...
	...

}
```
조회할 때에는 다양한 값을 가진 다양한 Article 객체를 생성하고 이를 테스트해야한다. \
ArticleFixture 클래스에서 다양한 Article 객체를 생성하기 위해서는 어떻게 해야할까 ?

# 동적으로 Article 객체를 생성하기 
---

```java
public class ArticleFixture {

    public static final BoardType DEFAULT_BOARD_TYPE = BoardType.ERROR;
    public static final Long DEFAULT_CUSTOMER_ID = 1L;
    public static final String DEFAULT_TITLE = "에러 발생했습니다";
    public static final String DEFAULT_CONTENT = "문세 상황 : ... ";
    public static final LocalDateTime DEFAULT_CREATED_AT = now();
    public static final String DEFAULT_NICKNAME = "impati";

    public static Article createDefaults() {
        return Article.builder()
                .customerId(DEFAULT_CUSTOMER_ID)
                .boardType(DEFAULT_BOARD_TYPE)
                .title(DEFAULT_TITLE)
                .content(DEFAULT_CONTENT)
                .nickname(DEFAULT_NICKNAME)
                .createAt(DEFAULT_CREATED_AT)
                .updatedAt(DEFAULT_CREATED_AT)
                .build();
    }

}
```
ArticleFixture 클래스에서 Article 빌더까지 생성한 뒤 필요한 값만을 설정해서 사용한다면 동적으로 Article 객체를 사용할 수 있다.

### 조회 테스트
```java
@Test
@DisplayName("게시글 제목으로만 검색")
public void searchOnlyTitleName() throws Exception {
    // given
    List<Article> articles = new ArrayList<>();

    articles.add(createDefault().title("안녕하세요").build());
    articles.add(createDefault().title("오류입니다.").build());
    articles.add(createDefault().title("반가워요").build());
    ...
    articleRepository.saveAll(articles);

    ...
	...

}
```
다른 값들은 기본값을 사용하되 title 값만을 설정해서 Article 객체를 생성하고 테스트 해볼 수 있게되었다.

# 문제점
---
동적으로 Article 객체 생성하기에서 문제점을 찾아보자

사용하는 입장에서 title(” … ”) 값만 설정하면 편리하다는 이점이 있지만 build() 를 통해서 객체를 생성하는 방법에 의존하게 되었다.
이를 ArticleFixture 에서 메서드를 제공함으로써 `ArticleFixture.createWithTitle(String title)` 캡슐화할 수 있지만 결국 title  , nickname, boardType … 의 속성을 모두 다르게 설정해 주어야 한다면 ArticleFixture 에서 Article 객체를 생성하는 API  가짓수가 많아지는 문제를 해결하지 못한다.ArticleFixture 의 도입으로 Article 객체 생성을 캡슐화하여 객체 생성 책임을 한 곳으로 모을 수 있었지만 보다 다양한 값을 설정해야하는 Article 객체 생성에는 여전히 문제가 있어보인다.


# EasyRandom 으로 문제해결
---
다양한 값을 설정해야하는 경우에 랜덤으로 Article 객체를 생성한다면 복잡하고 반복적인 API 를 만들지 않아도 된다.
EasyRandom 를 이용한다면 랜덤으로 Article 객체를 생성할 수 있어 간단하게 문제를 해결할 수 있다.

EasyRandom 는 리플렉션을 사용하여 자바 객체를 랜덤으로 생성해주는 라이브러리이다.

EasyRandom 을 사용하면 복잡한 빌더나 여러 파라미터를 가진 API 없이도  다양한 값을 가지는 Article 객체를 생성할 수 있다.

EasyRandom 을 사용하는 방법은 아주 간단하다.

먼저 gradle 의존성을 추가해준다.

```gradle
testImplementation 'org.jeasy:easy-random-core:5.0.0'
```

```java
public class ArticleFixtureFactory {

    public static Article createRandom() {
        EasyRandomParameters params = new EasyRandomParameters()
                .seed(new Random().nextLong())
                .objectPoolSize(10)
                .charset(StandardCharsets.UTF_8)
                .excludeField(FieldPredicates.named("id"))
                .stringLengthRange(5, 20)
                .ignoreRandomizationErrors(true);
        EasyRandom easyRandom = new EasyRandom(params);
        return easyRandom.nextObject(Article.class);
    }
}
```

- EasyRandomParameters : EasyRandom 인스턴스를 구성하는 객체로 파라미터를 설정하여 [랜덤 데이터 생성 방법을 설정할 수 있다.](https://github.com/j-easy/easy-random/wiki/Randomization-parameters)
    
- EasyRandom : 랜덤한 자바 객체를 생성하는 객체

기본값만을 사용해서 테스트를 진행해도 되지만 Aritlcle 의 id 값은 중복되면 안되니 [exclude](https://github.com/j-easy/easy-random/wiki/excluding-fields) 해주어 사용한다.

이외에도 생성되는 문자열 필드 값의 길이를 지정하여 문자열을 생성한다거나 시간 필드값에 범위를 줄 수 있다.

[지원되는 타입](https://github.com/j-easy/easy-random/wiki/Supported-types)이나 , [Bean Validaion](https://github.com/j-easy/easy-random/wiki/bean-validation-support) 등 다양한 내용이 공식 문서에 잘 나와있다.


### 조회 테스트
```java
@Test
@DisplayName("게시글 제목으로만 검색")
public void searchOnlyTitleName() throws Exception {
    // given
    List<Article> articles = new ArrayList<>();

    articles.add(ArticleFixtureFactory.createRandom());
    articles.add(ArticleFixtureFactory.createRandom());
    articles.add(ArticleFixtureFactory.createRandom());
    ...
    articleRepository.saveAll(articles);

    ...
	...

}
```

다양한 값을 가지는 복잡한 객체를 생성한 뒤 테스트를 수행해야한다면 EasyRandom은 좋은 선택이 될 수 있다.


# 마무리하며
---
다양한 값을 가지는 객체를 생성한 뒤 테스트를 수행해야 한다면 EasyRandom은 좋은 선택이 될 수 있지만 테스트에서 직관적으로 흐름을 보여주기에는 한계가 있어 보인다. 어떤 값을 가지는 객체가 생성되었는지 알기 어렵고 매번 다르기 때문이다.


# Reference
---
- https://github.com/j-easy/easy-random