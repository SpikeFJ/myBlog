# 一. 任务调度

普通意义上的任务调度分为两种：

1. 基于时间的周期性运行

> `eg`: 每天2点执行

`eg`:每隔`2s`执行，这里隐含了不管任务是否完成，都应该是基于时间精确调度

1. 基于执行周期的间隔运行

可以间隔理解为上一个任务执行完成后，间隔多长时间再次执行



# 二 `java`中的常用调度方式

### 2.1. `spring`的`@Scheduled`

#### 2.1.1 `fixedDelay`

示例代码如下：

```java
//正常执行
@Scheduled(fixedDelay = 1000)
public void t1() {
try {

System.out.println("T1---->" + DateUtil.getTimes());
}catch (Exception e) {

}
}

//超时执行
@Scheduled(fixedDelay = 1000)
public void t1() {
try {

System.out.println("T1---->" + DateUtil.getTimes());
Thread.sleep(2000);
}catch (Exception e) {

}
}
```

执行显示：

```
正常执行：
T1---->2021-03-11 10:44:22
T1---->2021-03-11 10:44:23
T1---->2021-03-11 10:44:24
T1---->2021-03-11 10:44:25

超时执行：
T1---->2021-03-11 10:44:26
T1---->2021-03-11 10:44:29
T1---->2021-03-11 10:44:42
T1---->2021-03-11 10:44:45
```

**结论：**

**当`fixedDelay`大于等于任务执行耗时时，以`fixedDelay`的值作为调度间隔（这里是`1s`）**

**当`fixedDelay`小于等于任务执行耗时时，以`fixedDelay` + 任务执行耗时 作为调度间隔（这里就是`1s+2s=3s`）**

#### 2.1.2 `fixedRate`

示例代码如下：

```java
//正常执行
@Scheduled(fixedRate = 1000)
public void t2() {
try {

System.out.println("T2---->" + DateUtil.getTimes());
}catch (Exception e) {

}
}

//超时执行
@Scheduled(fixedRate = 1000)
public void t2() {
try {

System.out.println("T2---->" + DateUtil.getTimes());
Thread.sleep(2000);
}catch (Exception e) {

}
}
```

执行显示：

```
正常执行：
T2---->2021-03-11 10:49:38
T2---->2021-03-11 10:49:39
T2---->2021-03-11 10:49:40
T2---->2021-03-11 10:49:41

超时执行：
T2---->2021-03-11 10:50:36
T2---->2021-03-11 10:50:38
T2---->2021-03-11 10:50:40
T2---->2021-03-11 10:50:42
```

**结论：**

**当`fixedRate`大于等于任务执行耗时时，以`fixedRate`的值作为调度间隔（这里是`1s`）**

**当`fixedRate`小于等于任务执行耗时时，上一次任务执行完成后立刻执行下一次任务，`fixedRate`不生效**

### 2.2 `cron`表达式

示例代码如下：

```java
//正常执行
@Scheduled(cron = "0/1 * * * * ?")
public void t3() {
try {
System.out.println("T3---->" + DateUtil.getTimes());
}catch (Exception e) {

}
}

//超时执行
@Scheduled(cron = "0/1 * * * * ?")
public void t3() {
try {

System.out.println("T3---->" + DateUtil.getTimes());
Thread.sleep(2000);
}catch (Exception e) {

}
}
```

执行显示：

```
正常执行：
T3---->2021-03-11 10:54:25
T3---->2021-03-11 10:54:26
T3---->2021-03-11 10:54:27
T3---->2021-03-11 10:54:28

超时执行：
T3---->2021-03-11 10:56:17
T3---->2021-03-11 10:56:20
T3---->2021-03-11 10:56:23
T3---->2021-03-11 10:56:26
```

**结论：**

**当`cron`间隔大于等于任务执行耗时时，以`cron`的值作为调度间隔（这里是`1s`）**

**当`cron`间隔小于等于任务执行耗时时，以`cron`调度间隔 + 任务执行耗时 作为调度间隔（这里就是`1s+2s=3s`）**

**效果同`fixedDelay`**