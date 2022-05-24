---
title: "[DesignPattern] 팩토리 메서드(Factory Method)"

categories: DesignPattern

toc: true
toc_sticky: DesignPattern

date: 2022-05-24
last_modified_at: 2022-05-24
---

# 팩토리 메서드(Factory Method)

객체를 생성하기 위해 인터페이스를 정의합니다. 어떤 클래스의 인스턴스를 생성할지 서브클래스를 통해서 결정해 줍니다.

## 활용성 

팩토리 메서드는 다음과 같은 상황에 사용합니다.

- 어떤 클래스가 자신이 생성해야 하는 객체의 클래스를 예측할 수 없을 때
- 생성할 객체를 기술하는 책임을 자신의 서브클래스가 지정했으면 할 때
- 객체 생성의 책임을 몇개의 보조 서브클래스 가운데 하나에게 위임하고, 어떤 서브클래스가 위임자인지에 대한 정보를 모아 묶고 싶을 때

## 구조

![팩토리 메서드 구조]({{site.url}}/assets/image/2022-05-24/factory.png)

- Product(Document) : 팩토리 메서드가 생성하는 객체의 인터페이스를 정의합니다.
- ConcreteProduct(MyDocument) : Product 클래스에 정의된 인터페이스를 실제로 구현합니다.
- Creator(Application) : Product 타입의 객체를 반환하는 팩토리 메서드를 구현합니다. Creator 클래스는 팩토리 메서드를 기본적으로 구현하는데, 이 구현에서는 ConcreteProduct 객체를 반환합니다. 또한 Product 객체의 생성을 위해 팩토리 메서드를 호출합니다.
- ConcreteCreator(MyApplication) : 팩토리 메서드를 재정의하여 ConcreteProduct의 인스턴스를 반환합니다.

## 구현

팩터리 메서드 구현에 설계한 디렉토리 구조는 아래와 같습니다.

```
.
├── concretecreator
│   ├── MetalAnimalRobotStore
│   └── WoodAnimalRobotStore
├── concreteproduct
│   ├── MetalCatRobot
│   ├── MetalDogRobot
│   ├── WoodCatRobot
│   └── WoodDogRobot
├── creator
│   └── AnimalRobotStore
├── Product
│   └── AnimalRobot
└── Main
```

### 1. 생성하고자 하는 객체의 인터페이스 정의

만들고자 하는 `AnimalRobot`의 객체를 먼저 정의해 줍니다. 

```java
/**
 * Product(Document)
 * 팩토리 메서드가 생성하는 객체의 인터페이스를 정의합니다.
 */
public abstract class AnimalRobot {
    protected String type;
    protected String cry;
    protected String texture;

    public void makeRobot() {
        System.out.println("make robot type is " + type);
    }

    public void crying() {
        System.out.println(texture + " " + cry);
    }
}
```

### 2. 추상화된 객체를 상속받아 Product을 구현

`AnimalRobot` 객체를 상속받아 해당 객체의 자세한 상세 내역을 구현합니다.  
각각 나무, 금속 재질로 된 동물로봇들을 구현 했습니다. 

```java
/**
 * ConcreteProduct(MyDocument)
 * Product 클래스에 정의된 인터페이스를 실제로 구현합니다.
 */

/* 금속 고양이 로봇 */ 
public class MetalCatRobot extends AnimalRobot {

    public MetalCatRobot() {
        type = "Metal Cat Robot";
        texture = "metal";
        cry = "mew";
    }
}
/* 금속 강아지 로봇 */
public class MetalDogRobot extends AnimalRobot {

    public MetalDogRobot() {
        type = "Metal Dog Robot";
        texture = "metal";
        cry = "bow-wow";
    }
}
/* 나무 고양이 로봇 */
public class WoodCatRobot extends AnimalRobot {

    public WoodCatRobot() {
        type = "Wood Cat Robot";
        texture = "wood";
        cry = "mew";
    }
}
/* 나무 강아지 로봇 */
public class WoodDogRobot extends AnimalRobot {

    public WoodDogRobot() {
        type = "Wood Dog Robot";
        texture = "wood";
        cry = "bow-wow";
    }
}
```

### 3. Product 타입을 반환한는 팩토리 메서드 구현

type을 매개변수로 받아 type 별로 맞는 `AnimalRobot` 객체를 반환하는 팩토리 메서드를 구현합니다.

```java
/**
 * Creator(Application)
 * Product 타입의 객체를 반환하는 팩토리 메서드를 구현합니다.
 * Creator 클래스는 팩토리 메서드를 기본적으로 구현하는데, 이 구현에서는 ConcreteProduct 객체를 반환합니다.
 * 또한 Product 객체의 생성을 위해 팩토리 메서드를 호출합니다.
 */
public abstract class AnimalRobotStore {
    public AnimalRobot makeAnimalRobot(String type) {
        AnimalRobot robot = setRobotType(type);
        robot.makeRobot();
        return robot;
    }

    protected abstract AnimalRobot setRobotType(String type);
}
```

### 4. Creator를 재정의하여 ConcreteProduct의 객체를 반환합니다.

`AnimalRobotStore`을 상속받은 `MetalAnimalRobotStore`, `WoodAnimalRobotStore`를 구현하여,  
각각 나무와 금속의 재질을 가진 `AnimalRobot`의 구체화된 객체를 반환합니다. 

```java
/**
 * ConcreteCreator(MyApplication)
 * 팩토리 메서드를 재정의하여 ConcreteProduct의 인스턴스를 반환합니다.
 */
/* 팩토리 메서드 재정의를 통한 금속 로봇 생성 */
public class MetalAnimalRobotStore extends AnimalRobotStore {

    @Override
    protected AnimalRobot setRobotType(String type) {
        if ("cat".equals(type)) {
            return new MetalCatRobot();
        } else if ("dog".equals(type)) {
            return new MetalDogRobot();
        } else {
            throw new IllegalArgumentException();
        }
    }
}
/* 팩토리 메서드 재정의를 통한 나무 로봇 생성 */
public class WoodAnimalRobotStore extends AnimalRobotStore {

    @Override
    protected AnimalRobot setRobotType(String type) {
        if ("cat".equals(type)) {
            return new WoodCatRobot();
        } else if ("dog".equals(type)) {
            return new WoodDogRobot();
        } else {
            throw new IllegalArgumentException();
        }
    }
}
```

### 5. 생성한 객체 출력

```java
public class Main {
    public static void main(String[] args) {
        AnimalRobotStore woodStore = new WoodAnimalRobotStore();
        AnimalRobotStore metalStore = new MetalAnimalRobotStore();

        AnimalRobot woodCatRobot = woodStore.makeAnimalRobot("cat");
        AnimalRobot woodDogRobot = woodStore.makeAnimalRobot("dog");

        AnimalRobot metalCatRobot = metalStore.makeAnimalRobot("cat");
        AnimalRobot metalDogRobot = metalStore.makeAnimalRobot("dog");

        woodCatRobot.crying();
        woodDogRobot.crying();
        metalCatRobot.crying();
        metalDogRobot.crying();
    }
}
```

**출력된 결과**

```shell
make robot type is Wood Cat Robot
make robot type is Wood Dog Robot
make robot type is Metal Cat Robot
make robot type is Metal Dog Robot
wood mew
wood bow-wow
metal mew
metal bow-wow
```

## 예제 코드

[팩터리 메서드 Github 전체 소스](https://github.com/dh37789/design-pattern/tree/main/src/com/design/pattern/No03FactoryMethod)