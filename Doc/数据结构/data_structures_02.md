本次主要介绍栈、队列和优先级队列三种数据结构。

不同于数组、链表、树等存储数据结构。


1. 此三者都是作为辅助工具来运用：生命周期相对较短，程序运行时被创建，结束后被销毁
2. 受限访问，不能像数组那样直接下标访问，即特定时刻只有一个数据项可以被操作
3. 更加抽象：主要通过接口和外界通讯，内部对外不可见。
> 如栈可以用数组实现，也可以用链表来实现；优先级队列既可以用数组实现也可以用堆来实现

# 一. 栈
`栈只可以访问最后插入的数据项，移除该数据项后才可以访问倒数第二个访问的数据项`

## 1. 栈的使用场景
1. 栈在[解析算术表达式](./data_structures_03.md)时起到极为重要的作用
2. 二叉树中用栈来遍历树节点；图中用栈来辅助查找图的顶点(一种可以用来解决迷宫问题的技术)
3. 大部分微处理器都采用基于栈的体系结构：当调用方法时，把它的返回地址和参数压入栈；方法返回时，数据出栈。

## 2. 栈的常用接口
1. `push`(入栈)
2. `pop`(出栈)
3. `peek`(查看)

## 3. 代码实现(基于数组)
```java
public class Stack {

    private final int maxSize;
    private int[] elements;
    //指向栈顶
    private int top = -1;

    public Stack(int maxSize) {
        this.maxSize = maxSize;
        elements = new int[maxSize - 1];
    }

    public void push(int value) {
        elements[++top] = value;
    }

    public int pop(int value) {
        return elements[top--];
    }

    public int peek(int value) {
        return elements[top];
    }

    public boolean isEmpty() {
        return top == -1;
    }

    public boolean isFull() {
        return maxSize == top + 1;
    }
}
```
## 4. 实际应用

1. 单词逆序
2. 分隔符匹配

总结：栈是一个概念上的辅助工具，可以极大的提升开发效率。

# 二. 队列
`队列遵循着FIFO(先进先出)的原则，即最后一个插入的数据项也会最后一个被移除，而栈是LIFO(后进先出)`

## 1. 循环队列
```java
public class Queue {

    private final int maxSize;
    private int[] elements;
    //实际队列元素个数
    private int elementNum;
    private int head;
    private int tail;

    public Queue(int maxSize) {
        this.maxSize = maxSize;
        elements = new int[maxSize];
        head = 0;
        tail = -1;
    }

    public void insert(int value) {
        //当队尾元素达到最大值时，返回初始位置
        if(tail == maxSize-1){
            tail=-1;
        }
        elements[++tail]=value;
        elementNum++;
    }

    public int remove(int value) {
        int tmp = elements[head++];
        //当对头元素道道最大值时，返回初始位置
        if(head==maxSize) {
            head=0;
        }
        elementNum--;
        return tmp;
    }

    public int peek(int value) {
        return elements[head];
    }

    public boolean isEmpty() {
        return elementNum == 0;
    }

    public boolean isFull() {
        return maxSize == elementNum;
    }
}
```
## 2. 双端队列
`双端队列就是一个两端都可以插入和移除数据项的队列`

# 三. 优先级队列

抢占式多任务操作系统中，程序队列就存放在优先级队列中

这里均采用数组实现，有两种实现思路
1. 缓慢插入，快速删除
> 插入时排序且需要移动过平均一半的数据，删除时移除数组最前面的元素

2. 快速插入，缓慢删除
> 插入时简单插入顶部，删除时需要找到最小值且需要移动过平均一半的数据

这里采用方式1.

更好的方法参见[堆结构](./data_structures_11.md)(因为堆不期望整体有有序，只需要局部有序所以性能更好)

代码如下
```java
 public void insert(int value) {
        if (elementNum == 0) {
            elements[elementNum++] = value;
        } else {
            //插入排序
            int i;
            for (i = elementNum - 1; i >= 0; i--) {

                if (elements[i] > value) {
                    elements[i + 1] = elements[i];
                } else {
                    break;
                }
            }
            elements[i] = value;
            elementNum++;
        }
    }

    public int remove(int value) {
        return elements[elementNum--];
       
    }
```