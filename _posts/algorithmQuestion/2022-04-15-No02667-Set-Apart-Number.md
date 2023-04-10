---
title:  "[Silver1] No.02667 단지번호붙이기"

layout: post
categories: algorithmQuestion

toc: true
toc_sticky: true

date: 2022-04-15
last_modified_at: 2022-04-15
---

# 단지번호붙이기

[No.02667 단지번호붙이기](https://www.acmicpc.net/problem/2667)

## 문제

<그림 1>과 같이 정사각형 모양의 지도가 있다. 1은 집이 있는 곳을, 0은 집이 없는 곳을 나타낸다. 철수는 이 지도를 가지고 연결된 집의 모임인 단지를 정의하고, 단지에 번호를 붙이려 한다. 여기서 연결되었다는 것은 어떤 집이 좌우, 혹은 아래위로 다른 집이 있는 경우를 말한다. 대각선상에 집이 있는 경우는 연결된 것이 아니다. <그림 2>는 <그림 1>을 단지별로 번호를 붙인 것이다. 지도를 입력하여 단지수를 출력하고, 각 단지에 속하는 집의 수를 오름차순으로 정렬하여 출력하는 프로그램을 작성하시오.

![단지번호붙이기]({{site.url}}/public/image/2022/2022-04-15/no02667.png)

## 문제풀이

해당 문제는 BFS나 DFS를 이용해 푸는 문제이다.

살짝 응용이 된 요소라면, 좌표 관련된 요소인데 해당방법을 이용하면 쉽게 구할 수 있다.

```java
static int[] dirX = {0, 0, -1, 1};
static int[] dirY = {-1, 1, 0, 0};

int x = 1;
int y = 1;

for (int i = 0; i < 4; i++) {
    int nx = x + dirX[i];
    int ny = y + dirY[i];
}
```

X와 Y의 좌표에서 위 아래 옆을 for문을 통해 더해주면서 상하좌우 좌표의 값 및 현황을 알 수 있다.

해당 좌표 탐색을 bfs와 합치면 아래와 같다.

```java
public static void bfs (int y, int x) {
        Queue<Node> queue = new LinkedList<>();
        queue.offer(new Node(y, x));
        count = 1;
        visited[y][x] = true;
        while(!queue.isEmpty()) {
            Node node = queue.poll();
            for (int i = 0; i < 4; i++) {
                int ny = node.y + dirY[i];
                int nx = node.x + dirX[i];
                if (ny < 0 || nx < 0 || ny >= n || nx >= n ||visited[ny][nx] || apart[ny][nx] == 0)
                    continue;
                queue.offer(new Node(ny, nx));
                visited[ny][nx] = true;
                count++;
            }
        }
    }
```

## 풀이 소스

<script src="https://gist.github.com/dh37789/10d520deb978dfc7ee8c5e016845d957.js"></script>
