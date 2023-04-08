---
title:  "[Java] 의존성_우아한테크세미나-190620-우아한객체지향 (1)"

categories: Java

toc: true
toc_sticky: true

date: 2022-02-19
last_modified_at: 2022-02-21
---

# 의존성_우아한테크세미나-190620-우아한객체지향 (1)

## 의존성

![의존성]({{site.url}}/assets/image/2022/2022-02-21/dependency_a.PNG)

- A가 B를 의존할 경우 B가 변경 될 경우 A도 변경 될 수 있다.

## 클래스 의존성의 종류

- 연관관계(Association)

A라는 클래스에 B라는 객체를 잠조하고있다.

```java
class A {
    private B b;
}
```

- 의존관계(Dependency)

A라는 클래스에 B 객체의 파라미터, 리턴타입 또는 인스턴스를 생성한다.

```java
class A {
    private B method (B b) {
        return new B();
    };
}
```

- 상속관계(Inheritance)

B의 객체가 변경 될 경우 A도 같이 변경이 된다.

```java
class A extends B{
    
}
```

- 실체화관계(Realization)

B객체의 구현체를 A객체에서 구현한다.

```java
class A implements B {
    
}
```

*상속관계와 실체화관계의 차이는 상속관계는 부모 객체가 변경될시 자식 객체가 같이 변경되지만  
실체화 관계는 부모 객체의 코드가 변경 될 경우 자식 객체가 변경된다.

## 패키지 의존성

패키지에 포함된 클래스 사이의 의존성
간단하게 설명하자면 클래스 내에 타 패키지를 import한 경우라고 볼 수 있다.

### 패키지 의존성 규칙

1. 양방향 의존성을 피하라
2. 다중성이 적은쪽으로 설계
3. 의존성이 필요없으면 제거
4. 패키지 사이의 의존성 사이클을 제거 해야한다.

## Example

### ex) 관계에는 방향성이 필요하다.

![exam1]({{site.url}}/assets/image/2022/2022-02-21/example_2022_02_21_001.PNG )

### 관계의 방향 = 협력의 방향 = 의존성의 방향

![exam2]({{site.url}}/assets/image/2022/2022-02-21/example_2022_02_21_002.PNG )

### 관계의 종류 결정하기 

- 연관관계 : 협력을 위해 필요한 **영구적인** 탐색구조 (객체참조)

![exam3]({{site.url}}/assets/image/2022/2022-02-21/example_2022_02_21_003.PNG )

- 의존관계 : 협력을 위해 **일시적으로** 필요한 의존성 (파라미터, 리턴타입, 자연변수)

![exam4]({{site.url}}/assets/image/2022/2022-02-21/example_2022_02_21_004.PNG )

## 연관관계 = 탐색가능성(navigability)

![연관관계1]({{site.url}}/assets/image/2022/2022-02-21/dependency_b.PNG )

- Order에서 OrderLineItem으로 탐색 가능
- 즉, Order가 뭔지 알면 Order를 통해 원하는 OrderLineItem을 찾을 수 있다.

![연관관계2]({{site.url}}/assets/image/2022/2022-02-21/dependency_c.PNG )

- 두 객체 사이에 협력이 필요하고 두 객체의 관계가 영구적이라면 연관관계를 이용해 탐색 경로 구현

```java
class Order {
    private List<OrderLineItem> orderLineItem;
    
    public void place(){
        validate();
        ordered();
    }
    
    private void validate(){
      ...
      // 연관관계를 이용해서 협력
      for(OrderLineItem orderLineItem : orderLineItem) {
          orderLineItem.validate();
      }
    }
}
```

## 해당 예제 구현하기

![exam5]({{site.url}}/assets/image/2022/2022-02-21/example_2022_02_21_005.PNG )


1. Shop & OrderLineItem의 연관관계를 연결한다.

```java
@Entity
@Table(naem="ORDERS")
public class Order {
    @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    @Column(name="ORDER_ID")
    private Long id;

    // Shop(상점정보)의 객체 참조
    @ManyToOne
    @JoinColumn(name="SHOP_ID")
    private Shop shop;

    // OrderLineItem(주문정보)의 객체 참조
    @OneToMany(cascade = CascadeType.ALL)
    @JoinColumn(name="ORDER_ID")
    private List<OrderLineItem> orderLineItems = new ArrayList<>();
    
    ...
  
    public void place() {
        validate();
        ordered();
    }
    
    private void validate() {
    }
    private void ordered() {
    }
}
```

2. 가게가 영업중인지 확인 & 주문금액이 최소주문금액이상인지

- (Order -> Shop)
  1. isOpen();
  2. isValidAmount();
  
Order
```java
public class Order {
    public void place() {
        validate();
        ordered();
    }
    
    private void validate() {
        if (orderLineItem.isEmpty()) {
            throw new IllegalStateException("주문 항목이 비어 있습니다.");    
        }
        if (!shop.isOpen()) {
            throw new IllegalStateException("가게가 영업중이 아닙니다.");
        }
        if (!shop.isValidOrderAmount(calculateTotalPrice())) {
             throw new IllegalStateException(String.format("최소 주문 금액 %s 이상을 주문해주세요.", shop.getMinOrderAmount()));
        }
        for (OrderLineItem orderLineItem : orderLineItems) {
            orderLineItem.validate();
        }
    }
    private void ordered() {
      this.orderStatus = OrderStatus.ORDERED;
    }
}
```

Shop
```java
public class Shop {
    public boolean isOpen(){
        return this.open;
    }
    
    public boolean isValidOrderAmount(Money amount) {
        return amount.isGreaterThanOrEqual(minOrderAmount);
    }
}
```

3. 주문중에 서버의 주문정보가 바뀌었을 수 있으니 재검증

- (Order -> OrderLineItem)
  1. validate();
- (OrderLineItem -> Menu)
  1. validateOrder(name, orderLineItem);

OrderLineItem
```java
public class OrderLineItem {
    public void validate() {
        menu.validateOrder(name, this.orderOptionGroups);
    }
}
```

4. 메뉴의 이름과 주문항목의 이름 비교

- (Menu -> OptionGroup Specification)
  1. isSatisfiedBy(oog);
- (OptionGroup Specification -> Order OptionGrop)
  1. getOption();

Menu
```java
public class Menu {
    public void validateOrder(String menuName, List<OptionGroup> optionGroups) {
        if (!this.name.equals(menuName)) {
            throw new IllegalArgumentException("기본 상품이 변경됐습니다.");
        }

        if (!isSatisfiedBy(optionGroups)) {
            throw new IllegalArgumentException("메뉴가 변경됐습니다.");
        }
    }

    private boolean isSatisfiedBy(List<OptionGroup> cartOptionGroups) {
        return cartOptionGroups.stream().anyMatch(this::isSatisfiedBy);
    }

    private boolean isSatisfiedBy(OptionGroup group) {
        return optionGroupSpecs.stream().anyMatch(spec -> spec.isSatisfiedBy(group));
    }
}
```

5. 옵션그룹의 이름과 주문옵션그룹의 이름 비교

- (OptionGroup Specification -> Option Specification)
  1. isSatisfiedBy(oOption);

OptionGroupSpecification
```java
public class OptionGroupSpecification {
  public boolean isSatisfiedBy(OptionGroup optionGroup) {
    return !isSatisfied(optionGroup.getName(), satisfied(optionGroup.getOptions()));
  }

  private boolean isSatisfied(String groupName, List<Option> satisfied) {
    if (!name.equals(groupName) ||
            satisfied.isEmpty() ||
        (exclusive && satisfied.size() > 1)) {
      return false;
    }

    return true;
  }

  private List<Option> satisfied(List<Option> options) {
    return optionSpecs
            .stream()
            .flatMap(spec -> options.stream().filter(spec::isSatisfiedBy))
            .collect(toList());
  }
}
```

6. 옵션의 이름과 주문옵션의 이름 비교 & 옵션의 가격과 주문옵션의 가격 비교

- (OptionGroup Specification -> Option Specification)
  1. getName();
  2. getPrice();

OptionSpecification
```java
public class OptionSpecification {
    public boolean isSatisfiedBy(Option option) {
        return Objects.equals(name, option.getName()) &&
                Objects.equals(price, option.getPrice());
    }
}
```

## 레이어 아키텍처 (Layered Architecture)

위의 해당 예제는 레이어 아키텍처 중 **Domain의 과정**에 들어가는 과정이다.  
영역의 관계를 정의하는 단계라고 보면 된다.  

![rayer1]({{site.url}}/assets/image/2022/2022-03-02/rayer001.PNG )

아래의 소스와 같이 Domain과정에서 벗어난 request를 받거나 DB에 접근하는 로직의 구현은 Service 레이어에서 진행한다.

![rayer2]({{site.url}}/assets/image/2022/2022-03-02/rayer002.PNG )



