## 线程终端
* public static boolean interrupted()
* public boolean isInterrupted()
* public void interrupt();

interrupt()是执行中断，但是实际上只是将Thread中的线程的 `中断状态` 置为true，

1.如果线程处于阻塞状态(sleep，wait)，那么将会跳出阻塞并抛出InterruptException，sleep/wait方法中应该有如下逻辑
```
    while(true)
    {
        if(Thread.currentThread.isInterrupted())
        {
            // 跳出阻塞
            // throw new InterruptException();
        }
        else{

        }
    }
```
2. 如果线程处于非阻塞状态，无任何影响