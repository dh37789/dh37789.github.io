---
title:  "[Java] Feign Client에서 페이징 처리하기"

categories: Java

toc: true
toc_sticky: true

date: 2022-09-03
last_modified_at: 2022-09-03
---

# Feign Client에서 페이징 처리하기

Feign Client는 넷플릭스에서 만든 외부 API를 쉽게 호출할수 있도록 도와주는 라이브러리입니다.

주로 MSA 환경에서 많이 쓰이는 기술이다.

```java
@FeignClient(value = "회원Client", url = "${client.url}")
public interface MemberClient {
    
    @PostMapping("/member/{id}")
    void getMember(@PathVariable("id") Long id);
}
```

위와 같이 interface에 api의 url과 메서드를 선언만 해주면 자동으로 해당 경로의 api를 호출하는 방식을 가지고 있다.

Feign Client의 자세한 사항은 다음에 알아보도록 하고, 편리한 기술이지만 JPA 환경에서 아쉬운 부분이 몇가지가 있다.

응답값으로 컬렉션이나, 객체로 받아올 수 는 있지만, page객체로 받아올 수가 없다. 만약 서버측 Api에서 pageImpl의 객체로 반환해준다고 예시를 들어보자

- Client 호출부

```java
@FeignClient(value = "회원Client", url = "${client.url}")
public interface MemberClient {
    
    @PostMapping("/member")
    Page<MemeberDto> getMemberList(Integer page, Integer size);
}
```

- Server 응답부

```java
@PostMapping("/member")
public Page<MemeberDto> getMemberList(Pageable pageable) {
    ...
    return new PageImpl<>(memberList, pageable, totalCount);        
}
```

이렇게 pageImpl을 통해 응답값을 전달해도 page객체에 List객체를 담을 수 없다는 Mapping관련 Exception만 발생한다.  
이렇게 FeignClient는 page관련 객체를 지원하지 않는다.

이를 간단하게 나마 해결 할 수 있는 방법이 있다.

응답값으로 List의 값들을 json으로 파싱하여 담아주면 된다.

```java
public class SimplePage<T> implements Page<T> {

    private final Page<T> page;

    public SimplePage(
            @JsonProperty("content") List<T> content,
            @JsonProperty("number") int number,
            @JsonProperty("size") int size,
            @JsonProperty("totalElements") long totalElements) {
        page = new PageImpl<>(content, new PageRequest(number, size), totalElements);
    }


    @JsonProperty
    @Override
    public int getTotalPages() {
        return page.getTotalPages();
    }

    @JsonProperty
    @Override
    public long getTotalElements() {
        return page.getTotalElements();
    }

    @JsonProperty("page")
    @Override
    public int getNumber() {
        return page.getNumber();
    }

    @JsonProperty
    @Override
    public int getSize() {
        return page.getSize();
    }

    @JsonProperty
    @Override
    public int getNumberOfElements() {
        return page.getNumberOfElements();
    }

    @JsonProperty
    @Override
    public List<T> getContent() {
        return page.getContent();
    }

    @JsonProperty
    @Override
    public boolean hasContent() {
        return page.hasContent();
    }

    @JsonIgnore
    @Override
    public Sort getSort() {
        return page.getSort();
    }

    @JsonProperty
    @Override
    public boolean isFirst() {
        return page.isFirst();
    }

    @JsonProperty
    @Override
    public boolean isLast() {
        return page.isLast();
    }

    @JsonIgnore
    @Override
    public boolean hasNext() {
        return page.hasNext();
    }

    @JsonIgnore
    @Override
    public boolean hasPrevious() {
        return page.hasPrevious();
    }

    @JsonIgnore
    @Override
    public Pageable nextPageable() {
        return page.nextPageable();
    }

    @JsonIgnore
    @Override
    public Pageable previousPageable() {
        return page.previousPageable();
    }

    @JsonIgnore
    @Override
    public <S> Page<S> map(Converter<? super T, ? extends S> converter) {
        return page.map(converter);
    }

    @JsonIgnore
    @Override
    public Iterator<T> iterator() {
        return page.iterator();
    }
}
```

이렇게 json 응답 값을 받는 CustomPage객체를 만들고 Page객체를 상속받자.
이제 새로 구현해준 CustomPage객체로 감싸서 응답 값을 받으면, 페이징 정보를 받아 올 수있다.

```java
@FeignClient(value = "회원Client", url = "${client.url}")
public interface MemberClient {
    
    @PostMapping("/member")
    SimplePage<MemeberDto> getMemberList(Integer page, Integer size);
}
```