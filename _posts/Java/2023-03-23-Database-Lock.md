---
title:  "[Java] Database Lock을 이용해 동시성 제어하기"

categories: Java

toc: true
toc_sticky: true

date: 2023-03-23
last_modified_at: 2022-03-23
---

# Database Lock을 이용해 동시성 제어하기

## 동시성 (Race Condition)

Race Condition이란 두개 이상의 동시적인(Concurrent) 프로세스나 스레드들이 하나의 리소스에 접근하기 위해 경쟁하는 상태를 말한다.

이러한 상태가 발생했을때 자료의 일관성을 해치는 결과가 나타날 수 있다.

예를 들면 1개 남은 재고의 N개 이상의 프로세스(스레드)가 접근한다면? 겨우 1개의 재고가 남은 상품이 2번 최악의 경우 N번의 주문이 완료되는 상황이 생길 수 있다.

이러한 경우를 방지하려면 어떻게 해야할까?

여러가지 방법이 있겠지만, 여기선 Lock을 이용하여 Race Condition을 제어해보고자 한다.

Application에는 이러한 상태를 제어하기 위한 Lock을 지원한다.

그중 **낙관적 락 (Optimistic Lock)**과 **비관적 락 (Pessimistic Lock)**에 대해 알아보도록 하자.


## Lock

낙관적 락과 비관적 락에 대해 알아보기 전에 Database의 Lock에대해 간략하게 짚고 넘어가고자 한다.

### 공유 락 (Shared Lock)

공유 락은 **데이터를 변경하지 않는 읽기 명령에 대해 주어지는 락**으로, `Read Lock` 또는 `S Lock` 으로 표기하기도 한다.  
읽기의 경우 동시에 데이터에 접근해도 데이터의 일관성에 영향을 주지 않기 때문에, 공유락끼리는 동시에 접근이 가능하다.

### 베타 락 (Exclusive Lock)

베타 락은 **데이터에 변경을 가하는 쓰기 명령에 대해 주어지는 락**으로 `Write Lock` 또는 `X Lock` 으로 표기하기도 한다.  
Lock이 걸릴 경우 Lock이 해제될 때까지 SELECT를 포함한 모든 트랜잭션은 해당 데이터에 접근 할 수 없다.


낙관적 락과 비관적 락중 비관적 락은 이 두가지를 이용해 Application에서 Lock을 거는 형태로 지원된다.


## 낙관적 락

Optimistic은 사전적 의미로는 '낙관적인' 으로 트랜잭션이 서로 충돌하지 않을 것이라고 긍적적으로 가정하는 제어법을 의미한다.

낙관적 락은 위에서 설명한 DB에서 제공하는 락의 기능을 사용하지는 않고, JPA가 제공하는 버전 관리 기능을 사용한다. 그러므로 Entity에 version이라는 컬럼이 추가되어야 한다.

또한 낙관적 락은 트랜잭션을 커밋하기 전까지는 트랜잭션의 충돌을 알 수가 없다. 그러므로, 충돌 후 트랜잭션 롤백 및 재시작하는 로직은 개발자가 직접 구현을 해주어야 한다.

낙관적 락을 도식화 한다면 아래와 같다.

## 비관적 락

Pessimistic은 사전적 의미로는 '비관적인' 으로 트랜잭션이 충돌한다고 부정적으로 가정하여, Lock을 우선 걸고 실행하는 제어법을 의미한다.

DB에서 제공하는 락의 기능을 사용하며, 트랜잭션이 시작할 때 Shared Lock 또는 Exclusive Lock 을 걸고 시작한다. 그래서 쿼리를 살펴보면 SELECT ... FOR UPDATE 구문이 붙어서 요청되는걸 확인 할 수 있다.

정보를 조회 또는 수정단계에서 Lock을 걸기 때문에, 수정단계에서 충돌유무를 확인할 수 있다. 하지만 Lock에 의해 교착상태(DeadLock)에 빠질 위험이 있다.


출처 : [Optimistic vs. Pessimistic locking - Stack Overflow](https://stackoverflow.com/questions/129329/optimistic-vs-pessimistic-locking)


## 예시

이론적으로만 설명한다면 이해하는데 조금 어려움이 있을 것이라고 생각된다.

간략하게 예시를 들어보자.

먼저 Lock이 걸리지 않았을 경우 실패하는 테스트를 확인해보자.


- Entity

```java
@Entity
@NoArgsConstructor
public class Stock {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long productId;

    private Long quantity;

    public Stock(Long productId, Long quantity) {
        this.productId = productId;
        this.quantity = quantity;
    }

    public Long getQuantity() {
        return quantity;
    }

    public void decrease(Long quantity) {
        if (this.quantity - quantity < 0) {
            throw new RuntimeException("Low quantity!!");
        }
        this.quantity = this.quantity - quantity;
    }
}
```


- StockService

```java
@Transactional
public void decrease(Long id, Long quantity) {
    Stock stock = stockRepository.findById(id).orElseThrow();

    stock.decrease(quantity);

    stockRepository.saveAndFlush(stock);
}
```


- Test

```java
@BeforeEach
public void before() {
    Stock stock = new Stock(1L, 100L);
    
    stockRepository.saveAndFlush(stock);
}

@Test
void 동시에_100번_요청() throws InterruptedException {

    // given
    int threadCount = 100;
    ExecutorService executorService = Executors.newFixedThreadPool(32);
    CountDownLatch countDownLatch = new CountDownLatch(threadCount);

    // when
    for (int i = 0; i < threadCount; i++) {
        executorService.submit(() -> {
            try {
                stockService.decrease(1L, 1L);
            } finally {
                countDownLatch.countDown();
            }
        });
    }

    countDownLatch.await();

    Stock stock = stockRepository.findById(1L).orElseThrow();

    // then
    assertThat(stock.getQuantity()).isEqualTo(0L);
}
```

`ExecutorService`는 스레드를 생성하여 병렬처리 하는 방법이다. 32개의 스레드를 생성하여, 100개의 재고를 만들고 재고 감소 로직에 한번에 접근해보았다 결과는 어떻게 되었을까?


![Database Lock]({{site.url}}/assets/image/2023/2023-03/26-con001.png)

0이되어야 할 재고가 96개나 남아있었다!

이는 트랜잭션이 커밋되기 전에 재고 리소스에 접근하여, 감소되지 않은 데이터를 가져와 작업을 하다보니 이러한 이슈가 발생한것이다.

여기에 낙관적 락과 비관적 락을 적용해보도록 하자.


### 비관적 락

JPA에서는 Annotation을 통해 비관적 락을 지원하고 있다.

Repository의 쿼리 메소드에 `@Lock` Annotation을 추가해 주면 된다.

`@Lock` 어노테이션은 세가지가 있다.


- PESSIMISTIC_WRITE 

비관적락을 적용한다고 하면 일반적으로 해당 옵션을 준다고 생각하면 된다.  
해당 옵션을 적용한 뒤 나가는 쿼리문을 확인하면 `SELECT ... FOR UPDATE`의 쿼리문이 나가는것을 볼 수 있다.  
비관적 Lock, 쓰기 Lock이 적용된다.

- PESSIMISTIC_READ

잘사용하지는 않지만, 데이터를 반복 읽기만 하고 수정하지 않을때 사용한다.
해당 옵션을 적용하면 `SELECT ... FOR SHARE` 의 쿼리가 나가는것을 볼 수 있다.
비관적 Lock, 읽기 Lock이 적용된다.

- PESSIMISTIC_FORCE_INCREMENT

비관적 락중에 버전관리를 하는 옵션이다.  
비관적 락을 적용하지만, 버전정보를 강제적으로 증가시킨다.


여기서는 주로 사용하는 `PESSIMISTIC_WRITE`을 적용해 볼 것 이다.


- StockRepository

`@Lock(LockModeType.PESSIMISTIC_WRITE)` Annotation을 추가해 비관적 락을 적용한다.

```java
@Repository
public interface StockRepository extends JpaRepository<Stock, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from Stock s where s.id = :id")
    Stock findByIdWithPessimisticLock(Long id);
}
```


-StockServiceTest

```java
@BeforeEach
public void before() {
    Stock stock = new Stock(1L, 100L);
    
    stockRepository.saveAndFlush(stock);
}

@Test
void 동시에_100번_요청_by_PessimisticLock() throws InterruptedException {

    // given
    int threadCount = 100;
    ExecutorService executorService = Executors.newFixedThreadPool(32);
    CountDownLatch countDownLatch = new CountDownLatch(threadCount);

    // when
    for (int i = 0; i < threadCount; i++) {
        executorService.submit(() -> {
            try {
                stockService.decrease(1L, 1L);
            } finally {
                countDownLatch.countDown();
            }
        });
    }

    countDownLatch.await();

    Stock stock = stockRepository.findById(1L).orElseThrow();

    // then
    assertThat(stock.getQuantity()).isEqualTo(0L);
}
```

정상적으로 테스트가 성공하는것을 볼 수 있다.

![Database Lock]({{site.url}}/assets/image/2023/2023-03/26-con002.png)

QueryDsl의 경우 아래와 같이 `.setLockMode(LockModeType.PESSIMISTIC_WRITE)` 의 메소드 체이닝을 추가해주면 된다.

```java
@Override
public Optional<Stock> findByIdWithPessimisticLockDsl(Long id) {

    return Stock findStock = queryFactory
            .selectFrom(stock)
            .where(stock.id.eq(id))
            .setLockMode(LockModeType.PESSIMISTIC_WRITE)
            .fetch();
}
```

### 낙관적 락

비관적 락과 동일하게 JPA에서는 Annotation을 통해 지원해 주고 있다.

Repository의 쿼리 메소드에 `@Lock` Annotation을 추가해 주면 된다.

`@Lock` 어노테이션은 세가지가 있다.


- NONE

별도로 Lock 옵션을 넣지 않아도, Entity에 `@Version` 컬럼을 추가하면 기본으로 적용된다.

- OPTIMISTIC

Entity를 조회할 경우 버전을 체크한다. 한번 조회된 Entity가 트랜잭션이 커밋될 동안 변경되지 않음을 보장한다.

- OPTIMISTIC_FORCE_INCREMENT

낙관적 락을 사용하면서 버전 정보를 강제로 증가시킨다.


예시로는 `OPTIMISTIC`을 적용해 볼 것이다.

- StockRepository

`@Lock(LockModeType.OPTIMISTIC)` Annotation을 추가해 낙관적락을 적용한다.

```java
@Repository
public interface StockRepository extends JpaRepository<Stock, Long> {

    @Lock(LockModeType.OPTIMISTIC)
    @Query("select s from Stock s where s.id = :id")
    Stock findByIdWithOptimisticLock(Long id);
}
```

`@Version` 어노테이션이 붙은 컬럼을 추가해 Entity가 버전 관리를 할 수 있도록 한다.

- Stock

```java
@Entity
@NoArgsConstructor
public class Stock {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long productId;

    private Long quantity;

    @Version
    private Long version;

    public Stock(Long productId, Long quantity) {
        this.productId = productId;
        this.quantity = quantity;
    }

    public Long getQuantity() {
        return quantity;
    }

    public void decrease(Long quantity) {
        if (this.quantity - quantity < 0) {
            throw new RuntimeException("Low quantity!!");
        }
        this.quantity = this.quantity - quantity;
    }
}
```


- StockService

낙관적 락은 사용자가 트랜잭션 충돌시 콜백처리를 구현해주어야 한다.

```java
public void decrease(Long id, Long quantity) throws InterruptedException {
    /* 실패시 재시도 */
    while (true) {
        try {
            optimisticLockStockService.decrease(id, quantity);

            break;
        } catch (Exception e) {
            Thread.sleep(50);
        }
    }
}
```


- StockServiceTest

```java
@BeforeEach
public void before() {
    Stock stock = new Stock(1L, 100L);
    
    stockRepository.saveAndFlush(stock);
}

@Test
void 동시에_100번_요청_by_OptimisticLock() throws InterruptedException {

    // given
    int threadCount = 100;
    ExecutorService executorService = Executors.newFixedThreadPool(32);
    CountDownLatch countDownLatch = new CountDownLatch(threadCount);

    // when
    for (int i = 0; i < threadCount; i++) {
        executorService.submit(() -> {
            try {
                optimisticLockStockFacade.decrease(1L, 1L);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                countDownLatch.countDown();
            }
        });
    }

    countDownLatch.await();

    Stock stock = stockRepository.findById(1L).orElseThrow();

    // then
    assertThat(stock.getQuantity()).isEqualTo(0L);
}
```

테스트가 성공하는 것을 볼 수 있다.

![Database Lock]({{site.url}}/assets/image/2023/2023-03/26-con003.png)


QueryDsl의 경우 아래와 같이 `.setLockMode(LockModeType.OPTIMISTIC)` 의 메소드 체이닝을 추가해주면 된다.

```java
@Override
public Optional<Stock> findByIdWithPessimisticLockDsl(Long id) {

    return Stock findStock = queryFactory
            .selectFrom(stock)
            .where(stock.id.eq(id))
            .setLockMode(LockModeType.OPTIMISTIC)
            .fetch();
}
```