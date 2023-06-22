---
title:  "[Java] Spring 직렬화(serialize)/역직렬화(deserialize) 이야기"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2023-06-19
last_modified_at: 2023-06-19
---

자바의 역직렬화(deserialize) 이야기에 앞서 직렬화(serialize)와 역직렬화(deserialize)를 간단하게 톺아보고 가도록 하자.


## 직렬화 (serialize)

직렬화란 자바의 객체(Object)의 상태를 바이트 형태의 데이터 스트림으로 변환하는 것을 말한다.

데이터 스트림은 흔히 우리가 자주 쓰는 `json` 형식이 될 수 있고 또는 `xml` 과 같은 다양한 형태로 변환이 가능하다.


## 역직렬화 (deserialize)

역직렬화란 직렬화와는 반대로 바이트 형태의 데이터 스트림을 자바의 객체(Object)의 상태로 변환해주는 것을 말한다.

![직/역]({{site.url}}/public/image/2023/2023-06/23-deserilize001.png)


이제 간단하게 개념을 살펴봤으니, 알아보고자 하는 역직렬화에 대해 살펴보고자 한다.

자바에서는 어떻게 JVM의 메모리상 남아있는 Object 데이터를 데이터 스트림으로 서로 변환을 시킬 수 있을까?<br>
또 어떻게 자바에서는 데이터 스트림에서 자바에 선언한 Object의 타입을 찾아서 인스턴스를 넣어줄 수 있는걸까?


## Jackson

자바에서는 대표적인 데이터 처리 라이브러리로 Jackson 라이브러리를 사용하고 있다.

[Jackson 라이브러리의 Github 페이지](https://github.com/FasterXML/jackson)에서 살펴보는 Jackson의 정의는 다음과 같다.

> **What is Jackson?**<br>
> Jackson has been known as "the Java JSON library" or "the best JSON parser for Java". Or simply as "JSON for Java".<br>
> More than that, Jackson is a suite of data-processing tools for Java (and the JVM platform), including the flagship streaming JSON parser / generator library,...

직역해보자면, Json parse의 라이브러리로 알려져 있는 Jackson의 라이브러리는 더 나아가 Java(및 JVM 플랫폼)의 데이터 처리 도구의 모음이다. 라는 설명이다.

실제로 Spring에서는 Jackson 라이브러리에 자체적으로 추가 기능을 더해 공식적으로 지원하고 있다.

주요 클래스는 `ObjectMapper`라는 클래스로 해당 클래스에서 데이터의 변환을 해주고 있다.


## 역직렬화는 어떻게 진행될까?








https://docs.oracle.com/javase/tutorial/jndi/objects/serial.html
