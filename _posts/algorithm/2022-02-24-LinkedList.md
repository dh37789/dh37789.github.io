---
title:  "[Theory] LinkedList"

categories: algorithm

toc: true
toc_sticky: true

date: 2022-02-24
last_modified_at: 2022-02-28
---

# 연결리스트(LinkedList)

![연결리스트]({{site.url}}/assets/image/2022-02-25/linkedList001.PNG)
연결 리스트(Linked List)란 각 노드가 데이터와 포인터를 가지고 한 줄로 연결되어 있는 방식으로 데이터를 저장하는 자료 구조를 말합니다.  
이름과 같이 각 데이터를 담은 노드들이 연결되어 있는데, 노드의 포인터가 다음이나 이전의 노드의 연결을 담당하고 있습니다.  

연결리스트의 종류로는 단일 연결 리스트, 이중 연결 리스트 등이 있습니다.

## 연결리스트 구현

LinkedListNode<T> : LinkedList의 노드객체입니다. link 변수는 해당 노드(Vertex)에 연결된 다른 노드의 주소를 가지고있습니다. 

```java
public class LinkedListNode<T> {
    T data;
    LinkedListNode<T> link;

    /** LinkedListNode의 초기화  */
    public LinkedListNode(T data){
        this.data = data;
        this.link = null;
    }
}
```

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

```java
public class LinkedList<T> {
    /** LinkedList의 head */
    private LinkedListNode<T> head;
    /** LinkedList의 tail */
    private LinkedListNode<T> tail;
    /** LinkedList의 현재 size */
    private int size;

    /** LinkedList의 초기화 */
    public LinkedList(){
        this.head = null;
        this.tail = null;
        this.size = 0;
    }

    /** LinkedList의 첫번째 위치의 head에 data를 삽입한다. */
    public void addFirst(T data){
        LinkedListNode<T> node = new LinkedListNode<>(data);
        if(isEmpty()){
            head = node;
            tail = node;
        } else {
            node.link = head;
            head = node;
        }
        size++;
    }

    /** LinkedList의 마지막 위치인 tail에 data를 삽입한다. */
    public void addLast(T data){
        LinkedListNode<T> node = new LinkedListNode<>(data);
        if(isEmpty()){
            addFirst(data);
        } else {
            tail.link = node;
            tail = node;
            size++;
        }
    }

    /** LinkedList의 index위치에 data를 삽입한다. */
    public void add(int idx, T data){
        if(isEmpty()){
            addFirst(data);
        } else {
            if (idx == 0){
                addFirst(data);
            } else if (idx == size){
                addLast(data);
            } else if (idx > size){
                throw new ArrayIndexOutOfBoundsException("OutOfBounds!! Index : " + idx + ", Size : " + size);
            } else {
                LinkedListNode newNode = new LinkedListNode(data);

                LinkedListNode tmpNode = getNode(idx-1);
                LinkedListNode tmpLinkNode = tmpNode.link;
                newNode.link = tmpLinkNode;
                tmpNode.link = newNode;
                size++;

                if (newNode.link == null){
                    tail = newNode;
                }
            }
        }
    }

    /** LinkedList의 첫번째 위치의 head부분의 데이터를 Object객체로 반환하고 삭제한다. */
    public Object removeFirst(){
        if(isEmpty()){
            throw new ArrayIndexOutOfBoundsException("OutOfBounds!! Size : " + size);
        } else {
            LinkedListNode tmpNode = head.link;
            Object returnData = head.data;

            head = tmpNode;
            size--;

            if (head == null){
                tail = null;
            }
            return returnData;
        }
    }

    /** LinkedList의 마지막 위치의 tail부분의 데이터를 Object객체로 반환하고 삭제한다. */
    public Object removeLast(){
        return remove(size-1);
    }

    /** LinkedList의 idx위치의 data를 Object객체로 반환하고 삭제한다. */
    public Object remove(int idx){
        if(idx >= size){
            throw new ArrayIndexOutOfBoundsException("OutOfBounds!! Index : " + idx + ", Size : " + size);
        } else if (idx == 0) {
            return removeFirst();
        } else {
            LinkedListNode tmpNode = getNode(idx - 1);
            LinkedListNode deleteNode = tmpNode.link;

            Object returnData = deleteNode.data;

            if (deleteNode == tail){
                tail = tmpNode;
                tmpNode.link = deleteNode.link;
            } else {
                tmpNode.link = tmpNode.link.link;
            }
            size--;
            return returnData;
        }
    }

    /** LinkedList의 모든 데이터를 삭제한다. */
    public void clear() {
        size = 0;
        head = null;
        tail = null;
    }

    /** LinkedList의 크기를 반환한다. */
    public int size(){
        return size;
    }

    /** LinkedList의 크기가 비어있는지 확인한다. */
    public boolean isEmpty(){
        return (head == null);
    }

    /** LinkedList의 data항목을 String객체로 반환한다. */
    public String toString(){
        if (size == 0){
            return "[]";
        } else {
            StringBuffer sb = new StringBuffer();
            LinkedListNode strNode = head;
            sb.append("[");
            while (strNode != null){
                sb.append(strNode.data);
                strNode = strNode.link;
                if(strNode != null) {
                    sb.append(",");
                }
            }
            sb.append("]");
            return sb.toString();
        }
    }

    /** idx 위치에 해당하는 LinkedList 노드 객체를 가져옵니다. */
    private LinkedListNode getNode(int idx){
        LinkedListNode node = head;
        for (int i = 0; i < idx;i++){
            node = node.link;
        }
        return node;
    }

}
```


