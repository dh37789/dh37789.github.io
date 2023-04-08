---
title:  "[Level1] 하샤드 수"

categories: algorithmQuestion

toc: true
toc_sticky: true

date: 2022-02-16
last_modified_at: 2022-02-16
---

```java
/**
 * link : https://programmers.co.kr/learn/courses/30/lessons/12947
 */
public class No19_Harshad_Number {

  public static void main(String[] args) {
    int x = 10;

    System.out.println(solution(x));
  }

  public static boolean solution(int x) {
    String num = String.valueOf(x);
    int tmp = 0;
    for (int i = 0; i < num.length(); i++) {
      tmp += Integer.parseInt(num.charAt(i) + "");
    }
    return x % tmp == 0;
  }
}
```