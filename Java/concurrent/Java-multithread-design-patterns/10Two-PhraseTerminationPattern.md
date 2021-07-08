# Chap 10

## Two-Phrase Termination Pattern

### 介绍

模式直译是“分阶段终止”的意思。它是一种先执行完终止处理逻辑再终止线程的模式。

```
                      开始 ⭕️
                        ⬇️
                      操作中
                        ⬇️
                      终止处理中
                        ⬇️
                      结束 ⭕️
```

该模式的特点（优雅地终止线程）有：

* 安全地终止线程（安全性）
* 必定会执行终止处理（生存性）
* 发出终止请求后尽快进行终止处理（响应性）

### 示例

``` java

class XThread extends Thread {
  private volatile boolean shutdownRequested = false;

  public void shutdownRequest() {
    shutdownRequested = true;
    interrupt();
  }

  @Override
  public void run() {
    try {
      while(!isShutdownRequested()) {
        doWork();
      }
    } catch(InterruptedException e) {
      //
    } finally {
      doShutdown();
    }
  }
}
```

### 拓展

* 不使用stop方法
  
  java.lang.Thread类提供了强制终止线程的stop方法，但是已经被标注为deprecated 方法，已不推荐使用。因为使用stop方法后，线程会抛出ThreadDeath的异常后终止，即使线程处于访问临界区（synchronized）的过程中也会终止。这样实例会失去安全性。

* 同时检测中断状态以及退出标志
  
  注意到shutdownRequest方法中除了设置退出标记外，还调用了interrupt方法，是为了确保线程在sleep以及wait时也能被终止, 提升了程序的响应性。

* 在长时间运行前检查退出标志
  
* NIO与多线程
  
  java.nio.channels.Channel 接口以及实现了该接口的类集合的设计中考虑了多线程的问题。线程在Channel上发生IO阻塞时，Channel#close方法会使得阻塞的线程接收到AsynchronousCloseException异常；Channel#interrupt方法会使得阻塞的线程接收到ClosedByInterruptException。

* join和isAlive方法
  
  Thread的join的方法等待指定的线程终止；isAlive来确认线程是否已经终止。

* ExecutorService接口与Two phrase Termination pattern
  
  ExecutorService接口可以“隐藏背后运行的线程，通过execute方法执行Runnable对象类型的工作”，为了优雅地执行终止运行的线程，该接口还提供了shutdown方法，以及一些辅助的方法：
  isShutdown 方法确认shutdown方法是否已经调用了（此时线程不一定已经停止）；
  isTerminated 方法确认线程是否已经停止了。

* 捕获程序整体的终止
  
  * 未捕获的异常的处理器
  
    Thread 中可以调用`setDefaultUncaughtExceptionHandler`静态方法来设置未捕获的异常的Handler（类型为`Thread.UncaughtExceptionHandler`），实际的处理是在该Handler 的`uncaughtException()`方法中。

  * 退出钩子(Shutdown hook)
    
    一般的，Java虚拟机退出，指的是`System.exit()`被调用或是全部非守护线程终止时。而退出钩子(Shutdown hook) 是指在Java虚拟机退出时启动的线程。可以使用`java.lang.Runtime`类的实例方法`addShutdownHook`来设置退出钩子。

### 延伸1 中断状态与InterruptedException异常的相会转换

* 中断状态 -> InterruptedException抛出异常 
  
  代码实现 如果线程处于中断状态就抛出 InterruptedException 异常

  ``` java
  if(Thread.interrupted()) {
      throw new InterruptedException();
  }
  ```

  Thread的实例方法interrupted()检查的是Thread.currentThread() 的中断状态。同时interrupted方法调用后则会清除中断状态；而如果想在不清除中断状态的前提下检查当前线程的中断状态，则可以使用isInterrupted()这个方法。

* InterruptedException异常 -> 中断状态
  
  想在指定的时间段让线程停止运行，可以这样实现：
  
  ```java
  try {
    Thread.sleep(1000);
  } catch(InterruptedException e){

  }
  ```

  上述处理InterruptedException异常会被忽略，如果某个线程在sleep时被其它线程中断，则“已经被中断”这个信息会遗失。为了保留这一信息，可以按如下操作再次中断自己。

  ```java
  try {
    Thread.sleep(1000);
  } catch(InterruptedException e) {
    Thread.currentThread().interrupt();
  }
  ```

### 延伸2 JUC 包与线程同步

* java.util.concurrent.CountDownLatch 类
  
  Thread.join方法只是等待了“线程终止”这一次性的操作，而使用CountDownLatch则可以实现“等待指定次数的countDown方法被调用”这一功能。通过持续倒数计，当计数值变为0后CountDownLatch的await方法会返回。

* java.util.concurrent.CyclicBarrier 类
  
  多次重复进行线程同步时，使用CyclicBarrier类会更方便。CyclicBarrier可以周期性地创建屏障（barrier）。在屏障解除之前，碰到屏障的线程是无法继续前进的。而屏障解除的条件是到达屏障的线程个数达到了构造方法中指定的个数。即指定个数的线程到达屏障处后，屏障就会被解除。CyclicBarrier 往往可以和CountDownLatch一起使用。

