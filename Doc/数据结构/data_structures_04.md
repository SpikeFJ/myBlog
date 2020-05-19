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
    
    public LinkList()
    {
        first = null;
	}
    
    public boolean isEmpty()
    {
        return first == null;
    }
}
```

示意图：

![](https://raw.githubusercontent.com/SpikeFJ/picgo/master/image-20200519081240999.png)

`first`实际也是一个Node节点，但是它的作用是用于定位其他的Node节点，从`Fistt`出发可以遍历所有的节点。

单链表常见的操作如下：

* 在头部插入一个数据

* 删除头部对象

* 遍历

* 查找指定元素

* 删除指定元素

  

###  头部插入

![](https://raw.githubusercontent.com/SpikeFJ/picgo/master/image-20200519082713024.png)

核心代码：

```java
public void insertFirst(T value)
{
    Node newNode = new Node(value);
    newNode.next = first;
    first = newNode;//注意此处是头结点对象指向新节点，而不是next。
}

```

###  删除头部元素

![](https://raw.githubusercontent.com/SpikeFJ/picgo/master/image-20200519083247691.png)

核心代码：

```java
public void deleteFirst()
{
    //示意图顺序颠倒了。。。。
    head = head.next;
    head.next = null;
}
```

### 遍历

核心代码：

```java
public void walk()
{
    Node  current = first;
    while(current.next !=null)
    {
        System.out.println(current.value);
        current = current.next;
    }
}
```

### 查找指定元素

```java
public Node find(T value)
{
    if(isEmpty())
    {
        return null;
    }
	Node current = first;
    while(current.data!= value)
    {
        //next为null说明是最后一个节点，没有找到
        if(current.next ==null)
            return null;
        current = current.next;
    }
    return current;//已找到
}
```



### 删除指定元素

```java
/***
* 删除时：
1. 如果删除的是头节点，需要将头节点指向deleteNode的下一节点
2. 如果删除不是头节点，需要将deleteNode的前一个节点的next执行deleteNode的下一节点
但是一般遍历时只是记录了当前节点:current，所以还需要一个指针节点指向前一节点：preNode
*/
public Node delete(T value)
{
    if(isEmpty())
    {
        return null;
    }
    
    Node pre = first;
    Node current = first;
    
    while(current.data != value)
    {
        if(current.next == null)
        {
            return null;
        }
        pre = current;
        current = current.next;
    }
    
    if(current == head){
        //如果删除的是头节点，需要将头节点指向deleteNode的下一节点
        head = current.next;
        
        return current;
    }
    else
    {
        //如果删除不是头节点，需要将deleteNode的前一个节点的next执行deleteNode的下一节点
        pre.next = current.next;
        return current;
    }
}
```

## 2. 双端链表（注意 不是**双向**）

![双端链表](https://raw.githubusercontent.com/SpikeFJ/picgo/master/image-20200519085612262.png)

如果单链表需要在表尾插入则需要遍历到最后一个元素插入

双端链表由于引入最后一个节点的引用，可以在链表头插入，也可以在链表尾插入；
```java
class LinkList{
    Node<T> first;
    Node<T> last;
}
```
### 插入

```java
public class FirstLast{
    public FirstLast()
    {
        first = null;
        last = null;
    }
    
    //双端链表insertFirst时，需要注意如果为空，需要调整Last指针
    public void insertFirst(T value)
    {
        Node newNode= new Node();
        newNode.data = value;
        
        if(isEmpty())
        {
            last = newNode;
        }
        newNode.next = first;
        first = newNode;  
    }
    
    //双端链表insertLast时，需要注意如果为空，需要调整First指针
     public void insertLast(T value)
    {
        Node newNode= new Node();
        newNode.data = value;
        
        if(isEmpty())
        {
            first = newNode;
        }
        last.next = newNode;
        last = newNode;  
    }
    
}
```

**注意**：

1. 初始化时候，头指针(`First`),尾指针(`Last`)都是null
2. 头部插入时，如果列表为空，需要将`Last`也指向新节点
3. 尾部插入时，如果列表为空，需要将`First`也指向新节点
4. 头部删除时，如果只有一个元素(`First.next=null`),则`Last`也需要指向`null`
5. 尾部删除时，如果只有一个元素,则`First`也需要指向`null`



## 3. 有序链表

有序链表中，数据是按照关键值有序排列的。相较于有序数组，链表插入速度很快，而且可以扩展方便，不需要初始化时就确定总体容量。

关键是插入时，从first节点一直找到直到小于本节点的位置。

## 4. 双向链表

传统的一个潜在问题是：反向的遍历比较困难，正向遍历采用 `current = current.next`即可。

![双向链表](https://raw.githubusercontent.com/SpikeFJ/picgo/master/image-20200519091335433.png)

```java
class Node<T>{
    T data;
    Node<T> previous;
    Node<T> next;
}
```

双向链表不必是双端链表，但是保持双端引用是有用的。

### 头部插入

![](https://raw.githubusercontent.com/SpikeFJ/picgo/master/image-20200519091846787.png)

```java
public void insertFirst(T value)
{
    //步骤1和2好像是可以颠倒的
    newNode.next = first;//①
    if(isEmpty())
    {
        last = newNode;
    }
    else
    {
        first.pre = newNode//②
    }
    
    first =newNode;//③
}
```

![指定元素后插入](https://raw.githubusercontent.com/SpikeFJ/picgo/master/image-20200519092827735.png)

```java
public void insertAfter(current)
{
    newNode;
    if(current ==last)
    {
        newNode.next = null;
        last =newNode;
    }
    else
    {
        //newNode和下一个节点关系,顺序是否可以颠倒？
        current.next.pre = newNode;
        newNode.next = current.next;
    }
    //newNode和前一个节点(current)关系
    newNode.pre = current;
    current.next = newNode;
   
}
```



## 5. 迭代器