---
layout: post
date: 2023-09-03
categories: [Spring Data JPA]
title:  "JPA 에서 한 요청에 여러 트랜잭션을 처리 할 때 주의할 점"
img_path: /img/path
---

## 들어가며

쇼핑몰 프로젝트를 진행하면서 하루에 한 번 상품의 재고 Entity 를 데이터베이스로 부터 영속성 컨텍스트로 가져온 뒤 재고 Entity 를 조건에 따라  Delete 하거나 Update 해야 했다. 이때 상품 Entity 와 재고 Entity 는 OneToMany 관계이다.

상품 하나에 재고 Entity 를 Delete 하거나 Update 로직이 하나의 트랜잭션 안에서 동작하고 이를 1만 개의 상품에 대해서 수행해야 한다고 했을 때 로직은 다음과 같다.

- `StockScheduler.class`

```java
    ...

    final List<Long> productIds = productRepository.findAllIds(); // 1만 개 상품 ID 를 가져온다.
    for (Long productId : productIds) {		
        stockProcessingService.doStockProcess(productId);
    }

    ...
```

- `StockProcessingService.class`

```java

@Transactional
public void doStockProcess(final Long productId) {
	
    Product product = getProductById(productId);

    // 상품의 재고를 찾아온 뒤 Delete 혹은 Update
	
    List<Stock> stocks = findStocksByProduct(product); 
    for (Stock stock : stocks) {
          ...
    	updateStockQuantity(stock);
    }	
    deleteStock(stock);
}
```

필자는 상품 재고를 찾아온 뒤 Delete 혹은 Update 하는 `doStockConsistencyProcess` 로직을 최적화하기 위해 성능 측정을 계획했다.

이는 개발 환경 서버에서 수행되어야 했으므로 ~~굉장히 원초적인 방법으로~~ `/scheduler` 라는 앤드포인트를 설계하고 요청을 보내 트랜잭션 단위로 걸린 시간을 로깅으로 측정하였다.

![](/osiv/request.png)

- `StockProcessingService.class`

```java

@Transactional
public void doStockProcess(final Long productId) {
    
    Product product = getProductById(productId);
    long startTime = currentTimeMillis();
   
    // 상품의 재고를 찾아온 뒤 Delete 혹은 Update
    List<Stock> stocks = findStocksByProduct(product); 
    for (Stock stock : stocks) {
          ...
    	updateStockQuantity(stock);
    }	
    deleteStock(stock);

    log.info("doStockConsistencyProcess 걸린 시간 = {}ms ", currentTimeMillis() - startTime);
}

```

## 문제점

1만 개의 상품에 대해 동기적으로 시간 측정을 했을 때 초기에는 걸린 시간이 평균적으로 56 ms 가 나오던 시간이 점점 오래 걸리기 시작하고 약 1000번째 상품에서 걸린 시간이 1300ms 이상이 소요되는 문제를 마주했다.

상품마다 재고 테이블에 DELETE , UPDATE 쿼리 전달하는 수는 거의 동일했다는 점에서 로직의 문제가 아니라 다른 문제가 있을 것이라고 생각했다. 팀원들을 포함해서 다른 우테캠 교육생 분들에게 문제 상황을 공유한 뒤 원인을 알아내려고 노력했다. 그때 우테캠 교육생 분들께서 “로직에 문제가 있는 것이 아니라면 영속성 컨텍스트와 관련이 있어보인다.” 라고 단서를 주었고 곧바로 트랜잭션이 끝난 뒤 영속성 컨텍스트를 비우는 로직을 추가하고 성능 테스트를 진행해 보았다.

- `StockProcessingService.class`

```java
@Transactional
public void doStockProcess(final Long productId) {
    
    Product product = getProductById(productId);
    long startTime = currentTimeMillis();

    // 상품의 재고를 찾아온 뒤 Delete 혹은 Update
    List<Stock> stocks = findStocksByProduct(product); 
    for (Stock stock : stocks) {
          ...
    	updateStockQuantity(stock);
    }	
    deleteStock(stock);
    log.info("doStockConsistencyProcess 걸린 시간 = {}ms ", currentTimeMillis() - startTime);
	
    // 영속성 컨텍스트 비우기
    entityManager.flush();
    entityManager.clear();
}
```

놀랍게도 결과는 상품 1만개에 대해 평균적으로 56 ~ 60 ms 가 소요되었다.

## 문제점의 원인은 OSIV

OSIV (Open In View Session) 은 View 레이어에서도 Session 을 유지하겠다는 의미로 영속성 컨텍스트와 트랜잭션은 일반적으로 같은 생명주기를 가지는데 OSIV 가 true 인 경우 트랜잭션이 닫히더라도 View 레이어까지 영속성 컨텍스트를 유지하는 기능을 말한다.

프로젝트에서 OSIV 설정을 별도로 해준 적이 없고 기본 설정이 true 이므로 `StockProcessingService` 의 `doStockProcess` 에서 트랜잭션이 종료되더라도 영속성 컨텍스트는 계속해서 유지가 되어 하나의 요청에 수행한 트랜잭션이 많아질수록 영속성 컨텍스트에 엔티티들이 계속해서 쌓이게 되어 오버헤드가 발생해 시간이 오래 걸린 것이었다.

즉 OSIV 옵션이 true 인 상태에서 하나의 요청에 대해 여러 트랜잭션을 호출하게 된다면 영속성 컨텍스트가 계속 유지되므로 의도하지 않았다면 주의해야 한다.

프로젝트 내에서 Controller, VIew 영역에서 지연로딩 등 영속성 컨텍스트를 유지할 필요가 없었으므로 `spring.jpa.open-in-view=false` 설정한 뒤 문제를 해결했다.

## 마치며

`StockScheduler` 클래스는 `@Scheduled(zone = "Asia/Seoul", cron = "0 0 0 * * ?")` 옵션을 사용하여 매일 자정에 로직을 수행하는데 이때에는 OSIV 설정이 동작하지 않으므로 이미 제대로 동작하고 있었다. 하지만 성능 측정을 위해서 API 요청 정의하고 호출하면서  한 요청에 여러 트랜잭션을 처리할 때 주의할 점에 대해 알게 된 뜻깊은 순간이었다.

번외로 OSIV 옵션이 true 인 경우 최초 데이터베이스 커넥션 시작 시점부터 API 응답이 끝날 때까지 데이터베이스 커넥션을 유지하므로 데이터베이스 병목을 줄이기 위해서라도 false 옵션을 기본적으로 가져가는 것이 좋다고 생각한다. 
