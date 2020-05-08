之前的大都数结构我们都采用了数组作为底层存储结构。

我们也看到了数组作为存储的一些缺陷：
* 无序数组搜索低效
* 有序数组插入低效
* 不管有序无序，删除都低效

在此，我们介绍一种新的存储结构：链表。

首先给出链表的伪代码结构
```java
class Node<T>{
    T data;
    Node<T> next;
}
```

## 1. 单链表
```java
class LinkList{
   Node<T> first;
}
```

## 2. 双端链表（注意 不是**双向**）

双端链表可以在链表头插入，也可以在链表尾插入
```java
class LinkList{
   Node<T> first;
    Node<T> last;
}
```
## 3. 有序链表

关键是插入时，从first节点一直找到直到小于本节点的位置。

## 4. 双向链表

```java
class Node<T>{
    T data;
    Node<T> previous;
    Node<T> next;
}
```

## 5. 迭代器