# Chap0

## 线程

正在执行程序的主体 成为线程

start 方法启动新的线程

* 先启动新的线程
  
* 调用其run方法

并行（parallel）：表示多个操作“同时处理”
并发（concurrent）：表示将一个操作分割成多个部分，并允许无序执行（多个CPU时可以变成并行运行）。

### 线程启动

* 利用Thread的子类

`class MyThread extends Thread {}`

* 利用Runnable的接口

`class MyTask implements Runnable {}`

* 利用ThreadFactory 启动新的线程

`threadFactory.newThread(yourRunnable).start()`

### 线程的暂停

* Thread.sleep
  
中途唤醒被`Thread.sleep` 休眠的线程，则可以使用 `interrupt`方法。

### 线程的互斥

多个线程有可能同时操作同一个实例，可能会引发data race, 或是 race condition 竞态条件。

关键字`synchronized`修饰的方法，每次只能由一个线程运行，也称同步方法。
实例方法中使用的是当前的对象this的锁来执行互斥处理的；而synchronized静态方法中使用的是该类的类对象（`YourClass.class`）的锁来执行互斥处理的。

每个Java对象都可以用作实现一个同步的锁，这些锁又被称为内置锁（Intrinsic Lock）或是监视器锁（Monitor Lock）。

### 线程的协作

执行以下方法时必须先持有内置锁：

`wait`，`notify`, `notifyAll`方法

* `wait` 方法将线程放入等待队列（虚拟概念），便会释放锁；

* `notify`方法会从等待队列中唤醒其中的一个线程（具体哪个不确定，取决于平台实现）；

* `notifyAll`方法唤醒的是所有正在等待的线程，然后都尝试去获取锁。

非必要的情况的下，尽量使用`java.util.concurrent`提供的并发工具。
