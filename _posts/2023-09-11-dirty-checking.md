---
layout: post
date: 2023-09-11
categories: [Spring Data JPA]
title:  "JPA에서 변경 감지 사용 안 하기"
img_path: /img/path
---

## 들어가며
---
JPA는 트랜잭션 내 영속성 컨텍스트에서 관리되는 엔티티의 변경이 일어났을 때 update 쿼리를 전달하지 않아도 변경 감지 기능에 의해 update 쿼리를 전달해 준다.

이러한 일이 가능한 이유는 엔티티를 영속성 컨텍스트에 보관할 때 최초 상태를 복사해서 저장해 두는 스냅샷을 관리하기 때문이다. 그리고 데이터베이스에 영속성 컨텍스트가 변경된 내용을 반영하는 시점에 현재 엔티티와 스냅샷을 엔티티의 컬럼이 같은지를 판단해서 update 쿼리를 전달하기 때문에 개발자는 별도의 update 쿼리를 작성하지 않아도 되는 것이었다.

여기서 만약 트랜잭션 내 영속성 컨텍스트에 가져온 엔티티가 모두 변경될 수 있는 가능성이 있다면 업데이트 쿼리는 몇 번 발생할까? \
가져온 엔티티가 N 개이고 이 엔티티들이 모두 변경되었다면 N 번의 update 쿼리가 발생할 수 있다. 성능이 중요한 로직이라면 개선의 여지가 필요해 보인다.

## N 번 쿼리의 위험성
---
프로젝트를 진행하면서 상품 테이블의 수량과 재고 테이블의 수량의 정합성을 맞추는 로직이 있었다. \
먼저 상품 테이블과 재고 테이블의 관계는 다음과 같다.

![](/dirty/erd.png)

재고 테이블은 유통기한과 수량으로 상품에 대한 재고를  관리하고 상품 테이블의 quantity 는 재고 테이블의 quatity 의 합 즉, 반정규화를 나타낸다. \
프로젝트에서는 특별한 방법으로 재고를 관리하는데 주문 시점에 상품 quantity 에만 재고 감소를 수행하고 재고 테이블에는 접근하지 않는 방법이다. 이러한 이유들로 특정 시간에 상품 quantity 와 재고 테이블의 quantity 합의 정합성을 맞춰줄 필요가 있었고 로직은 다음과 같다.

```java

@Transactional
public void run(final Product product) {
    // 1.재고 테이블에서 현재 상품의 재고를 모두 가져온다.
    List<Stock> stocks = stockRepository.findAllByProductId(product.getId());

    // 2.상품 재고 총합과 상품 quantity 의 차이를 구한다.
    Quantity difference = computeTotalStockQuantity(stocks).subtract(product.getQuantity());
		
    // 3.차이 만큼 재고 수량을 업데이트 해준다.
    for (Stock stock : stocks) {
        ...
    }
		
}
```
이때 상품의 quantity 와 재고 테이블의 상품 총 재고 수량의 차이가 커지면서 difference 값이 크다면 영향을 받은 stock 들이 더욱 많아질 것으로 예상된다.

![](/dirty/table.png)

예를 들어 위와 같이 상품 quantity 가 15개이고  재고 테이블의 재고 수량이 10개씩 6개 레코드가 존재했을 때 UPDATE 쿼리는 5번 발생한다.
단순한 문제로 여겨질 수 있지만 영향을 받을 재고 레코드를 예측할 수 없을뿐더러 프로젝트 내에서는 1만 개의 상품에 대해 위와 같은 로직을 수행하고 있었기 때문에 N * 10000 쿼리가 발생한다는 것을 의미했고 이를 개선할 필요가 있었다.

## IN 쿼리로 전환
---
위 로직에서 N번의 쿼리가 발생한 이유는 JPA 의 변경 감지기능을 사용했기 때문이다. 이는 쿼리를 아예 작성하지 않아도 되는 편리함을 가져다주었지만 해당 로직은 빨리 끝날수록 좋았기 때문에 재고 수량이 0 되는 재고의 ID 를 모아서 한번의 IN 쿼리로의 전환을 계획했다.

```java

@Transactional
public void run(final Product product) {
    // 1.재고 테이블에서 현재 상품의 재고를 모두 가져온다.
    List<Stock> stocks = stockRepository.findAllByProductId(product.getId());

    // 2.상품 재고 총합과 상품 quantity 의 차이를 구한다.
    Quantity difference = computeTotalStockQuantity(stocks).subtract(product.getQuantity());
		
    // 3.차이 만큼 재고 수량을 업데이트 해준다.
    for (Stock stock : stocks) {
        ...
    }
    
    // 4.재고 수량이 0인 stock의 id 를 모아 업데이트 쿼리를 전달
    stockRepository.zeroUpdateStockQuantity(...);	
}
```
앞서 3번에서 재고 수량을 변경 하고 난 뒤에 재고 수량이 0인 stock 의 id 를 모아 IN 쿼리를 사용해서 UPDATE 쿼리를 전달하면 재고가 0이되는 stock 레코드들에 대해서는 한번의 쿼리로 해결이 가능하다.

```java
@Modifying
@Query("update Stock s set s.quantity = 0 where s.id in :ids")
void zeroUpdateStockQuantity(@Param("ids") List<Long> stockIds);
```

## 현재 IN 쿼리에서 문제점
---
하지만 위 로직에서 큰 문제점이 하나 있다. 4번에서 벌크 연산을 수행하기 전 flush 를 자동으로 호출하는 옵션이 활성화되어 있기 때문에 벌크 연산을 수행하기 전에 어김없이 3번에서 변경된 재고 수량에 대한 변경 감지 UPDATE 쿼리가 전달되고 있는 것이었다. 즉 , 변경에 영향을 받은 쿼리에 변경 감지 쿼리는 계속 전달되고 있었다.

이러한 문제를 해결하기 위해서는 어떻게 해야 할까? \
**방법은 변경 감지 기능을 사용하지 않는 것이다.**  \
변경 감지를 사용하지 않는다면 3번에서 변경 감지 UPDATE 를 전달하지 않을 것이고 4번 벌크 연산 시 1번의 쿼리만을 전달할 것이다. \
(추가적으로 재고가 0이 되는 경우가 아닌 경우에는 단 한 번만 쿼리를 추가적으로 전달하면 된다)

## 변경 감지 사용 안 하기
---
변경 감지를 사용하지 않으려면 어떻게 해야 할까? \
1번에서 재고를 가져오는 로직을 현재 영속성 컨텍스트에서 준영속 상태로 만들어버리면 변경 감지 기능이 동작하지 않을 것이다.

### 재고를 가져온 뒤 EntityManager.clear() 로 영속성 컨텍스트 초기화

```java

@Transactional
public void run(final Product product) {
    // 1.재고 테이블에서 현재 상품의 재고를 모두 가져온다.
    List<Stock> stocks = stockRepository.findAllByProductId(product.getId());
    
    // 영속성 컨텍스트 초기화
    entityManager.clear();
    ...
}
```
재고를 가져온 뒤 영속성 컨텍스트를 entityManger 를 사용해서 초기화해주면 당장의 문제는 해결될 수 있다. 
즉, 변경 감지 기능을 이용하지 않으면서 N 번의 쿼리가 발생하던 문제를 1번의 IN 쿼리로 해결할 수 있었다.

### 재고를 가져온 뒤 EntityManager.detach() 로 재고 엔티티만 준영속화

하지만 영속성 컨텍스트를 초기화하는 entityManger.clear() 는 영속성 컨텍스트 내에 있던 모든 entity 를 삭제하는 행위이므로 사이드 이펙트를 발생시킬 수 있다. 이러한 문제를 해결하기 위해서는 특정 entity 만을 준영속화하는 detach를 사용하면 발생할 수 있는 사이드 이펙트를 줄일 수 있다.

```java
@Transactional
public void run(final Product product) {
    // 1.재고 테이블에서 현재 상품의 재고를 모두 가져온다.
     List<Stock> stocks = stockRepository.findAllByProductId(product.getId());
		
    // 조회한 재고 엔티티를 준영속화 시킨다.
    for(Stock stock : stocks){
        entityManager.detach(stock);
    }
	...

}
```

### 트랜잭션 전파 옵션으로 다른 영속성 컨텍스트에서 읽기

현재 영속성 컨텍스트 안에서 재고를 조회하는 것이 아닌 별도의 영속성 컨텍스트 안에서 조회하는 방법도 있다.

```java
@Service
@RequiredArgsConstructor
public class StockConsistencyProcessor {
    
    private final StockRepository stockRepository;
    private final StockFetcher stockFetcher;
    
    @Transactional
    public void run(final Product product) {
        // 1.재고 테이블에서 현재 상품의 재고를 모두 가져온다.
        List<Stock> stocks = stockFetcher.getStocks(product);
        
        ...
    }
    
    ...
}

=========

@Service
@RequiredArgsConstructor
public class stockFetcher {

    private final StockRepository stockRepository;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public List<Stock> getStocks(final Product product) {
        return stockRepository.findAllByProductId(product.getId());
    }
}
```

현재 트랜잭션과 다른 별도의 트랜잭션을 수행하겠다는 전파옵션인 `REQUIRES_NEW`을 사용해 준다면 재고를 가져온 stockFetcher 의 영속성 컨텍스트와 이를 호출하는 쪽에서 영속성 컨텍스트는 다른 영속성 컨텍스트이기 때문에 변경 감지가 작동하지 않는다. 하지만 이 방법은 하나의 스레드가 잠시라 할지라도 두 개의 커넥션을 사용하니 지양하는 것이 좋을 것 같다.

### 트랜잭션 전파 옵션으로 트랜잭션 없이 읽기

선언적 @Transactional 이 제공하는 전파 옵션은 여러 가지가 있는데 이 중에서 *`NOT_SUPPORTED`* 옵션을 사용한다면 비트랜잭션으로 실행하고 현재 트랜잭션이 있는 경우 일시 중지한다. 

```java
@Service
@RequiredArgsConstructor
public class stockFetcher {

    private final StockRepository stockRepository;

    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public List<Stock> getStocks(final Product product) {
        return stockRepository.findAllByProductId(product.getId());
    }
}
```

`NOT_SUPPORTED` 의 경우에는 트랜잭션 없이 읽기 때문에 이를 호출하는 입장에서 영속성 컨텍스트에 stocks 는 준영속상태이므로 변경 감지가 발생하지 않는다!

### 트랜잭션 없이 읽기

도메인 로직에 따라 다르겠지만 entityManager 나 트랜잭션 전파 옵션 없이도 트랜잭션이 시작하기 전에`stockRepository.findAllByProductId(product.getId())` 를 읽은 이후에  매개변수로 전달하는 형태로 구현한다면 재고 엔티티들을 준영속 상태에서 사용할 수 있다.

## 마무리하며
---
현재 프로젝트에서는 마지막 방법인 트랜잭션 시작하기 전에 재고를  읽어서 매개변수로 전달하는 형식으로 변경 감지를 사용하지 않았다. \
이렇게 변경 감지를 사용하지 않는 방법을 사용함으로써 N 번의 쿼리에서 한번의 IN 쿼리로  t4g.small 인스턴스에서 1만개 상품 , 상품 당 평균 30개의 레코드를 가진 상황에서의 성능을 7분 58초에서 3분 21초까지 개선할 수 있었다. \
변경 감지는 정말 편리하고 좋은 기능이라고 생각하지만 쿼리가 몇 번 발생할지 예측할 수 없는 상태라면 조금은 경계할 필요가 있다고 생각한다.

## Reference
---
- <a href="https://www.yes24.com/Product/Goods/90439472" target="_blank">자바 ORM 표준 JPA 프로그래밍</a>
