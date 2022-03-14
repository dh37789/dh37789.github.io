---
title:  "[Theory] Binary Search Tree"

categories: algorithm

toc: true
toc_sticky: true

date: 2022-03-11
last_modified_at: 2022-03-11
---

# 이진탐색트리 (Binary Search Tree)

![이진탐색트리1]({{site.url}}/assets/image/2022-03-11/bst001.PNG)

이진탐색트리란 이진 탐색 트리의 성질을 만족하는 이진트리이다.
- 트리의 각 노드들은 반드시 키 값을 가지고 있고, 키 값은 모두 달라야 한다.
- 왼쪽 서브 트리에 있는 데이터의 키는 그 루트의 키 값 보다 작다.
- 오른쪽 서브 트리에 있는 데이터의 키는 그 루트의 키 값보다 크다.
- 이진트리가 균형을 이루어져있다면 균형트리, 한쪽으로 편향되있을 경우 편향트리라고 한다.

## 이진탐색트리의 시간복잡도

- 이진탐색 : 탐색에 소요되는 시간복잡도는 O(logN), 하지만 삽입 삭제가 불가능하다.
- 연결리스트 : 삽입, 삭제의 시간복잡도는 O(1), 하지만 탑색하는 시간복잡도가 O(N)

해당 두가지의 장점을 합하여 만든것이 이진탐색트리 이다.

시간복잡도는 균등트리일때와 편향트리일때 두가지로 나눠진다.

- 균등 트리 : 노드 개수가 N개일 때 O(logN)
- 편향 트리 : 노드 개수가 N개일 때 O(N)

## 이진탐색트리의 구현

이진탐색트리의 삽입, 제거, 조회를 제네릭으로 구현해보았다.

```java
public class BinarySearchTree<T extends Comparable<T>> {
    private TreeNode<T> rootNode = null;

    private static class TreeNode<T extends Comparable<T>> {
        T data;
        TreeNode left;
        TreeNode right;

        public TreeNode(T data) {
            this.data = data;
            this.left = null;
            this.right = null;
        }
    }

    /**
     * 트리에서 item을 삽입한다.
     */
    public boolean insertNode(T item) {

        if (isEmpty()){
            rootNode = new TreeNode(item);
            return true;
        } else {
            TreeNode<T> node = rootNode;

            /** data 및 좌, 우를 비교하여 자식노드가 비어있는 곳을 찾아 넣어준다. */
            while (true) {
                if (node.data.compareTo(item) > 0) {

```