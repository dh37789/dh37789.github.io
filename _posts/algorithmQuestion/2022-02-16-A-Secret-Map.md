---
title:  "[Level1] 비밀지도"

layout: post
categories: algorithmQuestion

toc: true
toc_sticky: true

date: 2022-02-16
last_modified_at: 2022-02-16
---

```java
import java.util.Arrays;

/**
 * link : https://programmers.co.kr/learn/courses/30/lessons/17681
 */
public class No26_A_Secret_Map {

  public static void main(String[] args) {
    int n = 5;
    int[] arr1 = {9, 20, 28, 18, 11};
    int[] arr2 = {30, 1, 21, 17, 28};

    System.out.println(Arrays.toString(solution(n, arr1, arr2)));
  }

  public static String[] solution(int n, int[] arr1, int[] arr2) {
    String[] answer = new String[n];
    for (int i = 0; i < n; i++) {
      String str = zeroFill(Integer.toBinaryString(arr1[i] | arr2[i]), n)
          .replace('0', ' ').replace('1', '#');
      answer[i] = str;
    }
    return answer;
  }

  private static String zeroFill(String bin, int n) {
    while (bin.length() < n) {
      bin = "0" + bin;
    }
    return bin;
  }
}
```
