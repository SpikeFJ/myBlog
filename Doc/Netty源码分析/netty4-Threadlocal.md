在查看`MultithreadEventLoopGroup`代码时，发现有一个`newDefaultThreadFactory`,

该方法返回了一个`DefaultThreadFactory`对象，显然实现了`ThreadFactory`接口，但是`netty`的实现引起了我的注意：

![image-20200824192852887](https://i.loli.net/2020/08/24/ZRufVMatopCyKre.png)

和一般的实现，直接`new Thread`返回不同，`netty`返回了一个`FastThreadLocalThread`

从命名上看，应该是优化过`ThreadLocal`的线程对象。

首先查看下`Thread`原生的`ThreadLocal`实现

# 一. Thread原生实现


下面是一个`ThreadLocal`是使用样例，如果不使用`ThreadLocal`，则可能出现线程问题，导致最后打印不是100

![image-20200824194446943](https://i.loli.net/2020/08/24/fqS15n3wmPUWKTg.png)

我们跟踪`ThreadLocal`源码，看看到底其内部做了什么操作，让我们不采用锁也能做到线程安全。

### 1. `set`
首先观察下`set`方法：

```java
//ThreadLocal.java
public class ThreadLocal{
        public void set(T value) {
        //获取当前线程
        Thread t = Thread.currentThread();
        //根据当前线程获取ThreadLocalMap，在getMap中可以看到，ThreadLocalMap就是Thread对象的一个内部字段
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
     
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
	}
}

//Thread.java
public class Thread implements Runnable {
      ThreadLocal.ThreadLocalMap threadLocals = null; 
}

```

首次执行时，我们会调用`createMap`:

```java
//创建ThreadLocalMap对象，并填充到对应的线程对象中。
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

//ThreadLocalMap包含一个数据Entry，
//key:ThreadLocal
//value:对象值
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

后续执行`ThreadLocalMap`的`set`方法：

```java
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    //找到数组对应的下标
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

//依次找下一个数组，越界后则返回数组起始位置
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

到这里我们应该明白了，

当我们调用`ThreadLocal`的`set`方法时，实际上是将数据保存到当前线程的`ThreadLocalMap`对象中，

`ThreadLocalMap`对象以`ThreadLocal`为`key`，实际的值为`value`。



### 2.`get`

```java
public T get() {
    Thread t = Thread.currentThread();
    //找到当前线程对应的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```



