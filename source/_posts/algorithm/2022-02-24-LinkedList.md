---
title:  "[Theory] LinkedList"

categories: algorithm

toc: true
toc_sticky: true

date: 2022-02-24
last_modified_at: 2022-02-28
---

# 연결리스트(LinkedList)

![연결리스트](/assets/image/2022/2022-02-25/linkedList001.PNG)
연결 리스트(Linked List)란 각 노드가 데이터와 포인터를 가지고 한 줄로 연결되어 있는 방식으로 데이터를 저장하는 자료 구조를 말합니다.  
이름과 같이 각 데이터를 담은 노드들이 연결되어 있는데, 노드의 포인터가 다음이나 이전의 노드의 연결을 담당하고 있습니다.  

연결리스트의 종류로는 단일 연결 리스트, 이중 연결 리스트 등이 있습니다.

## 연결리스트 구현

LinkedListNode<T> : LinkedList의 노드객체입니다. link 변수는 해당 노드(Vertex)에 연결된 다른 노드의 주소를 가지고있습니다. 

<script src="https://gist.github.com/dh37789/5a5ef51dad331cfaa1ae32987d2ba003.js"></script>

LinkedList : Java를 통해 구현한 연결리스트의 구현부입니다. 구현한 연산함수는 아래와 같습니다.

- addFirst(data) : 첫번째 위치의 head에 data를 삽입한다.
- addLast(data) : 마지막 위치인 tail에 data를 삽입한다.
- add(index, data) :  index위치에 data를 삽입한다.
- removeFirst() : 첫번째 위치의 head부분의 데이터를 Object객체로 반환하고 삭제한다.
- removeLast() : 마지막 위치의 tail부분의 데이터를 Object객체로 반환하고 삭제한다.
- remove(index) : index위치의 data를 Object객체로 반환하고 삭제한다.
- clear() : 현재 LinkedList의 모든 데이터를 삭제한다.
- size() : 현재 LinkedList의 크기를 반환한다.
- isEmpty() : 현재 LinkedList의 크기가 비어있는지 확인한다.
- toString() : 현재 LinkedList의 data항목을 String객체로 반환한다.

<script src="https://gist.github.com/dh37789/a793706e176219e86c21cd5a11b8006d.js"></script>


