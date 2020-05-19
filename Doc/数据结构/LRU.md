# 一. 简介

LRU(Least Recent Used):最近最少使用，主要用于缓存淘汰。

![基本原理](https://raw.githubusercontent.com/SpikeFJ/picgo/master/image-20200519094654001.png)



# 二.数据结构选择

### 1. 队列/栈的问题

从上述示意图看，可能更加倾向于 选择 队列 或栈,如果选择 队列 或栈的问题在于：

如果读取一个已存在的元素，需要将元素移到头部；该元素如果在栈底还好，如果在栈/队列的中间，就很麻烦了。

### 2. 选择链表

所以我们考虑使用链表，链表可以针对元素方便的进行插入，移除操作。

既然已经确定了链表，那么应该选择 单端链表还是双端链表呢？单向还是双向呢？



1. 因为 `LRU`在超过指定容量后，需要移除尾部元素，如果采用单端链表，则需要遍历到尾部进行操作，性能太慢，

   **所以使用双端链表。**

2. 那么是单向还是双向呢？因为如果读取一个已存在的元素，需要将该元素移到头部。如果只是单向的话，需要记录前一个节点元素，而双向则更为方便。

   **实际单向也可以，但是双向操作更为简便**

### 3. 引入HashMap配合

以下两部分需要 `HashMap`:

1. 新增元素时，判断元素是否存在

2. 读取元素时，需要快速定位到该元素节点

   



总结下就是：`LRU`的基本行为是由 **双端链表** 来控制的，`HashMap`用于性能优化(提升存取速度)

# 三. 手写LRU

### 主体逻辑



**1. 插入逻辑**：

1. 如果不存在，
   1. 如果超过最大容量，则删除尾部元素
   2. 将新元素插入头部
2. 如果存在，则将元素移到头部

**2. 读取逻辑**：

​    1.如果存在，将元素移到头部，并返回



下面的代码应该是我们需要看到的效果

```java
public static void main(String[] args) {
    LRU lru = new LRU(3);
    lru.put(1);
    lru.print();//1

    lru.put(2);
    lru.print();//2,1

    lru.put(3);
    lru.print();//3,2,1

    lru.put(7);//移除1---> 7,3,2
    lru.print();


    lru.put(8);//移除2---> 8,7,3
    lru.print();

    lru.get(3);//3移到表头--->3,8,7
    lru.print();

    lru.put(4);//移除7---->4,3,8
    lru.print();
}
```

### 版本1核心代码

```java

public class LRU {
    //用于快速存取节点
    private HashMap<Integer, LinkNode> values = new HashMap<>();
    private LinkNode head;
    private LinkNode tail;
    private int maxSize;

    /**
     * 链表节点
     */
    class LinkNode {
        Integer value;
        LinkNode pre;
        LinkNode next;
    }


    public LRU(int size) {
        maxSize = size;
        head = new LinkNode();
        tail = new LinkNode();
    }


    public void put(Integer value) {
        if (!values.containsKey(value)) {

            //超过容量
            if (values.size() >= maxSize) {
                tail.pre.next = null;
                System.out.println("删除节点：" + tail.value);
                values.remove(tail.value);
                tail = tail.pre;
            }
            //新增
            LinkNode newNode = new LinkNode();
            newNode.value = value;


            if (values.size() == 0) {
                tail = newNode;
                head = newNode;
            } else {
                newNode.next = head;
                head.pre = newNode;

                head = newNode;
            }

            values.put(value, newNode);
        } else {

            //值已存在，则将该元素提到队里头
            LinkNode existObj = values.get(value);

            if (existObj != tail) {
                existObj.next.pre = existObj.pre;
                existObj.pre.next = existObj.next;
            } else {
                existObj.pre.next = null;
                tail = existObj.pre;
            }
        }

    }

    public Integer get(Integer value) {
        if (!values.containsKey(value)) {
            System.out.println("value = [" + value + "]不存在。");
            return null;
        }

        //值已存在，则将该元素提到队里头
        LinkNode existObj = values.get(value);

        if (existObj != tail) {
            existObj.next.pre = existObj.pre;
            existObj.pre.next = existObj.next;
        } else {
            existObj.pre.next = null;
            tail = existObj.pre;
        }

        head.pre = existObj;
        existObj.next = head;
        head = existObj;

        return existObj.value;
    }

    public void print() {
        if (head == null) {
            System.out.println("容器为空。");
        } else {
            LinkNode currenet = head;
            while (currenet != null) {
                System.out.print(currenet.value + " ");
                currenet = currenet.next;
            }
            System.out.println("");
        }
    }

    public static void main(String[] args) {
        LRU lru = new LRU(3);
        lru.put(1);
        lru.print();//1

        lru.put(2);
        lru.print();//2,1

        lru.put(3);
        lru.print();//3,2,1

        lru.put(7);//移除1---> 7,3,2
        lru.print();


        lru.put(8);//移除2---> 8,7,3
        lru.print();

        lru.get(3);//3移到表头--->3,8,7
        lru.print();

        lru.put(4);//4,3,8
        lru.print();
    }
}

```



### 版本2提取双端双向链表结构：

```java
/**
 * 队列节点
 */
class LinkNode {
    public Integer data;//可以使用泛型

    //此处使用双向链表，主要是方便元素在列表间移动
    public LinkNode pre;
    public LinkNode next;

    LinkNode(int data) {
        this.data = data;
    }
}


/**
 * 双端队列
 */
class LinkList {
    //此处采用双端队列,则是方便头插入，尾删除

    private LinkNode head;
    private LinkNode tail;

    public LinkList() {
        head = null;
        tail = null;
    }

    public boolean isEmpty() {
        return head == null;
    }

    public LinkNode insertFirst(int value) {
        LinkNode newNode = new LinkNode(value);
        if (isEmpty()) {
            tail = newNode;
        } else {
            head.pre = newNode;
        }
        newNode.next = head;
        head = newNode;

        return newNode;
    }

    public LinkNode removLast() {

        LinkNode removedNode = tail;
        if (head.next == null) {
            head = null;
        } else {
            tail.pre.next = null;
            tail = tail.pre;
        }
        return removedNode;
    }

    public LinkNode moveToFirst(LinkNode node) {
        if (node == head) {
            return node;
        }

        //断开已有关系
        if (node == tail) {
            removLast();
        } else {

            node.pre.next = node.next;
            node.next.pre = node.pre;
        }

        //维持新的关系
        node.next = head;
        head.pre = node;

        head = node;

        return node;
    }

    public void print() {
        if (head == null) {
            System.out.println("元素列表为空。");
            return;
        }

        LinkNode current = head;
        while (current != null) {
            System.out.print(current.data + " ");
            current = current.next;
        }
        System.out.println();
    }
}
```

```java
public class LRU2 {

    public LRU2(int maxSize) {
        this.maxSize = maxSize;
        linkList = new LinkList();
        values = new HashMap<>();
    }

    public void put(int value) {
        if (!values.containsKey(value)) {
            if (values.size() >= maxSize) {
                LinkNode removed = linkList.removLast();
                System.out.println("移除元素：" + removed.data);
            }

            LinkNode newNode = linkList.insertFirst(value);
            values.put(value, newNode);
        } else {
            linkList.moveToFirst(values.get(value));
        }
    }

    public int get(int value) {
        if (!values.containsKey(value)) {
            System.out.println("value = [" + value + "]不存在。");
            return -1;
        }

        LinkNode node = linkList.moveToFirst(values.get(value));
        return node.data;
    }

    public void print() {
        linkList.print();
    }


    private int maxSize;//最大容量
    private LinkList linkList;//维持元素实际元素的链表
    private HashMap<Integer, LinkNode> values;//用于快速读取指定元素

    public static void main(String[] args) {
        LRU2 lru = new LRU2(3);
        lru.put(1);
        lru.print();//1

        lru.put(2);
        lru.print();//2,1

        lru.put(3);
        lru.print();//3,2,1

        lru.put(7);//移除1---> 7,3,2
        lru.print();


        lru.put(8);//移除2---> 8,7,3
        lru.print();

        lru.get(3);//3移到表头--->3,8,7
        lru.print();

        lru.put(4);//移除7---->4,3,8
        lru.print();
    }
}
```

### 版本三-LinkedHashMap





总结：

LRU的关键在于链表的指针处理。如果不考虑性能，则可以实用单端链表，减少出错几率。