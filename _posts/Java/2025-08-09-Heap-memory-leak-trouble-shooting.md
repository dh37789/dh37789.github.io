---
title:  "주문 서버 Heap Memory Leak 분석 및 조치 정리"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2025-08-09
last_modified_at: 2025-08-09
---


시기가 좀 지났던 Trouble shooting 이지만, 기존에 단순히 끄적여놨던 원인 분석을 정리해서 올려보고자 한다.


## 1. 사건 개요

2024년 5월경, 주문 서버의 Heap 메모리 부족으로 인해 health-check API (SELECT 1)에 지연 알림이 발생했습니다.

### 발생 상황:

```text
name: 주문 API
address: http://{{Order}}/actuator/health
status: 200
latency: 3.02s
msg: 응답지연
```

- 서버 노드: 일부 인스턴스에서 *Connection Refused*, 일부에서는 *응답 지연 (최대 3초 이상)*

- 해당 알림이 여러 차례 반복 발생하며 실제 서비스 응답에도 영향을 주었습니다.


## 2. 메모리 사용 이력 확인 (kibana)

### Non-Heap Memory

![non_heap]({{site.url}}/public/image/2025/08/09_heap002.jpg)

- 5월 22일 이전까지 지속적으로 증가.

- 5월 22일 주문 서버를 전체적으로 재기동 후 감소.

- 이후 다시 증가하는 추세 확인.


### Heap Memory

![heap]({{site.url}}/public/image/2025/08/09_heap001.jpg)

- 위의 Non-Heap Memory와 동일하게 지속적으로 증가하다 22일 재기동 후 감소 추세

- 재기동 전 GC가 정상적으로 동작하지 못하며 메모리가 누적되는것으로 추축


### 일시적 대응:

주문 서버의 전체 인스턴스를 재기동하여 메모리 초기화 및 GC 복구를 진행 하였다. 하지만 근본적인 원인을 찾아 해결 하기 위해, 개발서버에서 현상을 재현하고 메모리의 덤프를 생성하여 분석을 진행하였다.


## 3. Heap 메모리 분석 (JMAP)

- Jmap은 Java Memory Map의 약자로 JVM의 메모리 상태를 확인하고 덤프를 생성할 수 있게 도와주는 도구 이다.
- 우선 덤프를 생성하기 이전 메모리의 현황을 조회해 보았다.

### 명령어 수행 예:
```shell
sudo jps sudo jmap -heap <PID>
```

### Heap 상태 요약:

```shell
Attaching to process ID 0000000, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.161-b12
using thread-local object allocation.
Parallel GC with 8 thread(s)
Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 2013265920 (1920.0MB)
   NewSize                  = 41943040 (40.0MB)
   MaxNewSize               = 671088640 (640.0MB)
   OldSize                  = 83886080 (80.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)
Heap Usage:
PS Young Generation
Eden Space:
   capacity = 651165696 (621.0MB)
   used     = 127736792 (121.81929779052734MB)
   free     = 523428904 (499.18070220947266MB)
   19.616634104754805% used
From Space:
   capacity = 7340032 (7.0MB)
   used     = 7102024 (6.773017883300781MB)
   free     = 238008 (0.22698211669921875MB)
   96.7573983328683% used
To Space:
   capacity = 9437184 (9.0MB)
   used     = 0 (0.0MB)
   free     = 9437184 (9.0MB)
   0.0% used
PS Old Generation
   capacity = 1342177280 (1280.0MB)
   used     = 1295780032 (1235.7521362304688MB)
   free     = 46397248 (44.24786376953125MB)
   96.54313564300537% used
```

jamp의 로그를 분석해 본다면, 아래의 내용과 같다.

### Eden / Survivor (Young)

- `Eden`영역은 아직 여유가 있다 (20%정도 사용)
- `Survivor From`영역이 97%로 꽉차있고, `To` 영역은 비어 있음 → 최근 `Minor GC` 이후 "From ↔ To" 스왑 직전 상태로 추정

### Old (Tenured)

- 사용율이 96.5%로 사실상 포화 상태이다.
- 다음 `Minor GC` 에서 `Eden/From`의 살아있는 객체를 `To` 로 복사하고, 승격이 필요하면 `Old`영역으로 올리려 할테지만 `Old`의 여유가 거의 없어 승격 실패 위험이 크다.


## 4. Heap 덤프 분석 (MAT)

Jmap 로그로써는 한계가 있어, 메모리 덤프를 생성하여, 분석을 해보았다.

MAT(Eclipse Memory Analyzer Tool)은 Java Heap Dump 파일(.hprof)을 열어 메모리 누수(Leak) 분석, 대용량 객체 탐지, 메모리 사용 구조를 파악 할 수 있는 툴이다.

### 덤프 파일 생성:
```shell
sudo jmap -dump:format=b,file=heapdump.hprof <PID>
```

### Heap Memory Diagram 분석

![heap]({{site.url}}/public/image/2025/08/09_heap003.jpg)

**Heap-Memory의 점유 상위 객체**
- `org.springframework.boot.loader.LaunchedURLClassLoader`
- Retained Size: 377.5MB (Heap 전체의 약 46.5%)
- Shallow Size: 80 B
- 직접적으로 많은 데이터를 갖고 있지는 않지만, 연결된 객체 그래프 전체를 붙잡고 있어 메모리를 대량으로 점유하고 있다.

![heap]({{site.url}}/public/image/2025/08/09_heap004.jpg)

- `LaunchedURLClassLoader`는 Spring Boot에서 JAR 실행시 사용되는 커스텀 클래스 로더이다.
- 정상적으로 앱 종료 또는 리디플로우시 GC로 회수되어야 하지만, 클래스 로더 인스턴스가 여전히 다른 객체나 정적 필드에서 참조하고 있다. → GC가 해제 불가
- 특히 `java.lang.Object[]` 대형 배열이 붙잡혀 있어 힙 메모리를 대량 점유

**정리**

- `Retained Size`가 크다는 건, 그 객체가 GC에서 제거되지 않는한 참조 그래프 전체가 해제되지 않는다는 뜻이다.
- `LaunchedURLClassLoader`가 위와 같이 큰 메모리를 점유하고 있다면, 해당 클래스 로더가 로딩한 클래스, 정적필드, ThreadLocal 등이 여전히 살아있다는 뜻이다.
- 이런 상태가 지속되면, Old Gen이 꽉차서 Full GC가 반복되다 OOM(Out of Memory)가 발생 할 수 있다.

**결론**

- 전형적인 **ClassLoader Memory Leak 패턴**이며, 특정 클래스가 해제되지 못하고 누수가 발생하는것으로 결론을 지었다.


### 의심 객체 분석

![heap]({{site.url}}/public/image/2025/08/09_heap005.jpg)

- `LaunchedURLClassLoader` 클래스 내부의 `Object[]` 배열이 327,680개의 객체를 참조하고 있고, 393MB를 보유 하고 있다.


![heap]({{site.url}}/public/image/2025/08/09_heap006.png)

- PayConditionUsingStatisticRedis_Accessor_* 관련 객체들
- 약 26만 개 생성됨
- 이런 런타임 생성 클래스는 보통 캐시에 들어가면 클래스 로더가 살아있는 동안 해제되지 않음


**정리**

- Vector → Object[] → 다양한 클래스 인스턴스를 참조하고 있어 **GC가 불가능**하다.
- `PayConditionUsingStatisticRedis_Accessor_*` 관련 객체들이 26만개 정도 생성 되어있고, GC가 이 클래스를 해제하지 않고 있다.


**결론**

- `PayConditionUsingStatisticRedis` 객체 관련 로직에서 `ClassLoader Leak`이 발생하고 있다!


## 5. Heap 분석 결론

### 직접 원인

- 위의 Jmap의 로그와 MAT의 덤프 분석을 통해 특정 클래스의 로직에서 `ClassLoader Leak`이 발생한다는 결론을 알 수 있었다.
- PayConditionUsingStatisticRedis 객체가 GC에 의해 해제되지 않고 누적되며 LaunchedURLClassLoader의 메모리를 비정상적으로 점유한다.
- 해당 객체들은 단일 Vector 내부에 집합 형태로 저장되어 있으며, 이 Vector가 클래스 로더의 수명과 같이 가면서 회수되지 않고 있다.


### 근본 원인

**ClassLoader Leak 패턴**
- Spring Boot의 LaunchedURLClassLoader는 애플리케이션 클래스 로딩 시 메모리를 할당함
- 이 클래스 로더를 통해 로드된 일부 클래스가 Vector를 통해 강한 참조를 유지하며 GC 루트에 남게 됨
- 특히 Redis 캐시 관련 도메인 객체(예: PayConditionUsingStatisticRedis)가 캐시된 채로 참조되고 있어 GC 대상에서 제외됨


### GC가 해당 객체를 정리하지 못한 이유

- GC는 객체 간 순환 참조가 있더라도 GC 루트(Root Set)로부터의 참조 체인을 끊을 수 있어야 회수가 가능함
- 본 이슈에서는 다음과 같은 경로로 참조 체인이 유지됨:
  > LaunchedURLClassLoader → Vector → Object[] → PayConditionUsingStatisticRedis
- LaunchedURLClassLoader 자체가 시스템에서 GC 루트로 간주되는 구조 내에 존재하고 있으며,
- 해당 ClassLoader가 언로드되지 않는 한 그 하위 객체도 정리되지 않음
- 이는 주로 ClassLoader에 의해 로드된 클래스 내부에 정적(static) 필드 또는 Singleton 객체가 외부 객체를 참조하고 있는 경우 발생
- 결과적으로, PayConditionUsingStatisticRedis 객체는 메모리에서 GC가 불가능한 상태로 계속 남게 됨



### GC Root 체계 시각화
```text
[JVM Root Set]
  └─ System ClassLoader
    └─ LaunchedURLClassLoader
      └─ static Vector
        └─ Object[]
          └─ PayConditionUsingStatisticRedis 인스턴스 (26만 개)
```
- 위 구조에서 Root Set에서 시작된 참조 체인이 끊기지 않기 때문에 GC 대상이 되지 않음


## 6. 직접 적인 원인 분석

### 관련 런타임 에러 로그

최근 간헐적으로 발생한 에러중에 분석한 클래스와 관련된 에러로그가 올라왔었다.

해당 로직을 좀더 자세히 살펴보도록 했다.

```shell
[2024-06-11 15:41:10.703] [ERROR] - Unexpected exception occurred invoking async method: public void
...PayConditionUsingStatisticService.savePayConditionUsingStatistic(...)

java.lang.RuntimeException: java.lang.IllegalStateException:  org.springframework.cglib.core.CodeGenerationException:  java.lang.LinkageError --> loader (instance of LaunchedURLClassLoader):
attempted duplicate class definition for name: "kr/test/common/domain/cache/PayConditionUsingStatisticRedis_Accessor_au369p"
```

**분석**

**1. CGLIB 프록시 생성 과정 에서 충돌**
   - **비동기 메서드 실행 중 예외 발생**
   - Spring이 `@Async`, `@Transactional`, `@Cacheable` 어노테이션이 붙은 메서드에 대해 CGLIB 기반의 프록시 클래스를 생성한다.
   - `PayConditionUsingStatisticRedis` 클래스에 대해 여러 번 프록시를 만들면서 클래스 이름 충돌이 발생했을 가능성이 높다.

**2. ClassLoader 문제**
   - Spring Boot의 LaunchedURLClassLoader가 클래스 이름 중복을 탐지하고 **LinkageError를 발생**시킴

### 비동기 처리와 누수 연관성

아래는 위의 런타임 에러가 발생한 비동기 메서드 코드이다.
이 메서드는 Redis 통계 데이터를 저장하는 기능을 수행하며, @Async 어노테이션을 통해 별도의 쓰레드에서 동작하는 로직이다.

```java
@Async
@Async
public void saveStatistic(CacheService.CacheType cacheType) {
    RLock lock = redisson.getLock("Lock");
    try {
        //레디스 락
        boolean isLock = lock.tryLock(1, 1, TimeUnit.MINUTES);
        if(!isLock) {
            throw new BaseException("결제 상태값 통계 redis lock 획득 실패");
        }
        ...
        ObjectHashMapper objectHashMapper = new ObjectHashMapper();
        Map<byte[], byte[]> map = objectHashMapper.toHash(statistic);
        redisTemplate.execute((RedisCallback<?>) connection -> {
            connection.multi();
            connection.hMSet(("PayConditionStatistic:" + LocalDate.now()).getBytes(), map);
            connection.exec();
            return null;
        });
    } catch (Exception e) {
        log.error(e);
    } finally {
        lock.unlock();
    }
}
```

위에 발생했던 런타임 에러와 함께 확인해본다면 아래와 같이 정리가 가능했다.

- 런타임 에러의 의미는 **CG LIB 동적 클래스 생성**이 반복되면서 동일한 이름의 Proxy 클래스가 중복 정의되었다는 의미이다.
- 에러의 원인으로는 Spring @Async 기반의 비동기 처리 도중 Proxy 클래스가 GC되지 않고 누적되었을 가능성을 주고있다.


#### 종합적 원인 정리

- 비동기 로직 내에서 **매 요청마다 새로운 CGLIB Proxy 클래스**가 생성되며,
- 이 클래스들이 `LaunchedURLClassLoader`를 통해 로드됨 → 해제 불가
- Proxy 인스턴스가 static 필드에 보관되거나, ClassLoader에 종속된 구조일 경우 GC 대상에서 제외됨
- 결과적으로 ClassLoader는 언로드되지 않고 계속해서 메모리를 증가시키는 형태로 누수 발생


### 문제 해결을 위한 코드 리팩토링

중복된 CGLIB 프록시 클래스 정의 및 GC 누수 문제를 방지하기 위해, 기존 통계 객체의 저장 로직을 다음과 같이 별도의 static 유틸 메서드로 분리하였습니다.

이로 인해 객체 직렬화 및 Redis 저장 처리 시 ClassLoader 참조가 축소되고, 재사용이 가능해져 메모리 안정성을 확보할 수 있었습니다.

```java
public static void saveObject(String key, Object obj, RedisTemplate redisTemplate, Long expireDay) {
    Map<byte[], byte[]> map = toHash(obj);

    redisTemplate.execute((RedisCallback<?>) connection -> {
        connection.hMSet(key.getBytes(), map);
        if(expireDay != null) {
            connection.expire(key.getBytes(), 3600 * 24 * expireDay);
        }
        return null;
    });
}
```

**1. 불필요한 CGLIB 프록시 생성 방지**

- 문제 상황에서는 비동기 경로에서 CGLIB가 생성한 접근자/프록시 클래스가 중복 정의되어 `LinkageError(attempted duplicate class definition)`가 발생했다.
- 저장 로직을 정적(static) 메서드로 이동하면 이 구간은 스프링 프록시가 관여할 포인트컷에서 벗어나며, 참조 그래프가 단순해서 `ClassLoader`가 잡는 점유를 줄어들게 된다.
- GC 루트 참조 그래프가 단순해져 Old Gen 회수가 원활해지고, Retained Heap이 비정상적으로 커지는 현상이 해소된다.

**2. 직렬화 및 변환도 단순화**

- 기존에는 고수준 직렬화기(Jackson Serializer 등)를 거치며 리플렉션 기반 Accessor·프록시 클래스가 반복 생성했다.
- ObjectHashMapper로 Map<byte[], byte[]>를 직접 구성해 넣으면 이 변환 계층을 우회해, CGLIB 클래스 중복 로딩과 이름 충돌(LinkageError) 가능성이 크게 줄어든다.


