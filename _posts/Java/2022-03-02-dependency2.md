---
title:  "[Java] 의존성_우아한테크세미나-190620-우아한객체지향 (2)"

categories: Java

toc: true
toc_sticky: true

date: 2022-03-02
last_modified_at: 2022-03-02
---

# 의존성_우아한테크세미나-190620-우아한객체지향 (2)

## 설계 개선하기

설계에서 중요한것은 각 객체의 의존성(dependency)를 살펴봐라

**설계를 진화시키기 위한 출발점은 처음부터 완벽 할 수 없다.**  
코드 작성 후 의존성 관점에서 점차적으로 설계를 검토하며 개선하자

해당 강의에서는 두가지의 문제와 그에 대한 해결법에 대해 강의하고자 한다.

- 객체 참조로 인한 결합도 상승

![문제1]({{site.url}}/assets/image/2022-03-02/exam001.PNG)

- 패키지 의존성 사이클

![문제2]({{site.url}}/assets/image/2022-03-02/exam002.PNG)

## 의존성 살펴보기

기존 강의 (1) 항목의 예제를 레이어로 도식화하면 아래와 같이 나타난다.

![문제3]({{site.url}}/assets/image/2022-03-02/exam003.PNG)

해당 문제점을 자세히 살펴보면

![문제4]({{site.url}}/assets/image/2022-03-02/exam004.PNG)

shop의 객체와 order의 객체가 사이클이 도는것을 확인 할 수 있다.

## 무엇이 문제인가?

![문제5]({{site.url}}/assets/image/2022-03-02/exam005.PNG)

1. Order 객체 -> Shop 객체 
  - Order -> Shop
  - 주문이 가게가 영업중인지, 최소금액을 만족하는지 판단하기 위해서  
    Order -> Shop 방향에 대한 의존성이 필요하다.  
```java
Class Order {
    private void validate() {
        if(!shop.isOpen()) {...}    
    }    
}
```

2. Shop 객체 -> Order 객체
  - OptionGroup Specification -> Order OptionGroup, Option Specification -> OrderOption 
  - 이름이나 가격을 가져오기위해 주문에서 해당 데이터를 가져온다.
```java
Class OptionGroupSpecification {
    private boolean isSatisfiedBy(OrderOptionGroup group) {
        ...
    }    
}
```
```java
Class OptionSpecification {
    private boolean isSatisfiedBy(OrderOption option) {
        ...
    }
}
```

이렇게 해당 객체는 의존성 사이클이 돌고 있다.

## 어떻게 해결해야 하는가?

1. 중간 객체를 이용해 의존성 사이클 끊기

![문제6]({{site.url}}/assets/image/2022-03-02/exam006.PNG)

OptionGroup, Option라는 객체를 만들어 OrderOptionGroup와 OrderOption을 각각 변환 해주면 방향이 한뱡항으로 흐룰 수 있게 된다. 

아래의 변환 소스를 생성해준다면

```java
class OrderOptionGroup{
    public OptionGroup convertToOptionGroup(){
        return new OptionGroup(name,...);
    }
}
```

```java
class OrderOption{
    public OptionGroup convertToOption(){
        return new Option(name,...);
    }
}
```

위에 작성한 OptionGroupSpecification, OptionSpecification클래스의 소스를 아래와 같이 변환해서 의존성 사이클을 해결 할 수 있다.

```java
Class OptionGroupSpecification {
    private boolean isSatisfiedBy(OptionGroup group) {
        ...
    }    
}
```

```java
Class OptionSpecification {
    private boolean isSatisfiedBy(Option option) {
        ...
    }
}
```

