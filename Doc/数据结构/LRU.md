LRU(Least Recent Used):最近最少使用，主要用于缓存淘汰。

![基本原理](https://raw.githubusercontent.com/SpikeFJ/picgo/master/image-20200519094654001.png)

因为LRU涉及到插入，移除操作比较多，所以我们应该想到链表应该是首选，但是链表不能快速定位元素是否存在

所以应该配合`HashMap`。



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

版本1核心代码：

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



版本2:提取双端双向链表结构：

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

总结：

LRU的关键在于链表的指针处理。如果不考虑性能，则可以实用单端链表，减少出错几率。