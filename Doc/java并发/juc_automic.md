#  AQS原理

> 获取锁：
>
> 1.  如果是独占锁获取不到则阻塞，进阻塞队列；当状态为允许获取锁时候，就要唤醒相应线程并从队列移除
>
> 2. 如果是共享锁获取不到就直接失败返回
>
>    

> 释放锁：
>
>    1.修改状态位，该动作会触发因为状态位而阻塞的线程

要支持上述两个api，需要具备三个条件

1. 状态位。必须是原子操作。
2. 阻塞和唤醒线程的能力。
3. 一个有序的阻塞队列

## 一. 状态位

> 这里采用32位的整数来描述该状态。Atomic包具备原子操作的能力。当然64位的同步器(AbstractQueuedLongSynchronizer)也是有的。

## 二. 阻塞/唤醒线程

> 标准的javaAPI是无法阻塞/唤醒一个线程的。Thread.suspend/resume都是过时的API，不推荐使用。
>
> JDK1.5之后利用JNI在LockSupport中实现了相关特性
>
> ```java
> LockSupport.park()
> LockSupport.park(Object)
> LockSupport.parkNanos(Object, long)
> LockSupport.parkNanos(long)
> LockSupport.parkUntil(Object, long)
> LockSupport.parkUntil(long)
> LockSupport.unpark(Thread)
> ```
>
> 

## 三. 有序队列

> 在AQS中采用CHL链表来实现队列保存阻塞的线程集合。

首先看下入队列的方法：

```java
   */
    private Node enq(final Node node) {
       //外层是一个死循环
        for (;;) {
            Node t = tail;
            //第一次插入时，将head、tail都指向一个新的节点
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //node的前置=tail
                node.prev = t;
                //如果tail=t，则tail = node
                if (compareAndSetTail(t, node)) {
                    //将尾节点的next指向node
                    t.next = node;
                    return t;
                }
            }
        }
    }
```



## 核心属性

```java
static final class Node {
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;
    
    	//节点操作因为超时或者对应的线程被interrupt。节点不应该留在此状态，一旦达到此状态将从CHL队列中踢出。
        static final int CANCELLED =  1;
    	//节点的继任节点是（或者将要成为）BLOCKED状态（例如通过LockSupport.park()操作），因此一个节点一旦被释放（解锁）或者取消就需要唤醒（LockSupport.unpack()）它的继任节点。
        static final int SIGNAL    = -1;
    	//表明节点对应的线程因为不满足一个条件（Condition）而被阻塞。
        static final int CONDITION = -2;
    	//正常状态，新生的非CONDITION节点都是此状态。
        static final int PROPAGATE = -3;

    	//节点的等待状态，对应上面的4种状态。非负值标识节点不需要被通知（唤醒）。
        volatile int waitStatus;
		//前置节点。节点的waitStatus依赖于前置节点的状态
        volatile Node prev;
   		//后置节点。后置节点是否被唤醒依赖于当前节点是否被释放
        volatile Node next;
        //节点对应的线程
        volatile Thread thread;
        //下一个等待条件(Condition)的节点，由于Condition是独占，所以单独一个队列
        Node nextWaiter; 
    }
    private transient volatile Node head;
    private transient volatile Node tail;
	//有多少个线程获取了锁。互斥锁<=1
    private volatile int state;

```



# ReentrantLock

1. 公平锁

```java

```



1. 不公平锁

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```



可以看出来两者差别在于公平锁是按顺序请求，不公平锁是先直接更改标志位，成功则直接获取锁，失败则和公平锁流程一致。



下面看下`acquire`的内部逻辑：

```java
/**
*独占模式获取对象，忽略中断。
*
/
public final void acquire(int arg) {
	//1.尝试获取锁，成功则返回
	//2.获取锁失败，则新增阻塞节点入CHL队列
	//3.自旋等待
	//4.成功获取锁后
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

```

可以看到有4个主要的方法，下面我们一一分析：

### 1. `tryAcquire`

留待子类实现，我们以`ReentrantLock`为例，

首先看公平锁：

```java
final void lock() {
    //由于ReentrantLock是独占锁，所以state=1
    acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    //获取状态位
    int c = getState();
    if (c == 0) {
        //1.判断队列中是否有排在当前线程之前的线程
        //2.没有则更改状态位
        //3.更改状态位成功后设置当前线程为独占线程
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        //如果是当前线程，则可重入，在现有状态位上叠加
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

/***
*队列是否包含其他线程节点,需要同时满足两个条件：
*1. head节点不等于tail节点
 2. head节点的后置节点为null  或 后置节点不等于当前线程
*/
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}

```

再看不公平锁：

```java
final void lock() {
    //首先直接强制获取锁，失败则后续和公平锁逻辑一直
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```





### 2. `addWaiter`方法：

```java
//1.首先直接尝试将新增节点置于tail之后，成功则返回
//2.失败则采用enq方法一直尝试直到成功。
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```



### 3.  `acquireQueued`

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // -1:
    if (ws == Node.SIGNAL)
        /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
        return true;
    //1：前置节点被取消了。
    if (ws > 0) {
        /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //0：将前置节点置为-1
        /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```





## 释放锁

```java
public final boolean release(int arg) {
    //tryRelease：true表示完成释放锁，当返回true时，才会继续执行后续的唤醒操作
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

