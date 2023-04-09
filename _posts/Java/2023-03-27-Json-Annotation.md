---
title:  "[Java] No serializer found for class (Jackson Databind 에러) 해결하기"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2023-03-27
last_modified_at: 2022-03-27
---

## No serializer found for class

테스트 코드를 작성 하던 도중 테스트를 실행하니 아래와 같은 에러가 발생했다.

```shell
org.springframework.web.util.NestedServletException: Request processing failed;
nested exception is org.springframework.http.converter.HttpMessageConversionException: Type definition error: [simple type, class com.example.TestClass];
nested exception is com.fasterxml.jackson.databind.exc.InvalidDefinitionException: No serializer found for class com.example.Class and no properties discovered to create BeanSerializer
(to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS) (through reference chain: com.example.TestClass2)
```

여기서 핵심으로 봐야하는 에러문구는 아래의 문구이다.

```shell
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: No serializer found for class com.example.Class and no properties discovered to create BeanSerializer
```

Jackson 라이브러리가 TestClass와 TestClass 내부의 properties를 직렬화(Serializer) 하지 못해 데이터 바인딩에 실패하였기 때문에 발생한 에러이다.

> Serialize 란?
>
> 자바 Object 또는 Data를 외부의 byte 형태로 데이터를 변환하는 기술
> 반대로 Deserialize 가 있으며 이는 반대로 byte 형태의 데이터를 자바의 Object 또는 Data로 변환하는걸 뜻한다.

Jackson 라이브러리는 Serialize 하는 과정에서 접근 제한자가 public이거나 getter/setter를 이용한다.

만약 둘중 하나도 없다면 위와 같은 에러가 발생하게 된다.


## 해결방법

해결방법은 여러가지 방법이 있다.


### Getter 추가

변수의 접근제어자를 public으로 만드는 방법도 있겠지만 이는 좋지 않은 생각이다.
Getter를 만들어 Jackson 라이브러리가 접근할 수 있도록 해주자.

```java
public class People {
    private String name;

    public String getName() {
        return this.name;
    }
}
```

### @JsonProperty 속성 설정

@JsonProperty는 Json 데이터로 Serialize 할때 사용할 이름을 지정하는 Jackson annotation이다.

아래와 같이 속성을 추가해 준다.

```java
public class People {

    @JsonProperty("name")
    private String name;

    public String getName() {
        return this.name;
    }
}
```

### @JsonAutoDetect 속성 설정

@JsonAutoDetect는 필드의 직렬화 대상의 속성을 재정의 하는 Class annotation이다.

다시 말하자면 필드의 직렬화 대상을 직접 지정해 Jackson이 접근 할 수 있도록 하는 어노테이션이다.

아래와 같이 클래스 위에 Annotaion을 지정하면 된다.

ANY, DEFAULT, NON_PRIVATE, NONE, PROTECTED_ANT_PRIVATE, PUBLIC_ONLY 의 fieldVisibility을 지정해 줄 수 있다.

이중 ANY는 모든 접근제어자를 접근하는것을 말하고, DEFAULT는 시스템의 기본값을 사용하는 것, NONE은 AutoDetect의 자동감지를 사용하지 않을때 설정한다.

보통 fieldVisibility의 속성을 ANY로 지정해 public부터 private의 접근제어자 까지 모두 접근할 수 있도록 지정해 준다.

```java
@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.ANY)
public class People {

    @JsonProperty("name")
    private String name;

    public String getName() {
        return this.name;
    }
}
```


참고 : [JacksonAnnotations Docs](https://github.com/FasterXML/jackson-docs/wiki/JacksonAnnotations)

