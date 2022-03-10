---
title:  "[Level1] 자연수 뒤집어 배열 만들기"

categories: programmers

toc: true
toc_sticky: true

date: 2022-02-16
last_modified_at: 2022-02-16
---

```java
import java.util.Arrays;

/**
 * link : https://programmers.co.kr/learn/courses/30/lessons/12932
 */
public class No24_Reverse_Natural_Numbers_To_Create_An_Array {

  public static void main(String[] args) {
    long n = 12345L;

    System.out.println(Arrays.toString(solution(n)));
  }

  public static int[] solution(long n) {

    StringBuffer sb = new StringBuffer(String.valueOf(n)).reverse();

    int[] answer = new int[sb.length()];
    for (int i = 0; i < sb.length(); i++) {
      answer[i] = Integer.parseInt(sb.charAt(i) + "");
    }

    return answer;
  }
}
```