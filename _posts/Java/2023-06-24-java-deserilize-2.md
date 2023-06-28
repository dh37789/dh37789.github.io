---
title:  "[Java] Spring의 역직렬화(deserialize) 이야기"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2023-06-24
last_modified_at: 2023-06-24
---

[Java에서의 직렬화/역직렬화](https://dh37789.github.io/java/java-deserilize-1/)를 알아봤다면 이번엔 Spring에서의 직렬화/역직렬화에 대해 알아보도록 하자.


## ClassCastException

먼저 Spring의 역직렬화를 정리하게된 계기가 있다면 `ClassCastException` 에러를 만나고 나서 였다.


## 예시

간단한 예시를 들어보자면, `FeignClient`를 이용해 MemberDto의 객체를 데이터를 A프로젝트에서 B프로젝트로 가져오는 중

#### ResponseData

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

`ResponseEntity\<List\<MemberDto\>\>` 와 같은 응답 객체가 아닌 위의 ResponseData 객체와 같은 Custom 응답 객체에 `List\<?\>`의 와일드카드 타입의 List Collection 객체에 데이터를 넣어 응답을 가져올 때 발생했다.

B프로젝트에서 데이터를 가져온 뒤 가져온 `MemberDto`의 가공을 위해 `MemberDto`객체로 캐스팅을 진행하고 가공을 하려던 차에 아래의 예외가 발생했다.

```java
List<?> data = ResponseData.getData();
List<MemberDto> members = data.stream()
        .map(member -> (MemberDto) member)
        .collect(Collectors.toList());
```

```shell
org.springframework.web.util.NestedServletException: Request processing failed; nested exception is java.lang.ClassCastException: Cannot cast java.util.LinkedHashMap to kr.api.model.payment.PaymentForPersonal
```

List\<?\> 데이터가 jackson을 통해 List\<MemberDto\>의 타입으로 변환 되었을 줄 알았지만 제대로 알지 못했던 내 불찰 이었다.


## 에러의 원인?

결론적으로 말하자면 Jackson에서는 응답 데이터의 타입을 찾지 못할경우 `LinkedHashMap`으로 객체를 반환한다고 한다.

그래서 응답 데이터가 List\<MemberDto\> 가 아닌 List\<LinkedHashMap\> 으로 반환이 된 것이다.

```java
public ResponseData getMembers() {
    ResponseData response = memberFeign.getMembers();
    List<?> data = ResponseData.getData();
    List<MemberDto> members = data.stream()
            .map(member -> (MemberDto) member)
            .collect(Collectors.toList());

    return memberFeign.getMembers();
}
```

이때는 저번에 정리했던 `convertValue`를 이용해서 변환해 줘야 한다.

```java
public ResponseData getMembers() {
    ObjectMapper objectMapper = new ObjectMapper();
    ResponseData response = memberFeign.getMembers();
    List<?> data = response.getData();
    List<MemberDto> members = data.stream()
            .map(member -> objectMapper.convertValue(member, MemberDto.class))
            .collect(Collectors.toList());
    return memberFeign.getMembers();
  }
```


## Spring에서의 Deserialize

서론이 길었다. `public <T> T convertValue(Object fromValue, Class<T> toValueType)` 에서는 매개변수 `toValueType`를 이용해 클래스의 타입을 찾아 변환했다.

그렇다면 Spring에서는 json데이터를 어떻게 자동으로 찾아서 알맞은 객체에 인스턴스 및 데이터를 불어 넣어줄까?

jackson에서는 jackson-databind라는 모듈에 타입 및 Collection 또는 프레임워크별로 Deserialize와 Serialize를 구현해 놓았다.

![jackson-databind]({{site.url}}/public/image/2023/2023-06/28-des001.png)
(수 많은 Deserialize와 Serialize의 구현체들...)

결과적으로 Spring framework에서는 `Http Response` => `Json String` => `Object` 의 과정으로 API에서 받아온 응답값을 변환하는데, 과정을 간략하게 살펴보도록 하자.

-----

### Http Response To Json String

먼저 API를 통해 Http 응답으로 온 Response 데이터를 Json String으로 변환시켜주는 것은 Spring Framework core에서 진행 하고 있다.

##### ResponseEntityDecoder
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

`ResponseEntityDecoder`는 `Decoder` 인터페이스를 구현하고 있다.

Response 데이터와 Controller에서 반환타입으로 지정된 객체 타입을 가져와, Response 데이터를 String 데이터로 치환한다.

예시에서는 `public ResponseData getMembers()`를 사용하여, ResponseData 객체를 `Type type` 매개변수로 전달해준다.

이후 Decoder를 이용해 Http Respone 응답값을 String 타입으로 변환해준다.

##### HttpMessageConverterExtractor.java
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

`HttpMessageConverterExtractor` 는 `ResponseExtractor<T>` 인터페이스를 구현하였다.

String으로 변환한 Response 데이터중 body이 없을 경우 null을 반환하고, body값이 있을 경우엔 `HttpMessageConverter<T>` 인터페이스로 보내 reponse 데이터를 읽어오도록 한다.

여기서 `this.responseClass` 는 반환타입을 가져오며 예시에서의 `ResponseData` 객체를 가져온다.

##### AbstractJackson2HttpMessageConverter
```java
@Override
public Object read(Type type, @Nullable Class<?> contextClass, HttpInputMessage inputMessage)
    throws IOException, HttpMessageNotReadableException {

    JavaType javaType = getJavaType(type, contextClass);
    return readJavaType(javaType, inputMessage);
}
```

![jackson-databind]({{site.url}}/public/image/2023/2023-06/28-des004.png)

`AbstractJackson2HttpMessageConverter` 는 `HttpMessageConverter<T>` 인터페이스를 상속받은 `GenericHttpMessageConverter<T>`를 구현하였다.

이제 여기서 Response 데이터를 jackson의 `ObjectMapper`으로 넘겨주는 역할을 한다.

`getJavaType(type, contextClass)` 에서는 `ObejctMapper.constructType` 을 이용해 해당 타입으로 Object를 매개변수로 가져온 type으로 캐스팅하여 **반환 타입의 인스턴스를 가져온다.**

##### TypeFactory
```java
protected JavaType _fromAny(ClassStack context, Type srcType, TypeBindings bindings)
    {
        JavaType resultType;

        // simple class?
        if (srcType instanceof Class<?>) {
            // Important: remove possible bindings since this is type-erased thingy
            resultType = _fromClass(context, (Class<?>) srcType, EMPTY_BINDINGS);
        }
        // But if not, need to start resolving.
        else if (srcType instanceof ParameterizedType) {
            resultType = _fromParamType(context, (ParameterizedType) srcType, bindings);
        }
        else if (srcType instanceof JavaType) { // [databind#116]
            // no need to modify further if we already had JavaType
            return (JavaType) srcType;
        }
        ...
    }
```

그다음 `readJavaType(javaType, inputMessage)`에서 Response 데이터와 `InputStream`의 데이터스트림 형태로 `ObjectReader`에 넘겨준다.


### Json String To Object

`ObjectReader`에서는 받아온 데이터스트림을 이용해 json 데이터의 역직렬화(Deserialize)를 진행한다. 이후 최종적으로 완성된 객체를 반환다.

이제 역직렬화 되는 과정을 살펴보도록 하자.

##### ObjectReader
```java
protected Object _bindAndClose(JsonParser p0) throws IOException
{
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
        // No need to consume the token as parser gets closed anyway
        if (_config.isEnabled(DeserializationFeature.FAIL_ON_TRAILING_TOKENS)) {
            _verifyNoTrailingTokens(p, ctxt, _valueType);
        }
        return result;
    }
}
```

여기서 반환되는 Object 타입의 result는 Json 데이터를 Deserialize 하여 최종적으로 완성된 반환 객체를 말한다.

if문 안에서 token으로 전달되는 문자가 Json의 형태인 "{", "}", "[", "]"와 같은 문법인지를 비교해 result에 Deserialize한 데이터를 넣어준다.


### Deserialize

위에서 말한 수많은 Deserialize의 분기는 `ObjectReader._findRootDeserializer` 메소드에서 생성자 주입을 통해 들어온 `this._rootDeserializer` 변수를 통해서 결정된다.

```java
protected JsonDeserializer<Object> _findRootDeserializer(DeserializationContext ctxt)
        throws DatabindException
    {
        if (_rootDeserializer != null) {
            return _rootDeserializer;
        }
        ...
    }
```

Spring에서는 `BeanDeserializer`의 구현체를 이용해 Deserialize를 진행하기 때문에, `this._rootDeserializer` 전역변수에 `BeanDeserializer`의 빈이 주입되어 있다.

##### DefaultDeserializationContext
```java
public Object readRootValue(JsonParser p, JavaType valueType,
            JsonDeserializer<Object> deser, Object valueToUpdate)
        throws IOException
{
    ...
    if (valueToUpdate == null) {
        return deser.deserialize(p, this);
    }
    return deser.deserialize(p, this, valueToUpdate);
}
```

`DefaultDeserializationContext` 에서는 매개변수 `JsonDeserializer<Object> deser`에서 실제 Deserialize를 진행할 `JsonDeserializer<T>`의 구현체 `BeanDeserializer`로 데이터를 넘겨준다.

##### BeanDeserializer
```java
@Override
public Object deserialize(JsonParser p, DeserializationContext ctxt) throws IOException
{
    // common case first
    if (p.isExpectedStartObjectToken()) {
        ...
        return deserializeFromObject(p, ctxt);
    }
    return _deserializeOther(p, ctxt, p.currentToken());
}
```

`deserialize` 의 구현 메소드에서 `deserializeFromObject`로 넘겨줘 실제 Object에 Property를 맵핑을 진행하도록 한다.

##### BeanDeserializer
```java
@Override
    public Object deserializeFromObject(JsonParser p, DeserializationContext ctxt) throws IOException
    {
        ...
        if (p.hasTokenId(JsonTokenId.ID_FIELD_NAME)) {
            String propName = p.currentName();
            do {
                p.nextToken();
                SettableBeanProperty prop = _beanProperties.find(propName);
                if (prop != null) { // normal case
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

위 예시의 `ResponseData`의 property는 `int code`, `String message`, `List<?> data` 이렇게 세가지이다.

그러면 `String propName = p.currentName();` 에서 property의 이름인 `code`, `message`, `data`를 각각 String으로 가져와 property의 타입을 분류한다.

이후 가져온 property 정보를 이용해 `prop.deserializeAndSet(p, ctxt, bean)` 에서 Deserialize를 진행한다.

여기서 타입별로 Deserialize를 진행하는 방법을 간단하게 정리하자면 아래와 같다.

- **String은 `StringDeserializer`**
- **int,long과 같은 숫자 타입은 `NumberDeserializer`**
- **List나 Map과 같은 Collection은 `CollectionDeserializer`**
- **그외 타입을 특정할 수 없는 객체는 `UntypedObjectDeserializerNR`**

그렇다면 예시의 `ResponseData`의 상황에 대입하여 확인해보자.

`int code`는 숫자 타입이므로, `NumberDeserializer`

`String message`는 String 타입이므로 `StringDeserializer`

`List\<?\> data`는 컬렉션 객체 `List` 내부에 직접적으로 타입이 지정되지 않은 와일드카드 형식의 객체 `\<?\>`가 들어갔기 때문에 두번의 Deserialize를 진행한다.

먼저 `CollectionDeserializer`을 이용해 `List` 객체를 역직렬화 한 뒤, `List` 원소 내부의 객체에 대한 역직렬화를 진행한다. 하지만 와일드카드 제네릭 `\<?\>`은 타입을 특정할 수 없으므로 `UntypedObjectDeserializerNR` 에서 LinkedHashMap으로 반환한다.

이렇게 최종적으로 완성된 객체 데이터를 위에서 말한 `ObjectReader`가 `Object` 반환타입으로 반환하게 된다.


### 특정되지 않은 타입?

부가적으로 처음에 왜 특정되지 않은 타입이 어떻게 `LinkedHashMap`의 형식으로 가져오나에 대해 확인해보니 아래와 같았다.

jackson-databind에서는 `UntypedObjectDeserializerNR` 역직렬화 구현체를 이용하여 특정되지 않은 타입이 object 형식일 경우에는 `LinkedHashMap`, 배열타입일 경우에는 `ArrayList` 타입으로 반환하도록 되어 있었다.

##### UntypedObjectDeserializerNR
```java
private Object _deserializeNR(JsonParser p, DeserializationContext ctxt,
            Scope rootScope)
        throws IOException
{
    ...
    outer_loop:
    while (true) {
        if (currScope.isObject()) {
            String propName = p.nextFieldName();
            ...
            for (; propName != null; propName = p.nextFieldName()) {
                Object value;
                ...
                currScope.putValue(propName, value);
            }
            ...
        } else {
            // Otherwise we must have an Array
            arrayLoop:
            while (true) {
                JsonToken t = p.nextToken();
                Object value;
                ...
                currScope.addValue(value);
            }
        }
    }
}
```

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

json 형태가 Object일 경우 LinkedHashMap의 타입으로 만들어 property의 변수명을 key값, 내용을 value값을 put해서 반환하고 있다.

```java
public void addValue(Object value) {
    if (_list == null) {
        _list = new ArrayList<>();
    }
    _list.add(value);
}
```

json 형태가 배열일 경우 ArrayList의 타입으로 만들어 value값을 add해서 반환하고 있다.


## 그래서?

처음 당연히 타입을 받아올 거라 생각했던 `List\<?\>`에서 `ClassCastException`가 발생해 분석하다보니 나의 무지가 너무 창피하기도 했고, 다시 생각해보니 어떻게 응답 데이터를 타입에 맞게 변환해오는지 궁금증이 생겼다.

그래서 위와 같이 분석을 하게 된건데 이렇게 또 분석해보니 뿌듯하기도하고, 재미도 있었던것 같다.

앞으론 내가 사용하는 것들이 최소한 어떻게 돌아가는지는 알아야겠다..라는 생각이 들었다.


참고 : [java-lang-classcastexception-java-util-linkedhashmap-cannot-be-cast-to-com-test - stackoverflow](https://stackoverflow.com/questions/28821715/java-lang-classcastexception-java-util-linkedhashmap-cannot-be-cast-to-com-test)



