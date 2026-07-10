---
title:  "[Java] 비동기 처리 중 다른 회원 데이터가 섞인 이유: AttributeConverter와 Race Condition"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2026-07-09
last_modified_at: 2026-07-09
---

미납 결제건을 재결제하는 배치에서 이해하기 어려운 문제가 발생했다.

배치는 30일 이상 미납된 결제건을 조회한 뒤, 각 결제건을 비동기로 병렬 처리하고 있었다. 그런데 배치 실행 후 일부 결제 데이터의 구매자 이메일과 전화번호가 **다른 회원의 값으로 UPDATE**되는 현상이 확인되었다.

처음에는 DTO 생성을 위해 엔티티의 getter를 호출했을 뿐인데 DB가 바뀐 것처럼 보였다. 하지만 실제 원인은 getter가 아니었다.

처음에는 단순히 비동기 처리나 트랜잭션 경계 문제처럼 보였다. 하지만 원인을 따라가다 보니 JPA `AttributeConverter`, 암복호화 Value Object, Hibernate flush 과정이 함께 얽혀 있었다.

이 글에서는 문제가 발생한 배경부터 시작해, 어떤 코드가 잘못된 값을 만들었고 왜 그 값이 DB UPDATE까지 이어졌는지 순서대로 정리한다.


## 문제 배경

### 기능 설명

미납 결제 재결제 배치는 대략 다음 흐름으로 동작했다.

1. 30일 이상 미납된 결제건을 조회한다.
2. 각 미납 결제건을 `CompletableFuture`와 `ExecutorService`를 이용해 비동기로 병렬 처리한다.
3. 결제 승인 요청 전, 외부 결제 API 호출에 필요한 DTO를 생성한다.
4. DTO 생성 과정에서 `Payment` 엔티티의 암호화된 필드, 예를 들어 전화번호와 이메일을 복호화해서 사용한다.
5. 결제 승인 요청 후 성공/실패 후처리를 수행한다.

구조를 단순화하면 다음과 같다.

```text
order-service → billing-service → PG사 결제 승인 API
```

### 발생한 문제

미납 결제 재결제 배치 실행 후, 특정 결제 데이터의 구매자 이메일과 전화번호가 다른 회원의 데이터로 변경되는 현상이 발생했다.

```text
A회원의 Payment row
    BUYER_PHONE = B회원 전화번호로 변경
    BUYER_EMAIL = C회원 이메일로 변경
```

단순 예외가 아니라 **개인정보성 데이터가 다른 회원의 값으로 잘못 저장된 문제**였기 때문에 원인 파악이 필요했다.

---

## 1. 비동기 처리 로직

문제가 발생한 배치는 미납 결제 목록을 조회한 뒤 각 미납건을 병렬로 처리하고 있었다.

```java
@Transactional
public RepayUnpaidOrderDto.BatchResultDto repayUnpaid() {
    // 미납 대상 기준: 30일 전 ~ 당일 8시 전까지의 미납 결제
    List<UnpaidAutoPayInfo> unpaidOrderList = unpaidAutoPayInfoRepository.selectUnpaidListBeforeMonth(
            Yn.N,
            LocalDate.now().atTime(8, 0, 0),
            LocalDate.now().minusDays(30).atStartOfDay()
    );

    CompletableFuture.allOf(
            StreamHelper.asStream(unpaidOrderList)
                    .map(unpaidOrder -> CompletableFuture.runAsync(
                            () -> repayPayment(unpaidOrder, successCnt, failCnt),
                            executorService
                    ))
                    .toArray(CompletableFuture[]::new)
    ).join();

    return RepayUnpaidOrderDto.BatchResultDto.of(successCnt.get(), failCnt.get());
}
```

여기서 중요한 점은 `repayPayment()`가 여러 스레드에서 동시에 실행된다는 것이다.

각 비동기 작업은 결제 승인을 위해 `Payment` 엔티티를 조회하고, 엔티티 로딩 시점에 JPA `AttributeConverter`가 호출된다.

> 핵심 포인트: 여러 스레드가 동시에 `Payment` 엔티티를 로딩하면서 같은 `AttributeConverter` 인스턴스를 사용할 수 있다.

---

## 2. DTO 생성 과정

결제 승인 요청을 보내기 전, `Payment` 엔티티에서 구매자 전화번호와 이메일을 읽어 외부 결제 API 요청 DTO를 만든다.

```java
public static ApprovalData makeData(Payment payment, Order order, UnpaidAutoPayInfo unpaidOrder) {
    return ApprovalData.builder()
            // ...
            .buyerTel(ObjectUtils.isEmpty(payment.getBuyerPhone().getPlain())
                    ? "-"
                    : payment.getBuyerPhone().getPlain())
            .buyerEmail(ObjectUtils.isEmpty(payment.getBuyerEmail().getPlain())
                    ? ""
                    : payment.getBuyerEmail().getPlain())
            .build();
}
```

`getBuyerPhone()`은 `CellPhone` 타입을, `getBuyerEmail()`은 `Email` 타입을 반환한다. 이 타입들은 DB에는 암호화된 문자열로 저장되어 있고, 엔티티에서는 복호화된 값을 가진 객체로 사용된다.

변환은 JPA `AttributeConverter`를 통해 자동으로 수행된다.

처음에는 getter 호출이 UPDATE를 유발한 것처럼 보였다. 하지만 getter 자체가 DB UPDATE를 발생시킨 것은 아니다.

정확한 흐름은 다음에 가깝다.

```text
Payment 엔티티 로딩
    → AttributeConverter 호출
    → Race Condition으로 필드에 잘못된 값이 들어감
    → 잘못된 값을 가진 엔티티가 영속성 컨텍스트에 올라감
    → flush/commit 시점에 변경 감지
    → 잘못 들어간 값이 DB에 UPDATE
```

즉, 문제는 getter가 아니라 **엔티티 로딩 시점의 변환 과정**에서 시작되었다.

---

## 3. 문제가 된 CellPhone 구현

문제가 된 클래스는 다음과 같은 구조였다.

```java
@Setter
@Getter
@Converter(autoApply = true)
public class CellPhone extends AbstractConverter implements AttributeConverter<CellPhone, String> {

    private String plain;
    private String encrypt;
    private CryptoHelper.Key key;

    @Builder
    public CellPhone(String plain, String encrypt) {
        this.plain = plain;
        this.encrypt = encrypt;
        this.key = CryptoHelper.Key.HPTEL;
    }

    @Override
    public String convertToDatabaseColumn(CellPhone cellPhone) {
        return Optional.ofNullable(cellPhone)
                .map(AbstractConverter::encrypt)
                .orElse(null);
    }

    @Override
    public CellPhone convertToEntityAttribute(String encrypt) {
        this.encrypt = encrypt; // 문제의 핵심
        return CellPhone.by(decrypt(), encrypt);
    }
}
```

가장 큰 문제는 `convertToEntityAttribute()` 안에서 `this.encrypt`를 변경하고 있다는 점이다.

```java
this.encrypt = encrypt;
```

`AttributeConverter`는 상태를 가진 도메인 객체가 아니라 변환기다. 변환기는 입력값을 받아 출력값을 반환해야 하며, 내부 인스턴스 필드를 변경하면 안 된다.

하지만 위 구조에서는 `CellPhone`이 도메인 값 객체이면서 동시에 JPA Converter 역할을 수행하고 있었다.

실제로 여러번 테스트를 돌려본 결과 데이터가 점점 잘못된 데이터로 Update하는 걸 볼 수 있었다.

![데이터오염1]({{site.url}}/public/image/2026/07/09-attribute_1.png)

![데이터오염2]({{site.url}}/public/image/2026/07/09-attribute_2.png)

---

## 4. 왜 이 구조가 위험한가

### Value Object와 Converter의 책임이 섞여 있음

기존 `CellPhone`은 두 책임을 동시에 가지고 있었다.

| 책임 | 기존 구조에서의 문제 |
|---|---|
| Value Object | 전화번호의 평문/암호문 값을 보관함 |
| AttributeConverter | DB 컬럼과 엔티티 필드를 변환함 |

이 두 역할이 같은 클래스에 있으면 Converter로 사용되는 인스턴스와 실제 엔티티 필드 값으로 사용되는 인스턴스의 경계가 흐려진다.

특히 Converter 인스턴스가 재사용될 때, 내부 상태가 여러 변환 작업 사이에 공유될 수 있다.

### AttributeConverter는 재사용된다

Hibernate는 `AttributeConverter` 인스턴스를 변환 작업마다 새로 생성하지 않는다. 일반적으로 Converter 인스턴스는 재사용된다.

따라서 아래와 같은 코드는 위험하다.

```java
@Override
public CellPhone convertToEntityAttribute(String encrypted) {
    this.encrypt = encrypted;
    return CellPhone.by(decrypt(), encrypted);
}
```

이 메서드가 동시에 여러 스레드에서 호출되면 `this.encrypt`는 공유 mutable state가 된다.

---

## 5. Race Condition 발생 메커니즘

두 스레드가 동시에 서로 다른 회원의 결제 정보를 로딩한다고 가정해보자.

```text
시간축 →

Thread A: this.encrypt = "USER_A_암호화값"

                                  Thread B: this.encrypt = "USER_B_암호화값"
                                            // A가 넣어둔 값을 덮어씀

Thread A: decrypt()
          // Thread A는 USER_A 값을 복호화한다고 생각하지만,
          // 실제 this.encrypt는 USER_B_암호화값일 수 있음

Thread A: return CellPhone.by("USER_B_평문", "USER_A_암호화값")
```

결과적으로 Thread A가 만든 `CellPhone` 객체는 다음처럼 불일치 상태가 된다.

```text
plain   = USER_B의 전화번호
encrypt = USER_A의 암호화값
```

즉, A회원의 엔티티 필드에 B회원의 평문 전화번호가 들어간다.

---

## 6. 왜 DB UPDATE까지 발생했나

이 문제에서 헷갈리기 쉬운 부분은 “getter만 호출했는데 왜 DB가 UPDATE 되었는가?”이다.

정확히는 getter가 UPDATE를 만든 것이 아니다.

엔티티 로딩 시점에 이미 `AttributeConverter`의 race condition으로 필드 값이 잘못 들어감되었고, 이후 트랜잭션 flush 시점에 Hibernate가 해당 필드를 변경된 값으로 판단하면서 UPDATE가 발생했다.

### 6.1 엔티티 로딩 시점

```text
DB 컬럼값: "USER_A_암호화값"
    ↓ convertToEntityAttribute() 호출
    ↓ race condition 발생
엔티티 필드: CellPhone(plain="USER_B_평문", encrypt="USER_A_암호화값")
```

이 시점에 엔티티 필드는 이미 잘못된 값을 가지고 있다.

단순히 “스냅샷과 현재 값이 다르므로 UPDATE된다”라고만 설명하면 부족하다. 엔티티 스냅샷도 로딩 시점에 생성되므로, 스냅샷 자체가 이미 잘못된 값을 가진 객체를 기준으로 만들어질 수 있기 때문이다.

핵심은 다음이다.

> 잘못된 값을 가진 Java 객체가 영속성 컨텍스트에 올라갔고, flush 시점에 DB 컬럼 값으로 다시 변환되면서 원래 DB 값과 다른 값이 만들어질 수 있다.

### 6.2 flush 시점

`convertToDatabaseColumn()`은 `CellPhone` 객체를 다시 DB 컬럼 값으로 변환한다.

```java
@Override
public String convertToDatabaseColumn(CellPhone cellPhone) {
    return Optional.ofNullable(cellPhone)
            .map(AbstractConverter::encrypt)
            .orElse(null);
}
```

이때 `encrypt()`가 `plain` 값을 기반으로 암호문을 생성한다면, 잘못 들어간 `plain` 값이 그대로 DB 저장값으로 변환된다.

```text
현재 엔티티값: CellPhone(plain="USER_B_평문", encrypt="USER_A_암호화값")
    ↓ convertToDatabaseColumn()
DB 저장 후보값: "USER_B_암호화값"
```

결과적으로 원래 DB 값과 다른 값이 만들어진다.

```text
원래 DB 값: "USER_A_암호화값"
저장 후보값: "USER_B_암호화값"
```

Hibernate는 flush 과정에서 해당 필드를 변경된 값으로 판단하고 UPDATE를 실행할 수 있다.

> 참고: Hibernate의 정확한 dirty checking 방식은 버전, 타입 매핑, mutability plan, `equals/hashCode` 구현 여부에 따라 달라질 수 있다. 이 글의 핵심은 특정 내부 구현을 단정하는 것이 아니라, 잘못된 값을 가진 엔티티가 flush 시점에 DB 컬럼 값으로 변환되며 UPDATE로 이어졌다는 점이다.

### 6.3 UPDATE 결과

최종적으로는 다음과 같은 UPDATE가 발생할 수 있다.

```sql
UPDATE PAYMENT
SET BUYER_PHONE = 'USER_B_암호화값',
    BUYER_EMAIL = 'USER_C_암호화값'
WHERE PAYMENT_ID = 'USER_A의_PAYMENT_ID';
```

---

## 7. 전체 흐름 정리

```text
미납 결제 재결제 배치 실행
    ↓
미납 결제 목록 조회
    ↓
CompletableFuture.runAsync()로 병렬 처리
    ↓
각 스레드에서 Payment 엔티티 로딩
    ↓
CellPhone/Email AttributeConverter 호출
    ↓
공유 Converter 인스턴스의 this.encrypt 동시 변경
    ↓
다른 회원의 평문 값이 엔티티 필드에 들어감
    ↓
결제 승인 요청 DTO 생성
    ↓
트랜잭션 flush/commit
    ↓
잘못 들어간 값이 다시 암호화되어 UPDATE
```

그림으로 보면 다음과 같다.

```text
┌──────────────────────────────────────────────────────────────┐
│ 미납 결제 재결제 배치                                      │
│                                                              │
│  unpaidOrderList 조회                                       │
│       ↓                                                      │
│  CompletableFuture.allOf(...).join()                         │
│       ↓                                                      │
│  ┌──────────────────┐        ┌──────────────────┐           │
│  │ Thread A          │        │ Thread B          │           │
│  │                  │        │                  │           │
│  │ Payment 로딩      │        │ Payment 로딩      │           │
│  │ Converter 호출    │        │ Converter 호출    │           │
│  │ this.encrypt=A    │        │ this.encrypt=B    │           │
│  │ decrypt() → B평문 │        │ decrypt() → ?     │           │
│  │ Entity에 잘못된 값 설정       │        │                  │           │
│  └──────────────────┘        └──────────────────┘           │
│       ↓                                                      │
│  flush/commit                                                │
│       ↓                                                      │
│  Dirty Checking / DB UPDATE                                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. 비동기 스레드와 `@Transactional`

여기서 하나 더 주의할 점은 Spring 트랜잭션과 비동기 스레드의 관계다.

Spring의 트랜잭션은 기본적으로 `ThreadLocal` 기반으로 관리된다. 따라서 메인 스레드에서 시작된 트랜잭션은 `CompletableFuture.runAsync()`로 실행되는 작업 스레드에 자동으로 전파되지 않는다.

```java
@Transactional
public RepayUnpaidOrderDto.BatchResultDto repayUnpaid() {
    CompletableFuture.runAsync(() -> repayPayment(...), executorService);
}
```

위 코드에서 `repayUnpaid()`의 트랜잭션은 메인 스레드에 적용된다. 비동기 내부의 `repayPayment()`가 어떤 트랜잭션에서 실행되는지는 `repayPayment()`의 선언 위치, 프록시 호출 여부, 별도 `@Transactional` 적용 여부에 따라 달라진다.

즉, 다음 내용을 반드시 확인해야 한다.

- 비동기 작업 내부에서 새 트랜잭션이 시작되는가?
- `repayPayment()`가 Spring proxy를 통해 호출되는가?
- 엔티티가 어떤 영속성 컨텍스트에 등록되는가?
- flush/commit 시점이 어디인가?

다만 트랜잭션 전파 여부와 별개로, 이 문제의 직접 원인은 `AttributeConverter`가 공유 상태를 가진다는 점이다.

---

## 9. 근본 원인 요약

| 구분 | 내용 |
|---|---|
| 직접 원인 | 재사용되는 Converter 인스턴스의 mutable field에 대한 동시 쓰기 |
| 설계 결함 | Value Object와 AttributeConverter를 하나의 클래스로 합친 구조 |
| 증폭 요인 | `CompletableFuture + ExecutorService` 기반 비동기 병렬 처리 |
| DB 반영 | 잘못된 값을 가진 엔티티가 flush 시점에 DB UPDATE로 이어짐 |
| 잘못된 최초 가설 | getter 호출이 직접 UPDATE를 유발했다는 해석 |
| 실제 원인 | 엔티티 로딩 시점의 Converter race condition과 이후 flush |

---

## 10. 해결 방안

### 방안 1. Converter와 Value Object 분리 — 권장

가장 중요한 해결책은 Value Object와 Converter를 분리하는 것이다.

#### Value Object

`CellPhone`은 전화번호 값을 표현하는 객체로만 둔다. 가능하면 immutable하게 만든다.

```java
@Getter
public final class CellPhone {

    private final String plain;
    private final String encrypted;

    private CellPhone(String plain, String encrypted) {
        this.plain = plain;
        this.encrypted = encrypted;
    }

    public static CellPhone fromEncrypted(String encrypted, String plain) {
        return new CellPhone(plain, encrypted);
    }

    public static CellPhone fromPlain(String plain) {
        return new CellPhone(plain, null);
    }
}
```

#### AttributeConverter

Converter는 상태를 갖지 않는 순수 변환기로 만든다.

```java
@Converter(autoApply = true)
public class CellPhoneConverter implements AttributeConverter<CellPhone, String> {

    @Override
    public String convertToDatabaseColumn(CellPhone attribute) {
        if (attribute == null || attribute.getPlain() == null) {
            return null;
        }

        return CryptoHelper.encrypt(attribute.getPlain(), CryptoHelper.Key.HPTEL);
    }

    @Override
    public CellPhone convertToEntityAttribute(String dbData) {
        if (dbData == null) {
            return null;
        }

        String plain = CryptoHelper.decrypt(dbData, CryptoHelper.Key.HPTEL);
        return CellPhone.fromEncrypted(dbData, plain);
    }
}
```

핵심은 간단하다.

```text
Converter에 인스턴스 필드가 없다.
변환은 파라미터와 지역 변수만으로 끝난다.
```

이렇게 하면 여러 스레드가 동시에 Converter를 호출하더라도 공유 상태가 없으므로 race condition이 발생하지 않는다.

---

### 방안 2. 비동기 처리 제거 — 재현을 막는 회피책

비동기 병렬 처리를 제거하고 순차 처리로 바꾸면 해당 race condition은 재현되지 않을 가능성이 높다.

```java
unpaidOrderList.forEach(unpaidOrder -> repayPayment(unpaidOrder, successCnt, failCnt));
```

하지만 이 방법은 근본 해결책이 아니다.

상태를 가진 Converter라는 설계 결함은 그대로 남아 있으므로, 다른 병렬 조회 경로나 다른 배치에서 같은 문제가 다시 발생할 수 있다. 또한 순차 처리로 바꾸면 배치 처리 시간이 늘어날 수 있다.

따라서 비동기 제거는 긴급 회피책으로는 사용할 수 있지만, 최종 해결책은 Converter를 stateless하게 고치는 것이다.

---

### 방안 3. readOnly 트랜잭션 또는 DTO Projection — 피해 확산 방어책

결제 승인 요청 DTO를 만들기 위해 필요한 값만 읽는다면 엔티티를 영속 상태로 로딩하지 않는 방식도 고려할 수 있다.

예를 들어 조회 전용 트랜잭션을 사용할 수 있다.

```java
@Transactional(readOnly = true)
public Payment getPaymentForApproval(Long paymentId) {
    return paymentRepository.findById(paymentId).orElseThrow();
}
```

또는 엔티티 대신 DTO Projection으로 필요한 컬럼만 조회할 수 있다.

```java
public interface PaymentApprovalProjection {
    Long getPaymentId();
    String getBuyerPhone();
    String getBuyerEmail();
}
```

다만 이 방법도 근본 해결은 아니다.

`readOnly`나 DTO Projection은 의도치 않은 UPDATE를 막거나 줄이는 방어책일 수 있지만, Converter가 잘못된 값을 만드는 문제 자체를 해결하지는 않는다.

> 근본 해결은 Converter를 stateless하게 만드는 것이다.

---

## 11. 재발 방지 체크리스트

`AttributeConverter`를 작성할 때는 다음을 확인해야 한다.

- [ ] Converter에 인스턴스 필드가 있는가?
- [ ] `convertToEntityAttribute()` 또는 `convertToDatabaseColumn()`에서 `this.xxx`를 변경하는가?
- [ ] `static` mutable field를 사용하고 있는가?
- [ ] Value Object와 Converter 역할이 한 클래스에 섞여 있는가?
- [ ] Value Object가 불필요하게 mutable한가?
- [ ] 비동기 배치에서 싱글턴 객체나 공유 객체를 사용하는가?
- [ ] 조회만 필요한 로직에서 엔티티를 영속 상태로 로딩하고 있지는 않은가?
- [ ] 의도치 않은 UPDATE를 탐지할 수 있는 SQL 로그나 감사 로그가 있는가?

---

## 12. 결론

이번 문제는 단순한 비동기 처리 버그가 아니었다.

겉으로는 비동기 배치 실행 후 데이터가 잘못 UPDATE된 문제였지만, 실제 원인은 더 아래 계층에 있었다.

```text
비동기 병렬 처리
    + 재사용되는 AttributeConverter
    + 상태를 가진 Converter 구현
    + JPA dirty checking / flush
    = 잘못된 값이 DB에 UPDATE
```

직접적인 원인은 `AttributeConverter` 내부에서 인스턴스 필드를 변경한 것이었다.

```java
@Override
public CellPhone convertToEntityAttribute(String encrypted) {
    this.encrypt = encrypted;
    return CellPhone.by(decrypt(), encrypted);
}
```

Hibernate는 `AttributeConverter`를 변환할 때마다 새로 만들지 않고 재사용한다. 따라서 Converter는 여러 엔티티, 여러 스레드에서 동시에 호출될 수 있다고 보고 **상태를 갖지 않도록(stateless)** 작성해야 한다.

하지만 기존 `CellPhone` 클래스는 Value Object와 AttributeConverter 역할을 동시에 가지고 있었다.

| 역할 | 설명 |
|---|---|
| Value Object | `plain`, `encrypt` 값을 가진 데이터 객체 |
| AttributeConverter | DB 컬럼과 엔티티 필드 사이를 변환하는 인프라 코드 |

이 구조 때문에 비동기 스레드 A가 `this.encrypt`에 A회원의 암호화 값을 넣은 직후, 스레드 B가 같은 필드에 B회원의 암호화 값을 덮어쓸 수 있었다.

결과적으로 A회원의 엔티티에 B회원의 평문 전화번호/이메일이 들어갔고, 이후 트랜잭션 flush 시점에 해당 값이 다시 암호화되어 DB에 UPDATE되었다.

이번 사례에서 얻은 교훈은 다음과 같다.

1. `AttributeConverter`는 반드시 stateless하게 작성해야 한다.
2. Value Object와 Converter는 분리해야 한다.
3. 비동기 처리에서는 공유 mutable state를 특히 조심해야 한다.
4. getter 호출이 문제처럼 보여도, 실제 원인은 엔티티 로딩/flush 과정에 있을 수 있다.
5. 조회만 필요한 로직에서는 readOnly, DTO Projection, detach 등을 통해 의도치 않은 UPDATE를 방어해야 한다.

가장 중요한 원칙은 이것이다.

> 인프라 계층의 작은 상태 공유 하나가 비동기 환경에서는 실제 데이터가 잘못 저장되는 문제로 이어질 수 있다.

---

## 마무리

`AttributeConverter`는 편리하지만, JPA/Hibernate 내부에서 재사용되는 인프라 컴포넌트다. 따라서 일반 도메인 객체처럼 상태를 들고 있으면 안 된다.

특히 암복호화처럼 민감한 데이터를 다루는 Converter라면 더더욱 상태를 갖지 않도록 설계해야 한다.

이 문제를 계기로 Converter는 단순한 매핑 코드가 아니라, 멀티스레드 환경에서도 안전해야 하는 인프라 코드라는 점을 다시 확인할 수 있었다.
