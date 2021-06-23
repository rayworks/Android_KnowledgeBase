# Balking Pattern

## Chap 4

### 介绍

如果现在不适合执行这个操作，或者没有必要执行，就停止处理，直接返回。
Balking 模式与Guarded Suspension 模式一样也存在守护条件。在Balking 中如果守护条件不成立，则立即中断处理；而Guarded Suspension 模式中是一直等待至可运行。

Guarded object （被防护的对象）
其角色是有一个被同步的方法（guardedMethod）的类。当线程执行到 guardedMethod 方法时，若守护条件成立，则执行实际的处理；否则，就直接返回。

### 拓展

#### 何时使用

* 并不需要执行时

* 并不需要守护条件成立时，如果不成立直接返回

* 守护条件仅在第一次成立时（e.g. 单例模式中的懒加载方式 lazy initialization）

#### balk 结果的表示

* 忽略，直接返回

* 通过返回值

  如通过boolean值来表示balk, false表示发生了balk, true 表示未发生；对于应用类型，返回null, 表示发生了balk。

* 通过异常来表示balk发生了

  即balk发生时不是从方法中返回return，而是抛出异常。


### 延伸

在Balking 模式与Guarded Suspension 模式之间

介于直接返回和“等待到守护条件为止”这两种极端的处理方法之间，还有一种办法就是，“在守护条件成立之前等待一段时间”。在守护条件成立之前等待一段时间，如果到时条件还未成立，则直接balk, 我们称之为guarded timed 或是 timeout。

#### 限时的wait

在调用wait方法时，还可以以参数指定超时时间。e.g.

```
obj.wait(1000); // timeout set as 1s
```

执行改语句时，线程会进入obj的等待队列，停止运行，并释放持有的obj锁。
当下面的情况之一发生时，线程退出等待队列。

* notify方法被执行

* notifyAll方法被执行

* interrupt方法执行时

  即线程的interrupt方法被调用执行。此时等待队列中的线程会重新获取obj锁，然后抛出InterruptedException异常。

* 超时发生时
  即wait中参数到期的情况。

  注意到，wait方法返回时我们无法区分，是notify/notifyAll方法被调用了，还是等待超时了。

#### java.util.concurrent中的超时

java.util.concurrent中的超时处理主要有以下2中方法：

1） 通过异常处理

* java.util.concurrent.Future#get()
* java.util.concurrent.Exchanger#exchange()
* java.util.concurrent.Cyclicarrier#await()
* java.util.concurrent.CountDownLatch#await()

2） 通过返回值处理

当执行多次try时，则不使用异常，而是使用返回值来表示超时。

* java.util.concurrent.BlockingQueue 接口
  offer() -> false

  poll() -> null

* java.util.concurrent.Semaphore

  tryAcquire() -> false

* java.util.concurrent.locks.lock 接口

  tryLock() -> false
