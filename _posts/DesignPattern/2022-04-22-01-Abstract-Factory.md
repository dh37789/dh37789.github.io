---
title: "[DesignPattern] 추상 팩토리(ABSTRACT FACTORY)"

categories: DesignPattern

toc: true
toc_sticky: DesignPattern

date: 2022-04-22
last_modified_at: 2022-04-22
---

# 추상 팩토리(ABSTRACT FACTORY)


추상 팩토리 패턴은 서브클래스를 상세히 구현 하지 않아도, 서로 연관성이 있는 **여러 객체의 집합(군)** 을 생성하기 위해 인터페이스를 제공하는 구조를 말합니다.

## 활용성

추상팩토리는 다음의 경우 사용합니다.

- 객체가 생성되거나 구성, 표현 되는 방식과 무관하게 시스팀에 독립적으로 만들고자 할때
- 여러 제품중 하나를 선택에서 구현해야 하고, 한번 구성한 제품을 다른것으로 변경 할 수 있을때
- 관련된 제품 객체들이 함께 사용되도록 설계되었을때
- 제품에 대한 클래스 라이브러리를 제공하고, 구현이 아닌 인터페이스를 노출시키고자 할 때

## 구조

![추상팩토리 구조]({{site.url}}/assets/image/2022/2022-04-22/abstract.png)

- AbstractFactory : 제품에 대한 객체를 생성하는 인터페이스를 정의합니다.
- ConcreteFactory : 구체적인 제품에 대한 객체를 생성하는 구현을 담당합니다.
- AbstractProduct : 개념적 제품 객체에 대한 인터페이스를 정의합니다.
- ConcreteProduct : 구체적으로 팩토리가 생성할 객체를 정의하고 인터페이스를 구현합니다.
- Client : AbstractFactory와, AbstractProduct 클래스에 선언된 인터페이스를 사용합니다.

## 결과

1. 인터페이스를 상속받아 구현하는 클래스의 상세 객체를 분리할 수 있습니다.
2. 하나의 추상 팩토리를 상속받으므로 구현된 객체을 쉽게 대체할 수 있습니다.
3. 객체 사이의 일관성을 증가 시킬 수 있습니다.
4. 추상팩토리의 인터페이스를 상속받아 새로운 종류의 제품을 제공하는 것은 어렵습니다.

## 구현

추삭 팩토리 구현에 설계한 디렉토리 구조는 아래와 같습니다.

```
.
├── abstractFactory
│   ├── AbstractFactory
│   ├── Notebook
│   └── Television
├── lg
│   ├── LgFactory
│   ├── Gram
│   └── LgTelevision
├── samsung
│   ├── SamsungFactory
│   ├── Ion
│   └── SamsungTelevision
└── Client
```

### 1. 하나의 추상팩토리를 정의합니다.

제품에 대한 추상 팩토리 인터페이스를 정의합니다.  
여기서는 노트북과, 텔레비젼에 대한 제품을 공통적인 집합 객체로 묶었습니다.

- 냉장고와 노트북 제품에 대한 추상팩토리 정의

```java
/**
 * AbstractFactory
 * 개념적 제품에 대한 객체를 생성하는 연산으로 인터페이스를 정의합니다.
 */
public interface AbstractFactory {
    Notebook createNotebook();
    Television createTelevision();
}

/**
 * AbstractProductA
 * 개념적 A제품 객체에 대한 인터페이스를 정의합니다.
 */
public interface Notebook {
    String getName();
}

/**
 * AbstractProductB
 * 개념적 B제품 객체에 대한 인터페이스를 정의합니다.
 */
public interface Television {
    String getName();
}

```

### 2. 제품을 생성합니다.

AbstractFactory, Notebook, Television 인터페이스를 상속받아 브랜드별 제품객체를 구현합니다. 

- LG 제품 구현체

```java
/**
 * ConcreteFactory1
 * 구체적인 제품에 대한 객체를 생성하는 연산을 구현합니다.
 */
public class LgFactory implements AbstractFactory {

    @Override
    public Notebook createNotebook() {
        return new Gram();
    }

    @Override
    public Television createTelevision() {
        return new LgTelevision();
    }
}

/**
 * ProductA1
 * 개념적 A1제품 객체에 대한 인터페이스를 구현합니다.
 */
public class Gram implements Notebook {
    @Override
    public String getName() {
        return "LG GRAM";
    }
}

/**
 * ProductB1
 * 개념적 B1제품 객체에 대한 인터페이스를 구현합니다.
 */
public class LgTelevision implements Television {
    @Override
    public String getName() {
        return "LG TELEVISION";
    }
}
```

- SAMSUNG 제품 구현체

```java
/**
 * ConcreteFactory2
 * 구체적인 제품에 대한 객체를 생성하는 연산을 구현합니다.
 */
public class SamsungFactory implements AbstractFactory {

    @Override
    public Notebook createNotebook() {
        return new Ion();
    }

    @Override
    public Television createTelevision() {
        return new SamsungTelevision();
    }
}

/**
 * ProductA2
 * 개념적 A2제품 객체에 대한 인터페이스를 구현합니다.
 */
public class Ion implements Notebook {
    @Override
    public String getName() {
        return "SAMSUNG ION";
    }
}

/**
 * ProductB2
 * 개념적 B2제품 객체에 대한 인터페이스를 구현합니다.
 */
public class SamsungTelevision implements Television {

    @Override
    public String getName() {
        return "SAMSUNG TELEVISION";
    }
}
```

### 3. 정의한 팩토리들을 출력합니다.

AbstractFactory 인터페이스의 구현체에 대한 인스턴스를 각각 LG, SAMSUNG의 String 파라미터를 받아 각기 다른 브랜드를 인스턴스화 합니다. 

```java
public class Client {
    public static void main(String[] args) {
        /* LG */
        AbstractFactory lgfactory = getInstance("LG");

        Notebook lgNotebook = lgfactory.createNotebook();
        Television lgTelevision = lgfactory.createTelevision();

        System.out.println(lgNotebook.getName());
        System.out.println(lgTelevision.getName());

        /* SAMSUNG */
        AbstractFactory factory = getInstance("SAMSUNG");

        Notebook samsungNotebook = factory.createNotebook();
        Television samsungTelevision = factory.createTelevision();

        System.out.println(samsungNotebook.getName());
        System.out.println(samsungTelevision.getName());
    }

    static AbstractFactory getInstance(String brand) {
        AbstractFactory factory = null;

        if ("LG".equals(brand)) {
            factory = new LgFactory();
        } else if ("SAMSUNG".equals(brand)) {
            factory = new SamsungFactory();
        }

        return factory;
    }
}
```

**출력된 결과**

```shell
LG GRAM
LG TELEVISION
SAMSUNG ION
SAMSUNG TELEVISION
```

## 정리

간단하게 정리하자면 추상 팩토리 패턴은 **관련된 객체들의 집합(군)을 형성** 할때 사용하는 디자인 패턴으로 객체들의 집합을 추상화하고 클라이언트는 추상화한 추상 팩토리 인터페이스를 제공받는다.  

제공받은 추상 팩토리 인터페이스에 구현체를 달리해 일관되게 객체를 생성 할 수 있게된다.

## 예제 코드

[추상팩토리 Github 전체 소스](https://github.com/dh37789/design-pattern/tree/main/src/com/design/pattern/No01AbstractFactory)


