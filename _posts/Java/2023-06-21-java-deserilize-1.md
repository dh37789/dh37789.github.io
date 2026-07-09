---
title:  "[Java] Jackson 역직렬화 이해하기 1: ObjectMapper 기본 사용법"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2023-06-19
last_modified_at: 2026-07-09
---

Java에서 객체를 외부 시스템과 주고받을 때 가장 자주 사용하는 형식 중 하나가 JSON이다.

Spring Boot로 API를 만들다 보면 요청 JSON은 DTO로 들어오고, 응답 DTO는 다시 JSON으로 내려간다. 이 과정은 대부분 자동으로 처리되기 때문에 평소에는 크게 신경 쓰지 않는다.

하지만 실제로는 내부에서 Java 객체와 JSON 사이의 변환이 계속 일어나고 있다. 그리고 Spring 진영에서는 이 변환을 위해 주로 Jackson을 사용한다.

이번 글에서는 Java native serialization이 아니라, **Jackson `ObjectMapper`를 이용한 JSON 직렬화/역직렬화**를 정리한다.

---

## 직렬화(serialize)와 역직렬화(deserialize)

### 직렬화 (serialize)

직렬화는 객체의 상태를 저장하거나 전송 가능한 형태로 변환하는 과정이다.

Java native serialization에서는 객체가 바이너리 형태로 변환될 수 있고, Web API에서는 보통 JSON, XML 같은 텍스트 기반 포맷으로 변환된다.

이 글에서는 그중에서도 가장 흔하게 사용하는 JSON 변환을 다룬다.

```text
Java Object → JSON String
```

### 역직렬화 (deserialize)

역직렬화는 직렬화의 반대 과정이다.

외부에서 전달된 JSON 같은 데이터 형식을 Java 객체로 다시 변환한다.

```text
JSON String → Java Object
```

![직/역]({{site.url}}/public/image/2023/2023-06/23-deserilize001.png)

Spring MVC에서 `@RequestBody`로 JSON 요청을 DTO로 받을 때, 또는 `@RestController`에서 DTO를 응답으로 반환할 때도 이 직렬화/역직렬화 과정이 내부적으로 일어난다.

---

## Jackson과 ObjectMapper

Jackson은 Java 진영에서 가장 널리 사용되는 JSON 처리 라이브러리 중 하나다.

[Jackson Github](https://github.com/FasterXML/jackson)에서는 Jackson을 다음처럼 설명한다.

> **What is Jackson?**<br>
> Jackson has been known as "the Java JSON library" or "the best JSON parser for Java". Or simply as "JSON for Java".<br>
> More than that, Jackson is a suite of data-processing tools for Java (and the JVM platform), including the flagship streaming JSON parser / generator library,...

Spring Boot는 기본적으로 Jackson을 이용해 HTTP 요청/응답의 JSON 변환을 처리한다. 이때 핵심적으로 사용되는 클래스가 `ObjectMapper`다.

`ObjectMapper`를 사용하면 다음과 같은 변환을 직접 수행할 수 있다.

```text
Object → JSON String
JSON String → Object
Map/Object → 특정 DTO
```

---

## 예제 DTO

예제에서는 `MemberDto`를 사용한다.

처음에는 일부러 getter, setter, 기본 생성자가 없는 형태로 시작해보자.

```java
public class MemberDto {
    private String id;
    private String name;
    private String email;
    private int age;

    @Builder
    public MemberDto(String id, String name, String email, int age) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.age = age;
    }
}
```

이 DTO를 이용해 직렬화와 역직렬화가 각각 어떤 조건에서 실패하고, 무엇이 필요한지 살펴본다.

---

## Object → JSON 직렬화

`ObjectMapper.writeValueAsString()`을 사용하면 Java 객체를 JSON 문자열로 변환할 수 있다.

```java
@Test
public void serialize_테스트() throws JsonProcessingException {
    ObjectMapper objectMapper = new ObjectMapper();

    MemberDto memberObject = MemberDto.builder()
            .id("dh37789")
            .name("홍길동")
            .age(31)
            .email("kill-dong@mail.com")
            .build();

    String memberJson = objectMapper.writeValueAsString(memberObject);

    System.out.println(memberJson);
}
```

하지만 이 코드를 실행하면 다음 예외가 발생한다.

```shell
No serializer found for class com.mho.stock.facade.MemberDto and no properties discovered to create BeanSerializer
(to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS)

com.fasterxml.jackson.databind.exc.InvalidDefinitionException: No serializer found for class com.mho.stock.facade.MemberDto and no properties discovered to create BeanSerializer
```

Jackson이 `MemberDto`에서 JSON으로 만들 속성을 찾지 못했기 때문이다.

기본 설정의 Jackson은 보통 public getter를 통해 직렬화할 property를 찾는다. 현재 `MemberDto`에는 getter가 없기 때문에 Jackson 입장에서는 직렬화할 property가 없는 빈 객체처럼 보인다.

`@Getter`를 추가해보자.

```java
@Getter
public class MemberDto {
    // ...
}
```

다시 실행하면 정상적으로 JSON이 생성된다.

```shell
{"id":"dh37789","name":"홍길동","email":"kill-dong@mail.com","age":31}
```

![직/역]({{site.url}}/public/image/2023/2023-06/23-deserilize002.png)

정리하면 다음과 같다.

> 기본 설정의 Jackson은 직렬화 시 객체의 property를 찾기 위해 getter를 사용한다.
> getter가 없으면 직렬화할 속성을 찾지 못해 `FAIL_ON_EMPTY_BEANS` 예외가 발생할 수 있다.

물론 Jackson은 field visibility 설정, annotation, record 등 다른 방식도 지원한다. 여기서는 기본적인 Java Bean 규칙을 기준으로 설명한다.

---

## JSON → Object 역직렬화

이번에는 JSON 문자열을 `MemberDto` 객체로 변환해보자.

`ObjectMapper.readValue()`를 사용한다.

```java
@Test
public void deserialize_String_테스트() throws IOException {
    ObjectMapper objectMapper = new ObjectMapper();

    String memberJson = "{\"id\":\"dh37789\",\"name\":\"홍길동\",\"email\":\"kill-dong@mail.com\",\"age\":31}";
    MemberDto memberDto = objectMapper.readValue(memberJson, MemberDto.class);

    System.out.println(memberDto.toString());
}
```

이번에도 예외가 발생한다.

```shell
Cannot construct instance of `com.mho.stock.facade.MemberDto` (no Creators, like default constructor, exist):
cannot deserialize from Object value (no delegate- or property-based Creator)
 at [Source: (String)"{"id":"dh37789","name":"홍길동","email":"kill-dong@mail.com","age":31}"; line: 1, column: 2]
```

이번 예외의 핵심은 Jackson이 `MemberDto` 인스턴스를 생성할 방법을 찾지 못했다는 것이다.

기본 설정에서 Jackson은 보통 다음 흐름으로 객체를 만든다.

```text
기본 생성자로 객체 생성
    ↓
setter 또는 field 접근을 통해 값 주입
```

현재 `MemberDto`에는 기본 생성자도 없고, 값을 주입할 setter도 없다.

따라서 다음처럼 기본 생성자와 setter를 추가하면 역직렬화가 가능하다.

```java
@Getter
@Setter
@NoArgsConstructor
public class MemberDto {
    // ...
}
```

실행 결과는 다음과 같다.

```shell
MemberDto{id='dh37789', name='홍길동', email='kill-dong@mail.com', age=31}
```

![직/역]({{site.url}}/public/image/2023/2023-06/23-deserilize003.png)

정리하면 다음과 같다.

> 기본 설정의 Jackson은 역직렬화 시 객체를 만들 방법이 필요하다.
> 가장 단순한 방식은 기본 생성자와 setter를 제공하는 것이다.

다만 setter가 항상 필수인 것은 아니다. Jackson은 다음 방식도 지원한다.

- `@JsonCreator` + `@JsonProperty`
- field visibility 설정
- Java record
- builder 기반 역직렬화
- 생성자 기반 역직렬화

하지만 기본 Java Bean 스타일 DTO에서는 `@NoArgsConstructor`, `@Setter` 조합이 가장 단순하다.

---

## Map → DTO 변환: convertValue

`ObjectMapper.convertValue()`를 사용하면 이미 Java 객체로 존재하는 값을 다른 타입으로 변환할 수 있다.

예를 들어 `Map<String, Object>`에 담긴 회원 정보를 `MemberDto`로 변환할 수 있다.

```java
@Test
public void deserialize_Object_테스트() throws IOException {
    ObjectMapper objectMapper = new ObjectMapper();

    Map<String, Object> memberObject = Stream.of(
            new Object[]{"id", "dh37789"},
            new Object[]{"name", "홍길동"},
            new Object[]{"email", "kill-dong@mail.com"},
            new Object[]{"age", 31}
    ).collect(Collectors.toMap(
            data -> (String) data[0],
            data -> data[1]
    ));

    MemberDto memberDto = objectMapper.convertValue(memberObject, MemberDto.class);

    System.out.println(memberDto.toString());
}
```

결과는 다음과 같다.

```shell
MemberDto{id='dh37789', name='홍길동', email='kill-dong@mail.com', age=31}
```

![직/역]({{site.url}}/public/image/2023/2023-06/23-deserilize004.png)

`convertValue()`는 단순히 JSON 문자열을 읽는 `readValue()`와는 용도가 조금 다르다.

| 메서드 | 용도 |
|---|---|
| `writeValueAsString(object)` | Object를 JSON 문자열로 변환 |
| `readValue(json, Class<T>)` | JSON 문자열을 특정 타입 객체로 변환 |
| `convertValue(value, Class<T>)` | 이미 존재하는 Java 값을 다른 타입으로 변환 |

`convertValue()`는 다음 글에서 다룰 `List<?>`와 `LinkedHashMap` 문제를 해결할 때도 사용할 수 있다.

---

## 정리

이번 글에서는 Jackson `ObjectMapper`를 이용한 JSON 직렬화/역직렬화 기본 흐름을 살펴봤다.

핵심은 다음과 같다.

1. 이 글에서 다룬 직렬화/역직렬화는 Java native serialization이 아니라 JSON 변환이다.
2. 기본 설정의 Jackson은 직렬화할 property를 찾기 위해 getter를 주로 사용한다.
3. 기본 설정의 Jackson은 역직렬화할 때 객체를 만들 방법이 필요하다.
4. Java Bean 스타일 DTO에서는 기본 생성자와 setter가 가장 단순한 역직렬화 조건이다.
5. `convertValue()`는 `Map`이나 이미 만들어진 객체를 다른 DTO 타입으로 변환할 때 유용하다.

다음 글에서는 실제 Spring/FeignClient 응답 처리 중 `List<?>` 필드가 `List<MemberDto>`가 아니라 `List<LinkedHashMap>`으로 역직렬화되어 `ClassCastException`이 발생한 문제를 살펴본다.
