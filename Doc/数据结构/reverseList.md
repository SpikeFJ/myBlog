重温下链表的翻转。

# 一. 链表的表示

首先链表有两种表示方式，一种有哨兵(一个没有实际意义的Node节点),一种没有哨兵，

## 1. 哨兵

![image-20200815191810610](https://i.loli.net/2020/08/15/svkqOe4orcztCDR.png)

```java
public class LinkList2 {
    /**
    head作为哨兵节点，没有意义，-1是一个特殊值
    **/
    private Node head = new Node(-1, null);

    class Node {
        private int value;
        private Node next;

        public Node(int value, Node next) {
            this.value = value;
            this.next = next;
        }
    }

    public void add(int value) {
        Node last = head;//找到最后一个节点(next为null)
        while (last.next != null) {
            last = last.next;
        }
        last.next = new Node(value, null);
    }

    void print() {
        Node tmp = head;
        while (tmp.next != null) {
            tmp = tmp.next;
            System.out.print(tmp.value + " ");
        }
        System.out.println();
    }
}
```

## 2. 无哨兵

![image-20200815191851291](https://i.loli.net/2020/08/15/c3uglIhCnHANzOQ.png)

```java
public class LinkList3 {
    private Node head;

    class Node {
        private int value;
        private Node next;

        public Node(int value, Node next) {
            this.value = value;
            this.next = next;
        }
    }

    public void add(int value) {
        //需要添加额外的判断
        if (head == null) {
            Node newNode = new Node(value, null);
            head = newNode;
        } else {
            Node last = head;//找到最后一个节点(next为null)
            while (last.next != null) {
                last = last.next;
            }
            last.next = new Node(value, null);
        }
    }

    void print() {
        Node tmp = head;
        while (tmp.next != null) {
            System.out.print(tmp.value + " ");
            tmp = tmp.next;
        }
        System.out.println();
    }
}
```

综上比较我们采用哨兵的表示方式，减少判断语句的使用。



# 二.递归实现

## 1. 大体思路

首先初始链表类似于`head--->1--->2--->3--->4`

可以将问题分解，head委托给以1起始的链表，1委托给以2起始的链表，形成递归

最后一个节点,需要

①翻转节点：4--->3，

②删除3--->4的指针



此时有个问题`head.next`指向了`null`，需要将`head.next`置为Node4,即链表最后一个节点，

所以需要保持最后一个节点，以便后续引用

```java
/**
 * 1--->2--->3--->4
 * 想要翻转以Node1为起始的链表
 * 可以委托给以2起始的链表，递归
 * 最后一个节点,需要翻转节点：4--->3，并且删除3--->4的指针
 *
 * 1<---2<---3<---4
 * 此时有个问题head.next仍旧指向了1，需要将head.next置为4的
 *
 *1<---2<---3<---4<---head
 */
```



```java
private void reverse() {
    //注意不能把head节点纳入到递归中
    reverse(head.next);
    head.next = newFirst;
}

//保存最后一个节点，以便后续head.next指针指向
private newFirst =null;

Node reverse(Node node) {
    if (node.next == null) {
        newFirst = node;
        return node;
    }
    //1.递归调用next节点，返回原节点值
    Node pre = reverse(node.next);

    //2.指针翻转
    pre.next = node;

    //3.原有指针删除
    node.next = null;

    //返回node而不是pre,执行到此处不要忘记rever函数的作用是翻转以node为首节点的链表
    //1--->2--->3--->4 返回的是1
    //2--->3--->4--->返回的是2
    return node;
}
```



## 2. 另一种递归

该方法少了最后一个节点的保存工作，因为`reverse2`永远返回的是最后一次节点，

```java
void anotherRever() {
    Node nodeLast = reverse2(head.next);
    head.next = nodeLast;
}

/**
** reverse2永远返回的是原链表的最后一个节点，这点有点别扭
*/
Node reverse2(Node node) {
    if (node.next == null) {
        return node;//①
    }
    //1.递归调用next节点，
    Node alwayLastNode = reverse2(node.next);

    //2.此处翻转当前首节点(node）的下一个节点
    node.next.next = node;

    //3.原有指针删除
    node.next = null;
    
	//此处永远返回的是①中返回的值，即最后一个节点
    return alwayLastNode;
}
```



# 三.非递归实现

非递归采用`pre`，`cur`两个指针，

每次将`cur.next`指向`pre`，

```java
/**
 * 采用非递归方式
 */
private void reverseWithDoublePoint() {

    if (head.next == null) {
        System.out.println("没有元素");
        return;
    }
    if (head.next.next == null) {
        System.out.println("只有一个元素");
        return;
    }

    Node pre = head.next;
    Node cur = pre.next;

    //第一个元素会转变为最后一个元素，所以指向为null，后续的元素会指向前一个元素
    pre.next = null;

    while (cur != null) {
        //cur的next元素后面需要指向前一个元素，所以需要将next暂存
        Node tmp = cur.next;

        cur.next = pre;

        pre = cur;
        cur = tmp;
    }
    head.next = pre;
}
```

# 四.扩展

## 1. 两个链表是否有相交的节点

首先需要知道两个链表的长度，两种方式：

1. 链表中包含`size`属性，每次`add`更新该元素
2. 递归调用，查询长度



用长链表减去短链表得出`diff=len2-len1`,短链表首先偏移`diff`，然后两个链表元素逐个比较



## 2. 单个链表是否有环

采用快慢指针

```java
public Node checkCycle() {
    Node slow = head;
    Node fast = head;

    while (fast != null && fast.next != null) {
        //快指针每次2格
        fast = fast.next.next;
        //慢指针每次一格
        slow = slow.next;
        
        //链表结束，没有环
        if (fast == null) {
            return null;
        }

        //相遇，有环
        if (slow == fast) {
            return slow;
        }
    }
    //链表结束，没有环
    return null;
}
```

关于链表环可以衍生出很多有趣的题目，大脑暂时不够用，算了，略。

