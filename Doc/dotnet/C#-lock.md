

今天在群里看到有群友抛出这样的问题

![image-20200908164339400](C:\Users\SpikeF\AppData\Roaming\Typora\typora-user-images\image-20200908164339400.png)

![image-20200908164622513](C:\Users\SpikeF\AppData\Roaming\Typora\typora-user-images\image-20200908164622513.png)

那到底能不能使用`lock（typeof)`呢

并且给出了官方指导

https://docs.microsoft.com/zh-cn/dotnet/csharp/language-reference/keywords/lock-statement

下面就做几个试验

# 一. `lock(object)`

这种就是标准的写法，官方推荐的

```C#

class A
{
    private object lockObject = new object();

    public void LockObjMethod1()
    {
        lock (lockObject)
        {
            for (int i = 0; i < 5; i++)
            {
                Console.WriteLine("LockObjMethod1---->");
                Thread.Sleep(1000);
            }

        }
    }

    public void LockObjMethod2()
    {
        lock (lockObject)
        {
            for (int i = 0; i < 5; i++)
            {
                Console.WriteLine("LockObjMethod2---->");
                Thread.Sleep(1000);
            }
        }
    }
}

[TestMethod]
public void TestLockAObj()
{
    A a = new A();

    Console.WriteLine("-----lock a obj-------");
    Thread thr1 = new Thread(a.lc);
    thr1.Start();

    //暂停1秒，确保A线程先启动
    Thread.Sleep(1000);

    Thread thr2 = new Thread(a.LockThis2);
    thr2.Start();

    Console.WriteLine("-----lock this-------");
    thr1 = new Thread(a.LockThis1);
    thr1.Start();
}
```

打印如下,`LockObjMethod1`一直被`thr1`持有，`lock`不释放，`LockObjMethod2`等到`thr1`执行完毕才开始。

```
LockObjMethod1---->
LockObjMethod1---->
LockObjMethod1---->
LockObjMethod1---->
LockObjMethod1---->
LockObjMethod2---->
LockObjMethod2---->
LockObjMethod2---->
LockObjMethod2---->
LockObjMethod2---->
```

# 二. `lock(this)`

```C#
[TestMethod]
public void TestLockThis()
{
    A a = new A();


    Thread thr1 = new Thread(a.LockThis1);
    thr1.Start();

    //暂停1秒，确保A线程先启动
    Thread.Sleep(1000);

    Thread thr2 = new Thread(a.LockThis2);
    thr2.Start();

    a.Wait(10);
}

public void LockThis1()
{
    lock (this)
    {
        for (int i = 0; i < 5; i++)
        {
            Console.WriteLine("LockThis1---->");
            Thread.Sleep(1000);
        }

    }
}

public void LockThis2()
{
    lock (this)
    {
        for (int i = 0; i < 5; i++)
        {
            Console.WriteLine("LockThis2---->");
            Thread.Sleep(1000);
        }
    }
}
```

执行结果和lock(object)一致，因为`lock`锁住的`this`，就是 `A a`对象

所以如果换一个对象，锁就失效了

```C#
[TestMethod]
public void TestLockThis()
{
    A a = new A();


    Thread thr1 = new Thread(a.LockThis1);
    thr1.Start();

    //暂停1秒，确保A线程先启动
    Thread.Sleep(1000);

    A b = new A();
    Thread thr2 = new Thread(b.LockThis2);
    thr2.Start();

    a.Wait(10);
}
```

执行结果就是 

```
LockThis1---->
LockThis2---->
LockThis1---->
LockThis2---->
LockThis1---->
LockThis1---->
LockThis2---->
LockThis2---->
LockThis1---->
LockThis2---->
```

# 三. `lock(typeof(A))`

```C#
[TestMethod]
public void LockTypeMethod()
{
    A a = new A();

    Thread thr1 = new Thread(a.LockTypeMethod1);
    thr1.Start();

    //暂停1秒，确保A线程先启动
    Thread.Sleep(1000);

    A b = new A();
    Thread thr2 = new Thread(b.LockTypeMethod2);
    thr2.Start();

    a.Wait(10);
}


public void LockTypeMethod1()
{
    lock (typeof(A))
    {
        for (int i = 0; i < 5; i++)
        {
            Console.WriteLine("LockTypeMethod1---->");
            Thread.Sleep(1000);
        }

    }
}

public void LockTypeMethod2()
{
    lock (typeof(A))
    {
        for (int i = 0; i < 5; i++)
        {
            Console.WriteLine("LockTypeMethod2---->");
            Thread.Sleep(1000);
        }
    }
}


```

打印如下：

```
LockTypeMethod1---->
LockTypeMethod1---->
LockTypeMethod1---->
LockTypeMethod1---->
LockTypeMethod1---->
LockTypeMethod2---->
LockTypeMethod2---->
LockTypeMethod2---->
LockTypeMethod2---->
LockTypeMethod2---->
```



可以看到`lock(typeof(A))`,相当于在类上面加上了一把锁，A的所有实例对象都需要排队等待锁

所以也就不需要将`typeof(A)`赋值给一个全局对象了，因为多次`typeof(A)`返回的是同一个对象。