# Chap7

## Thread-Per-Message Pattern

### 介绍

Thread-Per-Message Pattern 即为每个命令或请求分配一个处理线程。

在该模式中，消息的“委托端 ”和“执行端”是不同的线程。消息的委托端线程会告诉执行端线程“这项工作就交给你了”。

### 参与的角色

```
----------   request     --------    handle     ---------
| Client |  -------->    | Host  |  -------->   | Helper |
----------               --------               ---------
```

* `Client` 向Host角色发起请求，但并不知道Host角色是如何实现请求的。

* `Host` 收到Client的请求以后，会创建并启动一个新的线程，Host在创建的线程中会利用Helper角色来处理请求。

* `Helper` 为Host角色提供请求处理的角色。

### 拓展

* 提高响应性，缩短延时时间

  当handle操作较为耗时或是handle操作需要等待IO操作时，提升的效果非常明显。该模式下，Host角色会启动新的线程，由于启动线程本身也会耗费时间，所以想提高影响性时，是否采用Thread-Per-Message 模式取决于“handle操作花费的时间”和线程启动耗费时间之间的均衡（同时考虑受限的线程数量）。

* 适用于操作顺序没有要求时

  该模式中，处理请求的方法不一定按照原请求的调用顺序来执行的。因此当操作要求按照某种顺序执行时，Thread-Per-Message 模式并不适用。

* 适用于不需要返回值的情况

  在该模式中，request方法并不会等待handle方法执行结束，所以request得不到handle的运行结果。
  当需要获取某个操作的结果时，我们可以使用Future模式（第九章）。

* 应用于服务器

  为使服务器可以处理多个请求，可以使用Thread-Per-Message模式，服务器本身线程接收客户端的请求，而请求的处理则交由其它线程来执行。

* 调用方法 + 启动线程 -> 发送消息

  Thread-Per-Message模式中的request是一个普通方法，当该方法中的处理被执行完后，控制权就会返回。虽然request触发了目标操作，但是并不会等待处理结束。

### 延伸 : java.util.concurrent 和 Thread-Per-Message模式

* java.lang.Thread
  最基本的创建，启动线程的类

* java.lang.Runnable
  表示线程所执行的“工作”的接口

* java.util.concurrent.ThreadFactory
  将线程创建抽象化了的接口

* java.util.concurrent.Executor 接口
  将线程执行抽象化了的接口

* java.util.concurrent.Executors
  用于创建实例的工具类

* java.util.concurrent.ExecutorService 接口
  将被复用的线程抽象化了的接口

* java.util.concurrent.ScheduledExecutorService 类
  将被调度的线程的执行抽象化了的接口







