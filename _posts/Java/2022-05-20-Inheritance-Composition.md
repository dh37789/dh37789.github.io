---
title:  "[Java] 상속 (Is-A)과 컴포지션 (Has-A)"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2022-05-20
last_modified_at: 2022-05-20
---

# 상속 (IS-A)과 컴포지션 (Has-A)

이펙티브 자바 item18에 들어가기 앞서, 상속과 컴포지션의 각각의 관계에 대한 개념을 정리하고자 한다.
상속과 컴포지션은 얼핏보면 비슷하지만, 관계에 따른 차이가 있습니다.

## 상속

**IS-A 관계 (상속) : 하위 클래스가 상위 클래스의 특성을 재정의 한 것**

- A is a B : A는 B이다로 표현이 가능한 관계를 말한다.
- EX) 손은 사람에게 포함된다, 다리는 사람에게 포함된다.

```java
/**
 * 부모 상속 객체
 */
public class Animal {
    public String stringToAnimal() {
        return "나는 동물입니다.";
    }
}

/**
 * 자식 상속 객체
 */
public class Dog extends Animal {
   public String stringToDog() {
      return "나는 개입니다. " + stringToAnimal();
   }
}
```

```shell
나는 개입니다. 나는 동물입니다.
```

## 컴포지션

**Has-A 관계 (포함) : 기존 클래스가 새로운 클래스의 구성요소가 되는 것**

- A has a B : A가 B를 포함한다로 표현 가능한 관계를 말한다.
- EX) 개는 동물이다, 소는 동물이다.

```java
/**
 * 부모 컴포지션 객체
 */
public class Human {
    Arm arm = new Arm();

    public String  stringToHuman() {
        return "나는 사람입니다. " + arm.stringToArm();
    }
}

/**
 * 자식 컴포지션 객체
 */
public class Arm {
   public String stringToArm() {
      return "나는 팔을 가졌습니다.";
   }
}

```

```shell
나는 사람입니다. 나는 팔을 가졌습니다.
```
