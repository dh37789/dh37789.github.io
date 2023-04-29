---
title:  "[Java] Comparable을 이용해 객체를 한글, 영어, 숫자 순 정렬하기"

layout: post
categories: Java

toc: true
toc_sticky: true

date: 2023-04-29
last_modified_at: 2022-04-29
---

`Comparable<T>` interface는 구현한 객체에 정렬기준을 정해 Array.sort()나 Collections.sort()를 통해 정렬을 했을때, `Comparable<T>`를 구현한 compareTo를 기준으로 객체의 순서를 정렬해주는 interface이다.

아래의 `Node` 객체는 단순한 String type의 name을 필드로 가지고 있는 객체이다.

한번 `Comparable<T>`를 구현해 한글, 영어, 숫자 순으로 정렬 해보자.

```java
public class Node {

  private String name;

  public String getName() {
    return name;
  }

  public Node(String name) {
    this.name = name;
  }

}
```

@Override 한 `compareTo` 메소드에 현재 Node객체의 name과, 비교할 Node 객체의 name을 받아 순서를 정해준다.

비교하고자 하는 값보다 작으면 음수, 크면 양수를 반환하도록 구현한다.

내부 비교 로직에서 `Character.OTHER_LETTER` 는 한글을, `Character.isLetter(char)`는 알파벳을 `Character.isDigit(char)`는 숫자를 찾는 메서드이다.

```java
public class Node implements Comparable<Node>{

    private String name;

    public String getName() {
        return name;
    }

    public Node(String name) {
        this.name = name;
    }

    @Override
    public int compareTo(Node node) {
        String s1 = getName();
        String s2 = node.getName();

        char c1 = s1.charAt(0);
        char c2 = s2.charAt(0);

        /** 한글, 영어, 숫자 순서로 정렬 */
        if (Character.getType(c1) != Character.getType(c2)) {
            if (Character.getType(c1) == Character.OTHER_LETTER) {
                return -1;
            } else if (Character.getType(c2) == Character.OTHER_LETTER) {
                return 1;
            } else if (Character.isLetter(c1)) {
                return -1;
            } else if (Character.isLetter(c2)) {
                return  1;
            } else {
                return 1;
            }
        } else if (Character.isDigit(c1) && Character.isDigit(c2)) {
          return Integer.compare(Character.getNumericValue(c1), Character.getNumericValue(c2));
        } else {
            return s1.compareTo(s2);
        }
    }
}
```

한번 테스트코드를 작성해서 확인해 보도록 하자.

무작위의 단어들을 Node 객체에 넣어 정렬을 진행 하였다.

제대로 정렬이 된다면 순서대로 가, 나, 하, c, g, z, 1, 3, 5의 순서로 정렬이 되어야 한다.

```java
class NodeTest {

    @Test
    void 한글_영어_숫자_정렬테스트() {
        Node nodeKorean_가 = new Node("가가가aa");
        Node nodeEnglish_c = new Node("ccc22");
        Node nodeNumber_3 = new Node("3초는 어떻게 기다려");
        Node nodeKorean_나 = new Node("나나나vv");
        Node nodeKorean_하 = new Node("하하하송");
        Node nodeEnglish_g = new Node("ggg11");
        Node nodeEnglish_z = new Node("zi존법사");
        Node nodeNumber_5 = new Node("5늘은 말할꺼야");
        Node nodeNumber_1 = new Node("1초라도 안보이면");

        List<Node> nodes = Arrays.asList(nodeKorean_가, nodeEnglish_c, nodeNumber_3, nodeKorean_나, nodeEnglish_g, nodeNumber_5, nodeNumber_1, nodeKorean_하, nodeEnglish_z);
        Collections.sort(nodes);

        assertEquals(nodes.get(0).getName(), nodeKorean_가.getName());
        assertEquals(nodes.get(1).getName(), nodeKorean_나.getName());
        assertEquals(nodes.get(2).getName(), nodeKorean_하.getName());
        assertEquals(nodes.get(3).getName(), nodeEnglish_c.getName());
        assertEquals(nodes.get(4).getName(), nodeEnglish_g.getName());
        assertEquals(nodes.get(5).getName(), nodeEnglish_z.getName());
        assertEquals(nodes.get(6).getName(), nodeNumber_1.getName());
        assertEquals(nodes.get(7).getName(), nodeNumber_3.getName());
        assertEquals(nodes.get(8).getName(), nodeNumber_5.getName());
    }
}
```

테스트가 성공하는 것을 볼 수 있다.

![sort1]({{site.url}}/public/image/2023/2023-04/29-sort001.png)
