---
title:  "[Silver4] No.01920 수찾기"

layout: post
categories: algorithmQuestion

toc: true
toc_sticky: true

date: 2022-03-15
last_modified_at: 2022-03-15
---

# 수찾기

[No.01920 수찾기](https://www.acmicpc.net/problem/1920)

## 문제

N개의 정수 A[1], A[2], …, A[N]이 주어져 있을 때, 이 안에 X라는 정수가 존재하는지 알아내는 프로그램을 작성하시오.

## 풀이

처음엔 얼마전 이진탐색트리를 구현한 기념으로 수찾기를 이진탐색으로 삽입과, 조회만 구현해서 풀었는데 시간초과가 났다 ㅠㅠ.

### 이진탐색트리로 구현 할 경우 왜 안되었을까?

![편향트리1]({{site.url}}/public/image/2022/2022-03-16/leftTree001.PNG)

사진과 같이 이진탐색트리에서 편향 트리의 경우에는 1을 찾기 위해서는 17부터 1까지 모든 노드의 좌우를 검색하며 가야하므로 시간이 많이 소모된다.

얼마전 정리했지만 이진탐색트리의 경우
- 균등트리 일 경우 O(logN)
- 편향트리 일 경우 O(N)

을 가지므로, 해당 문제에서는 시간초과가 발생한 걸로 추측된다. 그러므로 이진탐색을 이용해서 풀어야 하는데, 이진탐색의 경우 시간복잡도는 **O(logN)**으로 볼 수 있다.

그러므로 올바른 풀이법은
1. 처음으로 받은 배열을 정렬한다.
2. 첫번째로 받은 배열에서 두번째로 받은 값들을 비교해서 값의 유무를 확인하다.

로 볼 수 있다.

다음엔 퀵정렬을 구현해봐야겠다.

## 풀이 소스

<script src="https://gist.github.com/dh37789/879caa099d594fa88670bb4136a508f8.js"></script>

