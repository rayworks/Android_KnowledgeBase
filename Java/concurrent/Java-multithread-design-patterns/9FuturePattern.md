# Chap 9

## Future Pattern

### 介绍

`Future`本意是未来，期货的意思。假设有个方法需要花费很长的时间才能得到获取运行的结果，那么与其一直等待结果，不如先拿出一张“提货单”。而获取提货单并不耗时，这里的“提货单”即可称为`Future`角色。

获取Future的线程会在稍后使用它来获取运行的结果，如果结果已经出来了，那么直接领取即可；如果运行的结果还没有出来，则需要等待运行的结果。

Future是提货单，是“未来”可以转化为实物的凭证。

### 拓展

#### 吞吐量会提高？

关注到“负责长时间处理的是哪个线程”，单CPU的Java虚拟机上，如果只是进行多线程的计算这样分担任务，是无法提高吞吐量的。但是如果结合了输入输出（IO）一起考虑，情况就会有所不同了，在程序进行磁盘读写操作时，CPU处于空闲状态，这时的空闲时间分配给其它线程，让它们进行处理却是可以提高吞吐量的（注：使用`java.nio`包中的非阻塞IO nonblocking IO 时，可以编写线程非阻塞IO的程序）。

#### 异步方法调用的返回值

Thread-Per-Message 模式通过在方法中创建一个线程，模拟实现了异步方法调用。它实现了即便被调用的方法处理没有全部终止时，调用方的处理仍然可以继续向前执行。而只是用上述模式是无法获取异步处理结果的，而可以通过使用Future模式来“稍后设置处理结果”，从而操作异步方法调用的“返回值”。

#### “准备返回值”和“使用返回值”的分离

Worker Thread pattern中分离了“准备方法的返回值”和“使用方法的返回值”。将伴随方法的一连串调用分解开来，把各个处理（启动，执行，准备返回值，使用返回值）分配给各个线程。

#### 变种：会发生变化的Future

通常情况下，返回值会被设置到Future中仅且仅有一次，但某些场景下，我们会考虑反复设置“返回值”。

比如，在通过网络获取图像数据的时候，可以先获取图像的长宽，接着取模糊的图像数据，最后取清晰的图像数据。

这也是Future模式的一个变种。

#### 回调与Future模式

如果想要等待处理完成后获取返回值，还可以考虑采用回调的处理方式。即在处理完成后，由Host启动的线程去调用Client角色的方法。不过需要确保Client角色中编写的逻辑处理是线程安全的。

### 延伸：JUC包与Future模式

* java.util.concurrent.Callable 接口

  `Callable` 接口将“返回值的某种处理的调用”抽象化了，该接口声明了`call`方法。`call`方法与Runnable 的`run`方法相似，但是call方法有返回值。

* java.util.concurrent.Future 接口

  该接口声明了`get`方法，但没有设置该值的方法；同时还声明了用于中断运行的`cancel`方法。

* java.util.concurrent.FutureTask 类

  该类实现了Future 接口和Runnable接口，并且提供了设置值的`set`方法以及设置异常的`setException` 方法。