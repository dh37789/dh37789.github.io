---
title:  "[Java] Java의 직렬화(serialize)/역직렬화(deserialize)"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2023-06-19
last_modified_at: 2023-06-19
---

자바에서 직렬화(serialize)와 역직렬화(deserialize)에 대해 간단하게 톺아보고 가도록 하자.


## 직렬화 (serialize)

직렬화란 자바의 객체(Object)의 상태를 바이트 형태의 데이터 스트림으로 변환하는 것을 말한다.

데이터 스트림은 흔히 우리가 자주 쓰는 `json` 형식이 될 수 있고 또는 `xml` 과 같은 다양한 형태로 변환이 가능하다.


## 역직렬화 (deserialize)

역직렬화란 직렬화와는 반대로 바이트 형태의 데이터 스트림을 자바의 객체(Object)의 상태로 변환해주는 것을 말한다.

![직/역]({{site.url}}/public/image/2023/2023-06/23-deserilize001.png)


이제 간단하게 개념을 살펴봤으니, 알아보고자 하는 역직렬화에 대해 살펴보고자 한다.

자바에서는 어떻게 JVM의 메모리상 남아있는 Object 데이터를 데이터 스트림으로 서로 변환을 시킬 수 있을까?<br>
또 어떻게 자바에서는 데이터 스트림에서 자바에 선언한 Object의 타입을 찾아서 인스턴스를 넣어줄 수 있는걸까?


## Jackson

자바에서는 대표적인 데이터 처리 라이브러리로 Jackson 라이브러리를 사용하고 있다.

[Jackson 라이브러리의 Github 페이지](https://github.com/FasterXML/jackson)에서 살펴보는 Jackson의 정의는 다음과 같다.

> **What is Jackson?**<br>
> Jackson has been known as "the Java JSON library" or "the best JSON parser for Java". Or simply as "JSON for Java".<br>
> More than that, Jackson is a suite of data-processing tools for Java (and the JVM platform), including the flagship streaming JSON parser / generator library,...

직역해보자면, Json parse의 라이브러리로 알려져 있는 Jackson의 라이브러리는 더 나아가 Java(및 JVM 플랫폼)의 데이터 처리 도구의 모음이다. 라는 설명이다.

실제로 Spring에서는 Jackson 라이브러리에 자체적으로 추가 기능을 더해 공식적으로 지원하고 있다.

주요 클래스는 `ObjectMapper`라는 클래스로 해당 클래스에서 데이터의 변환을 해주고 있다.


## ObjectMapper를 이용한 직렬화 및 역직렬화

MemberDto 객체를 이용해서 직렬화 및 역직렬화를 진행보려고 한다.


- MemberDto

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


### Serialize (직렬화)

`ObjectMapper` 객체에는 serialize를 지원하는 `writeValueAsString(Object object)` 메서드를 이용해서 `Object` => `Json(String)` 으로 변환이 가능하다.

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

하지만 테스트 코드를 실행하면 에러가 발생한다.

```shell
No serializer found for class com.mho.stock.facade.MemberDto and no properties discovered to create BeanSerializer (to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS)
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: No serializer found for class com.mho.stock.facade.MemberDto and no properties discovered to create BeanSerializer (to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS)
```

요약하자면 객체를 BeanSerialize 진행 할 수 없다는 뜻이다.

객체를 Serialize를 하기 위해서는 객체의 필드에서 데이터를 뽑아와야 하는데, Jackson에서는 Getter를 이용해 객체의 필드에 접근하고 있다. 따라서 접근자 메서드인 `@Getter`가 있어야 한다.

@Getter를 선언해준뒤 테스트를 다시 실행해 보도록 하자.

```java
@Getter
public class MemberDto {
  ...
}
```

```shell
{"id":"dh37789","name":"홍길동","email":"kill-dong@mail.com","age":31}
```

![직/역]({{site.url}}/public/image/2023/2023-06/23-deserilize002.png)


### Deserialize (역직렬화)

`ObjectMapper` 객체에는 deserialize를 지원하는 메소드 또한 있는데 두가지가 존재한다.

####`public <T> T readValue(String content, Class<T> valueType)` : Json(content)을 Object(valueType)로 변환한다.

`String` 타입의 `json` 데이터를 받아 `MemberDto.class`라는 객체타입이라는 것을 알리고, `MemberDto` 객체로 인스턴스화 한다.

```java
@Test
public void deserialize_String_테스트() throws IOException {
    ObjectMapper objectMapper = new ObjectMapper();

    String memberJson = "{\"id\":\"dh37789\",\"name\":\"홍길동\",\"email\":\"kill-dong@mail.com\",\"age\":31}";
    MemberDto memberDto = objectMapper.readValue(memberJson, MemberDto.class);

    System.out.println(memberDto.toString());
}
```

Deserialize를 진행해도 동일하게 에러가 발생한다.

```shell
Cannot construct instance of `com.mho.stock.facade.MemberDto` (no Creators, like default constructor, exist): cannot deserialize from Object value (no delegate- or property-based Creator)
 at [Source: (String)"{"id":"dh37789","name":"홍길동","email":"kill-dong@mail.com","age":31}"; line: 1, column: 2]
```

`Json` 타입을 객체의 필드에 주입해 주기 위해서는 위와 비슷하게 Jackson에서는 Setter를 이용해 객체의 필드에 주입해 주고있다. 따라서 접근자 메서드인 `@Setter`가 있어야 한다.

또한 필드의 데이터 외에 객체의 인스턴스를 주입해 주어야 하기 때문에, 생성자도 같이 생성 해 주어야 한다.

```java
@Getter
@Setter
@NoArgsConstructor
public class MemberDto {
  ...
}
```

```shell
MemberDto{id='dh37789', name='홍길동', email='kill-dong@mail.com', age=31}
```

![직/역]({{site.url}}/public/image/2023/2023-06/23-deserilize003.png)


#### `public <T> T convertValue(Object fromValue, Class<T> toValueType)` : Object(fromValue)를 Object(toValueType)로 변환한다.

Member의 정보가 들어있는 `Map`객체를 `convertValue`를 이용해 `MemberDto.class`라는 객체타입이라는 것을 알리고, `MemberDto` 객체로 인스턴스화 한다.

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

    /** MemberDto{id='dh37789', name='홍길동', email='kill-dong@mail.com', age=31} */
    System.out.println(memberDto.toString());
}
```

`convertValue`도 동일하게 `@Setter` 및 생성자를 통해 주입해주는데 이미 추가를 한 상태이므로 정상적으로 실행이 된다.

```shell
MemberDto{id='dh37789', name='홍길동', email='kill-dong@mail.com', age=31}
```

![직/역]({{site.url}}/public/image/2023/2023-06/23-deserilize004.png)
