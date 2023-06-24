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

간단한 예시를 들어보자면, `FeignClient`를 이용해 MemberDto의 객체를 데이터를 A프로젝트에서 B프로젝트로 가져오는 중


#### Result

```java
@Getter
@Setter
@NoArgsConstructor
public class Result {

    private String code;
    private String message;
    private List<?> data;
}
```

`ResponseEntity\<List\<MemberDto\>\>` 와 같은 응답 객체가 아닌 위의 Result 객체와 같은 Custom 응답 객체에 `List\<?\>`의 와일드카드 타입의 List Collection 객체에 데이터를 넣어 응답을 가져올 때 발생했다.

B프로젝트에서 데이터를 가져온 뒤 가져온 MemberDto의 가공을 위해 MemberDto객체로 캐스팅을 진행하고 가공을 하려던 차에 아래의 예외가 발생했다.

```shell
org.springframework.web.util.NestedServletException: Request processing failed; nested exception is java.lang.ClassCastException: Cannot cast java.util.LinkedHashMap to kr.api.model.payment.PaymentForPersonal
```

List\<?\> 데이터가 jackson을 통해 List\<MemberDto\>의 타입으로 변환 되었을 줄 알았지만 제대로 알지 못했던 내 불찰 이었다.


## 결론

결론적으로 말하자면 Jackson에서는 응답 데이터의 타입을 찾지 못할경우 `LinkedHashMap`으로 객체를 반환한다고 한다.

그래서 응답 데이터가 List\<MemberDto\> 가 아닌 List\<LinkedHashMap\> 으로 반환이 된 것이다.

참고 : [java-lang-classcastexception-java-util-linkedhashmap-cannot-be-cast-to-com-test - stackoverflow](https://stackoverflow.com/questions/28821715/java-lang-classcastexception-java-util-linkedhashmap-cannot-be-cast-to-com-test)



