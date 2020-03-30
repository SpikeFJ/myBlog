## 一. 类和接口
## 1. 如何区分好的设计与坏的设计

> **好的设计隐藏其内部数据和内部实现，对外暴露的API和其内部实现完全隔离。**

## 2. 组合优于继承

> 继承的缺点在于子类依赖于父类中的具体实现。

如何理解?

作者举例，有这样一个需求：想统计hashset从创建以来一共添加了多少元素，因为hashset有两个添加方法：`add`,`addAll`,采用继承实现如下：
```java
/**
* 该类看起来和合理。但是实际执行都是错误的返回,期望3返回却是6
* A a= new A();
* a.addAll(Array.asList("a","b","c"))
*
*/
A extends HashSet{
    private int addNum;

    public boolean add(E e)
    {
        addNum++;
        return super.add(e);
    }

    public boolean addAll(Collection<? extends E> c)
    {
        addNum+=c.size();
        return super.addAll(c);
    }
}
```

根本原因在于Hashset的`addAll`实现是基于`add`方法的，即`addAll`内部实现是调用了`add`。

解决方法有两种
1. 删除掉自己实现的`addAll`实现

> 但该操作隐含了如下的规则：hashset的AddAdll必须是基于add，而且必须告知外界自己的是基于add实现的这个内部细节。
> 这是不可以接受的，一旦hashset哪一天改变了`addAll`的实现，不是基于`add`方法的。那我们该如何处理呢？

2. A自己重写addAll,遍历所有元素，调用add
> 这样相当于重新实现了一遍父类的方法，而且父类的方法有可能是自用的，一些相关的私有域有可能子类根本无法获取到。

## 18. 接口优于抽象类

> 按照惯例，骨架实现被称之为AbstractInterface



## 二. 泛型