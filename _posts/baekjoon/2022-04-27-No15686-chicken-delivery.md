---
title: "[Gold5] No.15686 치킨배달"

categories: baekjoon

toc: true
toc_sticky: true

date: 2022-04-27
last_modified_at: 2022-04-27
---

# 치킨 배달

[No.15686 치킨배달](https://www.acmicpc.net/problem/15686)

## 문제

크기가 N×N인 도시가 있다. 도시는 1×1크기의 칸으로 나누어져 있다. 도시의 각 칸은 빈 칸, 치킨집, 집 중 하나이다. 도시의 칸은 (r, c)와 같은 형태로 나타내고, r행 c열 또는 위에서부터 r번째 칸, 왼쪽에서부터 c번째 칸을 의미한다. r과 c는 1부터 시작한다.

이 도시에 사는 사람들은 치킨을 매우 좋아한다. 따라서, 사람들은 "치킨 거리"라는 말을 주로 사용한다. 치킨 거리는 집과 가장 가까운 치킨집 사이의 거리이다. 즉, 치킨 거리는 집을 기준으로 정해지며, 각각의 집은 치킨 거리를 가지고 있다. 도시의 치킨 거리는 모든 집의 치킨 거리의 합이다.

임의의 두 칸 (r1, c1)과 (r2, c2) 사이의 거리는 |r1-r2| + |c1-c2|로 구한다.

예를 들어, 아래와 같은 지도를 갖는 도시를 살펴보자.
```text
0 2 0 1 0
1 0 1 0 0
0 0 0 0 0
0 0 0 1 1
0 0 0 1 2
```
0은 빈 칸, 1은 집, 2는 치킨집이다.

(2, 1)에 있는 집과 (1, 2)에 있는 치킨집과의 거리는 abs(2-1) + abs(1-2) = 2, (5, 5)에 있는 치킨집과의 거리는 abs(2-5) + abs(1-5) = 7이다. 따라서, (2, 1)에 있는 집의 치킨 거리는 2이다.

(5, 4)에 있는 집과 (1, 2)에 있는 치킨집과의 거리는 abs(5-1) + abs(4-2) = 6, (5, 5)에 있는 치킨집과의 거리는 abs(5-5) + abs(4-5) = 1이다. 따라서, (5, 4)에 있는 집의 치킨 거리는 1이다.

이 도시에 있는 치킨집은 모두 같은 프랜차이즈이다. 프렌차이즈 본사에서는 수익을 증가시키기 위해 일부 치킨집을 폐업시키려고 한다. 오랜 연구 끝에 이 도시에서 가장 수익을 많이 낼 수 있는  치킨집의 개수는 최대 M개라는 사실을 알아내었다.

도시에 있는 치킨집 중에서 최대 M개를 고르고, 나머지 치킨집은 모두 폐업시켜야 한다. 어떻게 고르면, 도시의 치킨 거리가 가장 작게 될지 구하는 프로그램을 작성하시오.

## 문제 풀이

해당 문제는 백트래킹을 이용해서 해결 할 수 있다.

> **백트래킹이란?**  
> 
> 가능한 모든 방법을 탐색하지만, DFS와 같이 완전탐색으로 모든 경우의 수를 확인 하는 것이 아닌, 탐색 중 조건에 맞지 않는 루트는 탐색하지않고 다른 루트를 찾아 탐색하는 알고리즘이다.

이번 문제에 대한 백트래킹 알고리즘의 핵심 코드는 아래와 같다.

각각 치킨집과 집을 List 각각 집과 치킨집의 좌표를 담아준다.

```java
List<Node> home = new ArrayList<>();
List<Node> chicken = new ArrayList<>();
```

```java
for (int i = index; i < chickenList.size(); i++) {
    /* 방문 할 치킨집을 선별 */
    if (visit[i] == false) {
        /* 방문할 치킨집으로 정한다. */
        visit[i] = true;
        /* 재귀 호출을 이용해 두번째 치킨집을 정한다. */
        backTracking(count + 1, i + 1);
        /* 재귀가 끝난뒤 방문할 치킨집을 해제 */
        visit[i] = false;
    }
}
```

해당 코드를 예시로 3개의 치킨집 중에서 2개만 남긴다고 가정해보자.

`1. i 가 0일 경우` 

- 해당 반복문을 처음 돌면 첫번 째 치킨집이 `visit[i] = true;`에 의해 방문 치킨집으로 선택 될것이다. 하지만 재귀를 통해 다시 탐색을 하려해도 치킨집은 두개가아닌 하나가 선택되어서 탐색에서 제외된다.

![치킨배달1]({{site.url}}/assets/image/2022-04-27/chicken1.PNG)

- 두번째 재귀문 `backTracking(count + 1, i + 1);`을 통해 다시한번 반복문을 통하면 visit배열은 다음과 같아진다.

![치킨배달2]({{site.url}}/assets/image/2022-04-27/chicken2.PNG)

- 다시한번 `backTracking(count + 1, i + 1);` 재귀 문을 돌면 모든 치킨집이 선택되지만 조건에 맞지 않으므로 탐색하지 않는다. 앞으로 3개가 골라지는 배열은 제외하도록 하겠다.

![치킨배달3]({{site.url}}/assets/image/2022-04-27/chicken3.PNG)

- 해당 메서드에서 빠져나와 다시 재귀함수를 탈 경우 아래와 같이 배열이 채워진다.

![치킨배달4]({{site.url}}/assets/image/2022-04-27/chicken4.PNG)

`2. i 가 1일 경우` 

- i가 1일 경우 다시 재귀함수 타면 아래와 같이 된다. 

![치킨배달5]({{site.url}}/assets/image/2022-04-27/chicken5.PNG)

`3. i 가 2일 경우`

- i가 2일 경우 다시 재귀함수 타면 아래와 같이 된다.

![치킨배달6]({{site.url}}/assets/image/2022-04-27/chicken6.PNG)

만약 위의 조건이 만족한다면, 

```java
if(count == m) {
    int total = 0;
    for (int i = 0; i < homeList.size(); i++) {
        int min = Integer.MAX_VALUE;
        for (int j = 0; j < chickenList.size(); j++) {
            if (visit[j] == true) {
                Node homeNode = homeList.get(i);
                Node chickenNode = chickenList.get(j);
                int road = Math.abs(homeNode.y - chickenNode.y) + Math.abs(homeNode.x - chickenNode.x);
                min = Math.min(min, road);
            }
        }
        total += min;
    }
    sum = Math.min(sum, total);
    return;
}
```

탐색하여 최솟값을 계산해 준다.

## 풀이소스

<script src="https://gist.github.com/dh37789/181a10160f1ea9ea233b1026a49e6d45.js"></script>