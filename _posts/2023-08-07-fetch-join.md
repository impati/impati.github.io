---
layout: post
date: 2023-08-07
categories: [Spring Data JPA]
title:  "상황을 고려하며 페치조인 적용하기"
img_path: /img/path
---

## 들어가며 
우아한 테크 캠프에서 쇼핑 미션을 진행하며 리뷰어분께 다음과 같은 리뷰를 받았었다.

![](/fetch/review.png)

먼저 `EntityGraph` 를 소개하자면 엔티티에 대해 어노테이션을 활용하여 연관관계에 있는 엔티트의 Fetch Plan 을 구성할 수 있도록 해준다.기본 type 값으로는 `EntityGraphType.FETCH` 이며 `attributePaths = {“필드 명” , ...}` 으로 설정할 수 있다.

나는 미션을 진행하면서  다음과 같은 쿼리에 `EntityGraph` 옵션을 주었다.

```java
public interface CartItemRepository extends JpaRepository<CartItem, Long> {

	...

    @EntityGraph(attributePaths = {"product", "member"})
    Optional<CartItem> findById(final Long cartItemId);

    @EntityGraph(attributePaths = {"product"})
    List<CartItem> findAllByMemberId(final Long memberId);
		
	...
}
```

  

이때 리뷰어님은 왜 `findAllByMemberId` 에서는  `@EntityGraph(attributePaths = {"product"})` 으로 `findById` 에서는     `@EntityGraph(attributePaths = {"product", "member"})` 으로 설정했는지와 그리고 `EntityGraph` 을 사용하면 즉 , `attributePaths` 에 설정한 연관관계에 있는 엔티티들을 fetch join 하면 어떤 이점이 있는지에 대해 물어본 것 같다고 생각했다.

이 질문을 받고 난 뒤 fetch join 을 사용할 때 고민을 해보았나? 라는 측면에서 되돌아보게되었고 ,  어떤 상황에서 fetch join 을 하면 좋을지 탐구해보았다.


## CartItem 에서 LAZY 로딩의 이유

---

먼저 `CartItem` 과 연관관계에 있는 `member`, `product` 를 왜 `LAZY` 로딩을 했는지 이해하는 것이 중요하다.

- `CartItem.class : EAGER`

```java
@Entity
public class CartItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "cart_item_id")
    private Long id;

    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "member_id")
    private Member member;

    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "product_id")
    private Product product;

 ...
```

`cartItem` 의 `member` 와 `product` fetch 옵션을 `EAGER` 로 설정하고 `cartItem` 을 `findByid` 로 가져오면 다음과 같은 쿼리가 발생한다.

```sql
select * 
from cart_item
left outer join member on cartitem.member_id = member.member_id 
left outer join product on cartitem.product_id = product.product_id
where cartitem.cart_item_id = ?
```

만약 `Member Entity` 에 `EAGER` 로 로딩하는 연관관계 Entity 가 더 있다면 ? 쿼리는 더 복잡? 무거워질 것이다.

- `Member.class`

```java
@Entity
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;

   ...

    @OneToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "box_id")
    private Box box;

   ...
```

`Member` 에 1 : 1 관계의 `Box` 앤티티가 있다면 다음과 같은 쿼리를 전달한다.

```java
select * 
from cart_item
left outer join member on cartitem.member_id = member.member_id 
left outer join box on member.box_id = box.box_id 
left outer join product on cartitem.product_id = product.product_id
where cartitem.cart_item_id = ?
```

- `CartItem.class : LAZY`

```java
@Entity
public class CartItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "cart_item_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private Member member;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

 ...
```

`CartItem` 에서 `fetch` 옵션을 `LAZY` 로 설정한 이유는 을 가져온 뒤 항상 `member` 와 `product` 에 대한 참조가 필요하지는 않으므로 join 쿼리를 사용할 필요가 없고 ,뿐만 아니라 모두 `EAGER` 로 설정되어 있다면 `CartItem` 에서 참조하고 있는 Entity 가 참조하고 있는 다른 Entity 의 객체도 Join 해서 가져올 가능성이 있기 때문에 이를 원초적으로 예방하고자 기본적으로 `LAZY` 로 설정하였다.


## LAZY 로딩하면 발생하는 일

`CartItem` 에서 `Member` `Product` 모두 `fetch` 옵션이 `LAZY`로 설정되어 있기 때문에 `CartItem` 를 `EntityGraph`없이 (fetch join 없이) 가져오는 쿼리를 작성하는 경우 `CartItem` 에서 `member` 혹은 `product` 를 사용할 때마다 `member` 혹은 `product` 에 대한 조회 쿼리가 발생한다.


## EntityGraph 를 사용하면 얻는 이점

`CartItem` 을 가져온 뒤 `member` 와 `product`에 대한 참조가 잦은 경우 `EntityGraph` 를 사용하여 실제 Entity 를 가져오면 이후에  참조할때마다 쿼리가 발생하지 않는다는 장점이 있다.

```java
public interface CartItemRepository extends JpaRepository<CartItem, Long> {
    ...
    @EntityGraph(attributePaths = {"product"})
    List<CartItem> findAllByMemberId(final Long memberId);
}

```

```java
public static CartItemResponse from(final CartItem cartItem) {
        return new CartItemResponse(
                cartItem.getId(),
                cartItem.getQuantity(),
                ProductResponse.from(cartItem.getProduct()) //  fetch join 을 하지 않으면 CartItem 수 만큼 쿼리가 발생할 가능성이 있다.
        );
    }

```

`findAllByMemberId` 메서드로 가져온 `CartItem` 에서 조회된 `CartItem 수` 만큼 cartItem.getProduct()) 를 참조하기 때문에 `CartItem 의 수` 만큼 `product` 조회 쿼리가 발생해야하지만 `@EntityGraph(attributePaths = {"product"})` 설정 덕분에 1번의 쿼리만 발생한다.


## EntityGraph 를 사용하면 잃는 점

`CartItem` 에서 `Member` 혹은 `Product` 를 참조하지 않아도 Join 쿼리가 발생한다. 즉 , 불필요한 조인 쿼리가 발생한다.


## EntityGraph를 사용한 findById 의 상황

```java
public interface CartItemRepository extends JpaRepository<CartItem, Long> {
    ...
    @EntityGraph(attributePaths = {"product","member"})
    Optional<CartItem> findById(final Long cartItemId);
}

```

`EntityGraph`를 사용한 `findById` 는 현재 다음과 같은 상황에서만 사용한다.

```java
[CartItemService.class]

    ...
    final Member member = getMemberById(memberId);
    final CartItem cartItem = getCartItemById(cartItemId);
    cartItem.validateMember(member);
    ...

================================================
[CartItem.class]

    public void validateMember(final Member member) {
        if (!this.member.equals(member)) {
            ...
        }
        
        ...
    }
```

이 경우 `attributePaths="product"` 는 불필요하다.  필요 없음에도 join 해오고 있었던 것이다!

`validateMember()` 에서 먼저 가져온 `Member` 와  `CartItem` 의 `Member` 가 같은지를 판단하는 로직에서 `CartItem` 의 `Member`에 대한 참조가 이루어진다. 다만 영속성 컨텍스트가 유지되는 범위안에서  `CartItemService` 에서 먼저 가져온 `Member` 와  `CartItem` 의 `Member` 가 같은 경우 먼저 가져온 `Member` 가 1차 캐시 되었으므로 추가적인 쿼리가 발생하지 않고 먼저 가져온 `Member` 와  `CartItem` 의 `Member` 가 다른 경우만 추가적인 쿼리가 발생한다.
이 시점에서 고민해볼 수 있는 점은 다음과 같다.

1. `findById` 에서 `EntityGraph`를 사용하지 않고 먼저 가져온 `Member` 와 `CartItem` 의 `Member` 가 다른 경우에만 추가적인 쿼리를 보내고 `CartItem`의 `findById` 쿼리에 `join`을 사용하지 않는 방법
2. `findById` 에서 `EntityGraph`를 사용하여 항상 `member` 에 대한 `join` 쿼리를 사용하고 항상 추가적인 쿼리를 보내지 않는 방법


## 일반적으로 어떤 요청이 더 많을까 ?

먼저 가져온 `Member` 와  `CartItem` 의 `Member` 가 다른 경우보다 같은 경우가 전체 요청 중에서 더 많을 것이라는 예상은 쉽게 해볼 수 있을 것 같다. 왜냐하면  `Member` 와 `CartItem` 의 `Member` 가 다른 경우는 실제로 호출하는 클라이언트가 없는 방어적인 코드이기 때문이다.
이러한 이유로 `findById` 에서 `EntityGraph`를 사용하지 않는 것이 더 낫다는 판단을 할 수 있다.

## 결론

현재 쇼핑몰 미션의 `cartItem`의  `findById` 에서 fetch Join을 사용하지 않는다! 

```java
public interface CartItemRepository extends JpaRepository<CartItem, Long> {

    Optional<CartItem> findById(final Long cartItemId); // 이미 정의되어 있으므로 제거한다.

    @EntityGraph(attributePaths = {"product"})
    List<CartItem> findAllByMemberId(final Long memberId);
}

```

## 마무리하며

이러한 사실을 코드를 작성할 때는몰랐나? 라고 하면 아니다 . 알고 있었다.

하지만 어느 상황에서 어떻게 사용하면 더 좋을지에 대한 고민을 하지 않아서  `findById` 에   `@EntityGraph(attributePaths = {"product", "member"})` 을 주었던 것 같다.오히려 주지 않았다면 더 좋았음에도 불구하고. 

이번 탐구를 계기로 fetch join 을 사용하면 좋을지 안좋을지에 대해 많이 생각해볼 수 있었고 앞으로 사용할 때에도 탐구하면서 적용한다면 쿼리가 무수히 많이 나가거나 너무 무겁게 나가는 일은 막을 수 있지 않을까 ?

