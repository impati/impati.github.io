---
layout: post
date: 2023-10-08
categories: [JPA]
title:  "JPA Entity 에서 equals() ,hashcode() 다시보기"
img_path: /img/path
---

## 들어가며

자바에서 equals() ,hashcode() 를 재정의하는 이유는 무엇일까? 생성한 객체의 동일성 수준의 같음을 넘어 동등성 수준의 같음을 원하기 때문이다.

여기서 동일성은 실제로 참조가 같은 객체를 참조하고 있는 경우를 말하며 동등성은 도메인 규칙에 의해 같은 객체이다라고 판단하는 기준을 말한다. 그리고 equals() ,hashcode() 는 항상 같이 재정의해줘야 하는데 이유는 [여기](https://tecoble.techcourse.co.kr/post/2020-07-29-equals-and-hashCode/)에서 알아볼 수 있다.

그렇다면 JPA Entity 에서 equals() ,hashcode() 는 재정의해야 할까? 해야한다면 어떻게 해야할까?

## JPA Entity 에서 equals() ,hashcode() 를 재정의해야하는 이유

JPA 의 영속성 컨텍스트는 데이터베이스에서 가져온 Entity 의 식별자가 이미 1차 캐시에 존재한다면 1차 캐시에 있는 Entity 를 반환하는 방법으로  어플리케이션 수준에서 **영속 상태인 엔티티의 동일성**을 보장해 준다.

즉, 같은 영속성 컨텍스트라면 equals() ,hashcode() 를 재정의해줄 필요 없이 올바른 엔티티 간 비교가 가능하다는 것을 의미한다. 하지만 모든 엔티티의 비교가 항상 같은 영속성 컨텍스트 안에서 이루어진다는 보장을 할 수 없기 때문에 JPA Entity 에서 equals() ,hashcode() 를 재정의해줘야 한다.

## equals() ,hashcode() 를 어떻게 정의해야할까

그렇다면 JPA Entity 에서 equals() ,hashcode() 를 어떻게 정의해야 할까?

이 물음에 대한 정답은 없다고 생각한다. 앞서 언급한 것처럼 “도메인 규칙에 의해 같은 객체이다라고 판단하는 기준”을 정하기 나름이기 때문이다.

> 보통은 데이터베이스 동등성 비교에서 사용되는 @Id 값은 엔티티를 영속화해야 식별자를 얻을 수 있기 때문에 null 값인 경우 정확한 Entity 비교가 불가능하다는 점에서 중복이 없고 거의 변하지 않는 값들인 **비지니스 키** 값이 동등성 비교에 적합하다고 한다. - 자바 ORM 표준 JPA 프로그래밍
> 

만약 모든 Entity 에 적합한 비지니스 키가 없다는 가정하고 Entity 의  @Id 값이 null 인 경우 다른 값들이 같아도 다른 Entity 로 본다는 도메인 규칙안에서 equals() ,hashcode() 를 재정의한다면 어떨지 , 그리고 equals() , hashcode() 를 재정의하는데 주의할 점은 없는지 알아보자.

 

## @Id 값을  equals() ,hashcode(**)**로 재정의

### Product Entity 와 테스트

먼저 equals() , hashcode() 재정의를 위해 간단한 상품과 테스트 케이스를 만들어보자.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    public Product(final String name) {
        this.name = name;
    }
    
    @Override
    public boolean equals(final Object o) {
        if (this == o) {
            return true;
        }
        if (o == null) {
            return false;
        }
        if (this.getClass() != o.getClass()) {
            return false;
        }
        
        Product product = (Product)o;
        return Objects.equals(id, product.id);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

상품 Entity 는 상품 이름과 id 를 가지고 equals(),hashcode() 를 위와 같이 정의 했다.

```java

@Test
@DisplayName("엔티티 비교에서 프록시 객체의 여부와 상관 없이 ID 가 같다면 같은 entity 여야한다.")
void product_equals() {
    Product product = getPersistProduct("hello");  // ID 가 있는 product 를 가져옴
    Product proxyProduct = getProxyProduct(product.getId()); // product ID 로 프록시 객체를 가져옴

    assertThat(product).isEqualTo(proxyProduct);
    assertThat(proxyProduct).isEqualTo(product);
}

private Product getPersistProduct(final String name) {
    Product product = new Product(name);
    entityManager.persist(product);
    entityManager.flush();
    entityManager.clear(); // 영속성 컨텍스트를 비움
    return product;
}

private Product getProxyProduct(final Long productId) {
    return entityManager.getReference(Product.class, productId);
}
```

그리고 엔티티가 프록시 객체인지 여부와 관계없이 ID 가 같으면 항상 같은 Entity 여야하고 따라서 위 테스트는 성공해야 한다.

### 프로시 타입 비교에서 주의할 점

하지만 위 테스트는 실패하게 되는데 그 이유는 프록시로 조회한 엔티티의 타입은 원본 타입을 상속받아 구현한 것이므로 타입간 동등 비교에서 false 인 것이다. 원본 엔티티라면 문제가 되지 않지만 JPA Entity 에서는 지연로딩을 위한 연관관계에서 프록시 엔티티를 가지고 있을 수 있으므로 다음과 같이 equals 를 재정의해줘야 한다.

```java
@Override
public boolean equals(final Object o) {
    if (this == o) {
        return true;
    }
    if (!(o instanceof Product product)) {
        return false;
    }
    
    return Objects.equals(id, product.id);
}
```

그럼에도 불구하고 위의 테스트는 여전히 실패하는 것을 알 수 있다. 일반적으로 equals()를 구현할 때는 멤버 변수를 직접 비교하는데 프록시의 경우에는 실제 데이터를 필드로 가지고 있지 않아 직접 접근하면 아무 값도 조회할 수 없기 때문이다. 결국 위 equals 메서드에서 비교 대상이 프록시 라면 id 값이 null 인 것이다. 이를 해결하기 위해서는 프록시의 데이터를 조회할 때 다음과 같이 접근자를 이용하여 비교해야 한다.

```java
@Override
public boolean equals(final Object o) {
    if (this == o) {
        return true;
    }
    if (!(o instanceof Product product)) {
        return false;
    }
    
    return Objects.equals(getId(), product.getId());
}
```

마지막으로 @Id 값이 null 인 경우 다른 값들이 같아도 다른 Entity 로 본다는 도메인 규칙을 적용한 equals(),hashcode() 는 다음과 같다.

```java
@Override
public boolean equals(final Object o) {
    if (this == o) {
        return true;
    }
    if (!(o instanceof Product product)) {
        return false;
    }
    
    return this.getId() != null && Objects.equals(getId(), product.getId());
}

@Override
public int hashCode() {
    return Objects.hash(getId());
}
```

## 마치며

프록시 타입 비교에서 주의할 점은 비지니스 키를 이용한 동등성 비교에도 똑같이 적용된다. 다만 동등성 비교에 사용되는 키값이 null 인 경우 같은 값으로 볼 것인지 ,다른 값으로 볼 것인지는 규칙을 어떻게 정하는지에 따라 다르다고 생각한다. 예제를 id 로 한 이유는 개인적으로 entity 는 id 값이 같아야 동등성 비교에서 true 를 반환하는 것이 자연스럽다고 생각했기 때문이다.

## Reference

---

- [자바 ORM 표준 JPA 프로그래밍](https://www.yes24.com/Product/Goods/90439472)