---
title:  "[Level1] 수박수박수박수박수?"

layout: post
categories: algorithmQuestion

toc: true
toc_sticky: true

date: 2022-02-16
last_modified_at: 2022-02-16
---

```java
/**
 * link : https://programmers.co.kr/learn/courses/30/lessons/12922
 */
public class No06_SuBakSuBakSuBakSu {

  public static void main(String[] args) {
    int n = 10;

    System.out.println(solution(n));
  }

  public static String solution(int n) {
    String answer = "";
    for(int i = 0; i < n; i++){
      if(i%2 == 0){
        answer = answer + "수";
      }else if(i%2 == 1){
        answer = answer + "박";
      }
    }
    return answer;
  }
}
```
