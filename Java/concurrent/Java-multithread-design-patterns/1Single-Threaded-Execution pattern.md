# Chap1

## Single Threaded Execution

### 模式定义

即设置限制，确保同一时间只能以一个线程执行处理。
SharedResource（共享资源）类，通过控制访问（e.g. `synchronized`）来实现。

* 非原子操作执行过程中可能出现线程切换，这就就可能带来线程安全的问题。
`this.counter++;`

* 将只允许单个线程执行的程序范围称为临界区（critical section）。

### 适用时机

* 多线程访问时

* 状态有可能发生变化时 （如果是多线程访问，但是资源各自独立的，也不会需要STE）

* 保证线程安全性时

### 可能的问题

* 线程先行持有资源锁的同时且相互等待，容易出现死锁

* Single Threaded Execution 可能导致性能降低
  * 获取锁耗费的时间
  * 线程冲突引起的等待

### 延伸

Single Threaded Execution 模式确保的是某个区域“只能由一个线程”执行。如果将其扩展开来，以确保某个区域“最多由N个线程”执行。`java.util.concurrent` 提供了表示计数信号量的`Semaphore` 类。

资源的许可个数（permits）通过`Semaphore` 类的构造方法设置。`acquire` 方法用以确保存在可用的资源。如果存在，则`acquire` 方法立即返回，同时信号量内部的资源个数减一；如无资源，线程则阻塞在`acquire` 方法内，直到出现可用的资源。

`Semaphore`的`release`方法用于释放资源。释放资源后，信号量内部的资源个数增加一；另外如果`acquire` 方法上有阻塞的线程，那么其中一个线程会被唤醒，并从`acquire` 方法返回。






