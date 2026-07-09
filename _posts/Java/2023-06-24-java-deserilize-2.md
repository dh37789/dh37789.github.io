---
title:  "[Java] Jackson 역직렬화 이해하기 2: List<?>가 LinkedHashMap이 되는 이유"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2023-06-24
last_modified_at: 2026-07-09
---

[이전 글](https://dh37789.github.io/java/java-deserilize-1/)에서는 Jackson `ObjectMapper`를 이용해 객체를 JSON으로 직렬화하고, JSON을 다시 객체로 역직렬화하는 기본 흐름을 살펴봤다.

이번 글에서는 실제로 겪었던 `ClassCastException` 문제를 다룬다.

FeignClient로 다른 서비스의 회원 목록을 조회하고 있었다. 응답은 정상적으로 내려왔고, `data` 필드에도 값이 들어 있었다. 그런데 `data`의 각 원소를 `MemberDto`로 캐스팅하는 순간 다음 예외가 발생했다.

```shell
java.lang.ClassCastException: Cannot cast java.util.LinkedHashMap to MemberDto
```

처음에는 Jackson이 응답 JSON을 알아서 `List<MemberDto>`로 변환해줄 것이라고 생각했다. 하지만 문제는 응답 DTO의 `data` 필드가 `List<?>`로 선언되어 있었다는 점이었다.

---

## 문제 상황

A 프로젝트에서 FeignClient를 이용해 B 프로젝트의 회원 목록을 조회하고 있었다.

응답 객체는 다음과 같은 커스텀 wrapper 형태였다.

```java
@Getter
@Setter
@NoArgsConstructor
public class ResponseData {

    private int code;
    private String message;
    private List<?> data;
}
```

`ResponseEntity<List<MemberDto>>`처럼 명확한 응답 타입을 사용한 것이 아니라, `ResponseData` 안에 `List<?>` 타입의 `data` 필드가 들어 있는 구조였다.

응답을 받은 뒤 `data`를 꺼내 `MemberDto`로 캐스팅하려고 했다.

```java
List<?> data = responseData.getData();

List<MemberDto> members = data.stream()
        .map(member -> (MemberDto) member)
        .collect(Collectors.toList());
```

하지만 실행 결과는 다음과 같았다.

```shell
org.springframework.web.util.NestedServletException: Request processing failed;
nested exception is java.lang.ClassCastException:
Cannot cast java.util.LinkedHashMap to MemberDto
```

`data`의 원소가 `MemberDto`가 아니라 `LinkedHashMap`이었던 것이다.

---

## 왜 LinkedHashMap이 되었을까?

결론부터 말하면 Jackson은 `List<?>`의 원소 타입을 알 수 없다.

`ResponseData`의 필드는 다음처럼 선언되어 있다.

```java
private List<?> data;
```

여기서 `List`라는 컬렉션 타입은 알 수 있지만, 리스트 안에 들어갈 원소 타입은 알 수 없다.

즉 Jackson 입장에서는 다음과 같이 보인다.

```text
data는 List다.
그런데 List 안의 원소 타입은 모른다.
```

JSON 배열 안에 객체가 들어 있다면, 원소 타입을 알 수 있을 때는 해당 DTO로 만들 수 있다.

```java
List<MemberDto>
```

하지만 원소 타입을 알 수 없다면 JSON object를 특정 DTO로 만들 수 없다. 이 경우 Jackson은 타입이 특정되지 않은 object를 기본적으로 `LinkedHashMap`으로 역직렬화한다.

따라서 실제 결과는 다음에 가까웠다.

```text
ResponseData
  code: 200
  message: "success"
  data: List<LinkedHashMap>
```

그래서 `MemberDto`로 직접 캐스팅하면 `ClassCastException`이 발생한다.

---

## convertValue로 후처리하기

이미 `List<?>`로 받은 상태라면, 각 원소는 `LinkedHashMap` 형태일 가능성이 높다.

이 경우 이전 글에서 다룬 `ObjectMapper.convertValue()`를 이용해 `LinkedHashMap`을 `MemberDto`로 다시 변환할 수 있다.

```java
public ResponseData getMembers() {
    ObjectMapper objectMapper = new ObjectMapper();

    ResponseData response = memberFeign.getMembers();
    List<?> data = response.getData();

    List<MemberDto> members = data.stream()
            .map(member -> objectMapper.convertValue(member, MemberDto.class))
            .collect(Collectors.toList());

    // members 후속 처리

    return response;
}
```

`convertValue()`는 이미 Java 객체로 만들어진 값을 다른 타입으로 변환한다.

```text
LinkedHashMap → MemberDto
```

이 방법은 이미 타입 정보를 잃은 뒤 후처리해야 할 때 현실적인 해결책이다.

하지만 가장 좋은 해결책은 애초에 타입 정보를 잃지 않게 설계하는 것이다.

---

## 더 나은 해결: 응답 타입을 명확히 하기

`convertValue()`는 문제를 해결할 수 있지만, 근본적으로는 `List<?>`를 사용한 응답 구조가 문제를 만든다.

가능하다면 응답 DTO를 제네릭으로 선언하는 편이 낫다.

```java
@Getter
@Setter
@NoArgsConstructor
public class ResponseData<T> {

    private int code;
    private String message;
    private T data;
}
```

그리고 FeignClient의 반환 타입을 명확히 적는다.

```java
@FeignClient(name = "member-service")
public interface MemberFeignClient {

    @GetMapping("/members")
    ResponseData<List<MemberDto>> getMembers();
}
```

이렇게 하면 Jackson은 `data`의 타입을 다음처럼 알 수 있다.

```text
ResponseData<List<MemberDto>>
```

즉, `data`가 List이고, 그 안의 원소가 `MemberDto`라는 타입 정보가 보존된다.

가능하다면 wrapper 없이 직접 명확한 타입을 반환하는 것도 방법이다.

```java
@GetMapping("/members")
List<MemberDto> getMembers();
```

정리하면 해결책의 우선순위는 다음과 같다.

| 우선순위 | 해결책 | 설명 |
|---:|---|---|
| 1 | 응답 DTO를 제네릭으로 선언 | `ResponseData<List<MemberDto>>`처럼 타입 정보 보존 |
| 2 | FeignClient 반환 타입을 구체화 | 가능한 한 `List<?>`나 `Object` 반환을 피함 |
| 3 | `TypeReference`/`JavaType` 사용 | ObjectMapper 직접 사용 시 제네릭 타입 전달 |
| 4 | `convertValue()` 후처리 | 이미 `LinkedHashMap`으로 받은 경우 실용적 |

---

## Spring/Feign/Jackson의 처리 흐름

문제를 이해하려면 각 계층의 역할을 나누어 보는 것이 좋다.

이번 문제의 흐름은 대략 다음과 같다.

```text
FeignClient 호출
    ↓
HTTP Response 수신
    ↓
Feign Decoder
    ↓
SpringDecoder / HttpMessageConverter
    ↓
MappingJackson2HttpMessageConverter
    ↓
ObjectMapper / ObjectReader
    ↓
Jackson Databind
    ↓
ResponseData<List<LinkedHashMap>>
```

즉, 이 문제는 단순히 “Spring이 이상하게 변환했다”기보다는, Feign 응답 디코딩 과정에서 Spring의 `HttpMessageConverter`와 Jackson이 대상 타입 정보를 바탕으로 역직렬화를 수행한 결과다.

그리고 대상 타입이 `ResponseData`였기 때문에 `data` 내부 원소 타입까지는 알 수 없었다.

---

## ResponseEntityDecoder

FeignClient 응답 처리 과정에서 `ResponseEntityDecoder`는 `ResponseEntity<T>` 타입인지 확인하고, 실제 응답 body 처리는 내부 decoder에 위임한다.

```java
@Override
public Object decode(final Response response, Type type) throws IOException, FeignException {
    if (isParameterizeHttpEntity(type)) {
        ...
        return createResponse(decodedObject, response);
    }
    else if (isHttpEntity(type)) {
        return createResponse(null, response);
    }
    else {
        return this.decoder.decode(response, type);
    }
}
```

![jackson-databind]({{site.url}}/public/image/2023/2023-06/28-des002.png)

여기서 중요한 점은 `ResponseEntityDecoder` 자체가 JSON을 문자열로 바꿔 DTO로 만드는 것이 아니라, 응답 타입을 판단한 뒤 실제 처리를 내부 decoder에 넘긴다는 것이다.

---

## HttpMessageConverter

Spring 쪽에서는 `HttpMessageConverter`가 HTTP body를 Java 객체로 읽는 역할을 한다.

```java
@Override
@SuppressWarnings({"unchecked", "rawtypes", "resource"})
public T extractData(ClientHttpResponse response) throws IOException {
    IntrospectingClientHttpResponse responseWrapper = new IntrospectingClientHttpResponse(response);

    if (!responseWrapper.hasMessageBody() || responseWrapper.hasEmptyMessageBody()) {
        return null;
    }

    MediaType contentType = getContentType(responseWrapper);

    try {
        for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
            ...
            if (this.responseClass != null) {
                if (messageConverter.canRead(this.responseClass, contentType)) {
                    ...
                    return (T) messageConverter.read((Class) this.responseClass, responseWrapper);
                }
            }
        }
    }
    ...
}
```

![jackson-databind]({{site.url}}/public/image/2023/2023-06/28-des003.png)

응답 body가 존재하고, Content-Type과 대상 타입을 처리할 수 있는 converter가 있으면 해당 converter가 body를 읽는다.

JSON 응답이라면 일반적으로 Jackson 기반 converter가 선택된다.

---

## AbstractJackson2HttpMessageConverter

Jackson 기반 converter는 대상 타입을 Jackson의 `JavaType`으로 변환한 뒤, `ObjectMapper`를 이용해 body를 읽는다.

```java
@Override
public Object read(Type type, @Nullable Class<?> contextClass, HttpInputMessage inputMessage)
    throws IOException, HttpMessageNotReadableException {

    JavaType javaType = getJavaType(type, contextClass);
    return readJavaType(javaType, inputMessage);
}
```

![jackson-databind]({{site.url}}/public/image/2023/2023-06/28-des004.png)

여기서 `getJavaType(type, contextClass)`는 Java의 `Type` 정보를 Jackson이 이해할 수 있는 `JavaType`으로 바꾼다.

이 과정에서 `ResponseData<List<MemberDto>>`처럼 제네릭 타입 정보가 있으면 Jackson은 더 구체적인 타입 정보를 알 수 있다.

반대로 단순히 `ResponseData` 또는 `ResponseData` 내부의 `List<?>`만 있으면 리스트 원소 타입은 알 수 없다.

---

## ObjectReader와 JsonDeserializer

Jackson은 `ObjectReader`를 통해 JSON token을 읽고, 대상 `JavaType`에 맞는 deserializer를 찾아 객체를 만든다.

```java
protected Object _bindAndClose(JsonParser p0) throws IOException {
    try (JsonParser p = p0) {
        Object result;

        final DefaultDeserializationContext ctxt = createDeserializationContext(p);
        JsonToken t = _initForReading(ctxt, p);

        if (t == JsonToken.VALUE_NULL) {
            if (_valueToUpdate == null) {
                result = _findRootDeserializer(ctxt).getNullValue(ctxt);
            } else {
                result = _valueToUpdate;
            }
        } else if (t == JsonToken.END_ARRAY || t == JsonToken.END_OBJECT) {
            result = _valueToUpdate;
        } else {
            result = ctxt.readRootValue(p, _valueType, _findRootDeserializer(ctxt), _valueToUpdate);
        }

        return result;
    }
}
```

대상 타입이 일반 POJO라면 `BeanDeserializer` 계열 deserializer가 선택된다. 이 deserializer는 JSON field 이름과 Java 객체의 property를 매핑한다.

```java
@Override
public Object deserializeFromObject(JsonParser p, DeserializationContext ctxt) throws IOException {
    ...
    if (p.hasTokenId(JsonTokenId.ID_FIELD_NAME)) {
        String propName = p.currentName();
        do {
            p.nextToken();
            SettableBeanProperty prop = _beanProperties.find(propName);
            if (prop != null) {
                try {
                    prop.deserializeAndSet(p, ctxt, bean);
                } catch (Exception e) {
                    wrapAndThrow(e, bean, propName, ctxt);
                }
                continue;
            }
            handleUnknownVanilla(p, ctxt, bean, propName);
        } while ((propName = p.nextFieldName()) != null);
    }
    return bean;
}
```

예제의 `ResponseData` 필드는 다음 세 개다.

```java
private int code;
private String message;
private List<?> data;
```

각 필드에 대해 Jackson은 타입에 맞는 deserializer를 선택한다.

| 필드 | 타입 | 사용되는 deserializer 예시 |
|---|---|---|
| `code` | `int` | NumberDeserializer |
| `message` | `String` | StringDeserializer |
| `data` | `List<?>` | CollectionDeserializer + UntypedObjectDeserializer |

`data`는 List이므로 컬렉션 자체는 `CollectionDeserializer`가 처리한다. 하지만 리스트 내부 원소 타입은 `?`라서 구체 타입을 알 수 없다.

따라서 각 원소가 JSON object라면 `UntypedObjectDeserializer` 계열이 처리하고, 기본적으로 `LinkedHashMap`으로 만들어진다.

---

## 타입이 없는 Object는 어떻게 LinkedHashMap이 될까?

Jackson Databind의 untyped object 처리에서는 JSON object를 `Map` 형태로 만든다.

예를 들어 `UntypedObjectDeserializerNR` 내부에는 object 값을 map에 넣는 흐름이 있다.

```java
public void putValue(String key, Object value) {
    if (_squashDups) {
        _putValueHandleDups(key, value);
        return;
    }
    if (_map == null) {
        _map = new LinkedHashMap<>();
    }
    _map.put(key, value);
}
```

JSON object의 field 이름은 key가 되고, field 값은 value가 된다.

```text
{
  "id": "dh37789",
  "name": "홍길동"
}
```

위 JSON object는 타입이 특정되지 않은 상황에서 대략 다음 형태가 된다.

```java
LinkedHashMap<String, Object>
```

배열은 `ArrayList`로 만들어진다.

```java
public void addValue(Object value) {
    if (_list == null) {
        _list = new ArrayList<>();
    }
    _list.add(value);
}
```

그래서 `List<?> data`는 결과적으로 다음처럼 만들어질 수 있다.

```text
List<LinkedHashMap<String, Object>>
```

---

## 정리

이번 문제의 핵심은 `List<?>`가 편해 보이지만, 역직렬화 시 원소 타입 정보를 잃는다는 점이었다.

처음에는 FeignClient와 Jackson이 응답 JSON을 자동으로 `List<MemberDto>`로 만들어줄 것이라 생각했다. 하지만 `ResponseData`의 `data` 필드는 `List<?>`였고, Jackson은 리스트 내부 원소 타입을 알 수 없었다.

그 결과 JSON object는 `MemberDto`가 아니라 `LinkedHashMap`으로 역직렬화되었다.

```text
List<?> data
    ↓
원소 타입을 알 수 없음
    ↓
JSON object를 LinkedHashMap으로 역직렬화
    ↓
MemberDto 캐스팅 시 ClassCastException
```

실무적으로는 다음 순서로 해결하는 것이 좋다.

1. 가능하면 응답 타입을 `ResponseData<List<MemberDto>>`처럼 명확히 선언한다.
2. `List<?>`, `Object`, raw type을 응답 DTO에 사용하는 것을 피한다.
3. 이미 `LinkedHashMap`으로 받은 경우에는 `ObjectMapper.convertValue()`로 후처리한다.
4. ObjectMapper를 직접 사용할 때는 `TypeReference`나 `JavaType`으로 제네릭 타입 정보를 전달한다.

결국 중요한 것은 Jackson이 어떤 타입으로 역직렬화해야 하는지 알 수 있도록 타입 정보를 잃지 않는 것이다.

---

## 마무리

이번 문제는 단순한 캐스팅 오류처럼 보였지만, 실제로는 FeignClient 응답 디코딩과 Jackson 역직렬화에서 타입 정보가 어떻게 사용되는지 이해해야 해결할 수 있었다.

특히 `List<?>`는 “아무 타입이나 담을 수 있다”는 점에서 편해 보이지만, 역직렬화 관점에서는 “원소 타입을 알 수 없다”는 의미가 된다.

앞으로는 외부 API 응답 DTO를 설계할 때 wrapper 구조를 사용하더라도 내부 데이터 타입을 제네릭으로 명확히 표현해야겠다고 느꼈다.

참고: [java-lang-classcastexception-java-util-linkedhashmap-cannot-be-cast-to-com-test - stackoverflow](https://stackoverflow.com/questions/28821715/java-lang-classcastexception-java-util-linkedhashmap-cannot-be-cast-to-com-test)
