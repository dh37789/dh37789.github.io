---
title:  "[Level1] 자릿수 더하기"

layout: post
categories: algorithmQuestion

toc: true
toc_sticky: true

date: 2022-02-16
last_modified_at: 2022-02-16
---

```java
/**
 * link : https://programmers.co.kr/learn/courses/30/lessons/12931
 */
public class No23_Add_Digits {

  public static void main(String[] args) {
    int n = 256;

    System.out.println(solution(n));
  }

  public static int solution(int n) {
    int answer = 0;

    while (n != 0) {
      answer += n % 10;
      n = n / 10;
    }

    return answer;
  }
}
```
