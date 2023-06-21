---
title:  "[Java] 분산 락(Distributed Lock)을 이용해 동시성 제어하기"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2023-06-19
last_modified_at: 2023-06-19
---

# 분산 락(Distributed Lock)을 이용해 동시성 제어하기


## 분산 락

기존 이전 포스트에서 설명한 [Datebase Lock](https://dh37789.github.io/java/Database-Lock/)과는 다른 방식으로 작동을 한다.

락을 거는 것은 같으나, 자원에 직접 락을 거는 것이 아닌 공통된 저장소에 락을 걸어 자원이 사용중인지를 체크하고, 사용 뮤무에 따라 프로세스를 진행하는 방식이다.


## Redis를 이용한 분산락 구현

서버들간 동기화된 처리가 필요하고, 여러 서버에서 공통된 저장소의 락을 바라봐야 하기 때문에 보통 분산락을 구현할때 redis를 이용해서 구현을 한다.

대표적인 Redis 라이브러리가 두가지가 있다.

Lettuce와 Redisson이 대표적인 예시인데, Lettuce는 간단하게 설명만 기술하고 Redission을 이용한 예시를 기술해보고자 한다.


## Lettuce

Lettuce는 스핀락을 기반으로 구현된 Redis 라이브러리이다.


### 스핀락?

스핀락이란? 앞서 분산 락에서 설명한 공통된 저장소를 스핀락의 말 그대로 빙글빙글 돌면서 계속 공통된 저장소의 락의 유무를 확인하는 기법이다.

![스핀락]({{site.url}}/public/image/2023/2023-06/19-lock001.png)

스레드나 프로세스가 단순히 루프에서 무한정 대기하면서 락의 획득 유무를 검사하다가, 기존의 락이 걸린 작업이 끝나고 뒤 이어 락을 얻으면 그때 대기했던 프로세스를 진행하는 것을 말한다.

무한정 대기하다 보니, 성능상에서는 크게 좋지않다.


## Redisson

스핀락의 무한 대기에 및 순회하는 동안 계속 Redis에 요청이 가게 되니 부하가 가게 될 것이라, Redisson에서는 다른 방법을 도입했다.

Pub / Sub (발행 / 구독) 모델을 도입해 락을 관리하도록 하였다.

![Pub/Sub]({{site.url}}/public/image/2023/2023-06/19-lock003.png)

계속해서 대기하는 것이 아니라, redis의 pub/sub을 이용해 이벤트가 왔을때 락의 획득 시도를 수행하도록 작성되었다.

### 예제

#### 의존성 추가

redisson은 레디스를 사용하기 때문에 로컬환경에 redis가 설치 되어있어야 한다.

간단한 docker 환경에서 구현해보도록 하겠다.

```shell
docker pull redis
docker run --name redis -d -p 6379:6379 redis
```

이제 Spring boot에서도 redis를 사용할 수 있도록 의존성을 추가해 줘야한다.

redisson을 사용할 수 있도록 gradle에 추가해준다.

```groovy
dependencies {
    // redis
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    // redisson
    implementation 'org.redisson:redisson-spring-boot-starter:3.17.4'
}
```

동시성을 테스트할 Entity 다음과 같다.

#### Entity

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

#### RedissonLockStockFacade

```java
@Component
public class RedissonLockStockFacade {

    private final RedissonClient redissonClient;

    private final StockService stockService;

    public RedissonLockStockFacade(RedissonClient redissonClient, StockService stockService) {
        this.redissonClient = redissonClient;
        this.stockService = stockService;
    }

    public void decrease(Long key, Long quantity) {
        RLock lock = redissonClient.getLock(key.toString());

        try {
            boolean available = lock.tryLock(5, 1, TimeUnit.SECONDS);

            if (!available) {
                System.out.println("lock 획득 실패");
                return;
            }

            stockService.decrease(key, quantity);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            lock.unlock();
        }
    }
}
```

> Facade 패턴 : 하위 시스템을 보다 쉽게 사용할 수 있게 해주는 고급 인터페이스이다.<br>
> wiki : [Facade 패턴 Wiki](https://ko.wikipedia.org/wiki/%ED%8D%BC%EC%82%AC%EB%93%9C_%ED%8C%A8%ED%84%B4)

Redisson을 사용하기 위해 RedissonClient의 의존성을 추가하고 다음과 같은 로직을 추가한다.

`RLock getLock(String name)`을 통해 매개변수로 전달받은 key값의 락을 흭득하며, <br>
`boolean tryLock(long waitTime, long leaseTime, TimeUnit unit)`을 통해 Lock의 획득을 시도한다.

- long waitTime: 락 획득을 기다리는시간
- long leaseTime: RLock획득 후 락이 만료되는 시간을 말한다.
- TimeUnit unit: 앞의 적시한 시간의 단위를 말한다.

[RLock](https://www.javadoc.io/doc/org.redisson/redisson/2.8.2/org/redisson/api/RLock.html)이란 Redisson에서 정의한 Lock 객체를 말한다.


이제 테스트를 진행해보자.

```java
@Test
void 동시에_100번_요청_by_RedissonLock() throws InterruptedException {

    // given
    int threadCount = 100;
    ExecutorService executorService = Executors.newFixedThreadPool(32);
    CountDownLatch countDownLatch = new CountDownLatch(threadCount);

    // when
    for (int i = 0; i < threadCount; i++) {
        executorService.submit(() -> {
            try {
                redissonLockStockFacade.decrease(1L, 1L);
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

테스트는 ExcutorService를 호출하여 32개의 쓰레드풀을 생성하여 병렬처리로 진행하였다.

재고의 수를 100개를 만들고 감소 로직을 100번 병렬처리로 돌려 정상적으로 감소가 되는지 확인하였다.

![테스트]({{site.url}}/public/image/2023/2023-06/19-lock002.png)

테스트가 성공하는 걸 볼 수 있었다.

참고 : [whats-a-distributed-lock-and-why-use-it - stackoverflow](https://stackoverflow.com/questions/11999324/whats-a-distributed-lock-and-why-use-it)
