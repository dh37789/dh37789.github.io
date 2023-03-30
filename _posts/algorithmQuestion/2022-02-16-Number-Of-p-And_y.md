---
title:  "[Level1] 문자열내의 p와 y의 개수"

categories: algorithmQuestion

toc: true
toc_sticky: true

date: 2022-02-16
last_modified_at: 2022-02-16
---

```java
/**
 * link : https://programmers.co.kr/learn/courses/30/lessons/12916
 */
public class No11_Number_Of_p_And_y {
  public static void main(String[] args) {
    String s = "pPoooyYy";

    System.out.println(solution(s));
  }
  public static boolean solution(String s) {

    int num[] = {0, 0};
    s = s.toLowerCase();

    for (int i = 0; i < s.length(); i++) {
      if (s.charAt(i) == 'p'){
        num[0]++;
      } else if (s.charAt(i) == 'y') {
        num[1]++;
      }
    }

    return num[0] == num[1] ? true : false;
  }
}
```