最近在熟悉 `netty`源码，`首先接触到的就是 `NioEventLoopGroup`，发现它继承了 `Executor`，

所以就研究了下 java 下的 `Executor`执行框架

首先我们看下 Executor 的 类注释

> Executor 是用来执行提交的 Runable 任务的。
>
> 该接口提供了一种将任务的提交和具体实现（包括如何入刑，线程明细及如何调度）分离的机制。
>
> 一般情况下，应该使用Executor而不是显式地创建线程。
>
> 相较于采用：
>
> ​	new Thread(new RunableTask()).start()
>
> 我们更倾向于使用：
>
> ​	Executor executor = anExecutor;
>
> ​	executor.execute(new RunnableTask1)
>
> ​	executor.execute(new RunnableTask2)
>
> 然而，Executor接口并没有明确要求执行必须是异步的。
>
> 最简单的样例，一个executor可以直接在调用的线程中运行提交的任务：
>
> class DirectExecutor implements Executor
>
> {
>
> ​	public void execute(Runable r)
> ​	{
>
> ​		r.run();
>
> ​	}
>
> }

