# Chap 12

## Active Object Pattern

### 介绍

`Active Object` 主动对象，一般指有自己特有的线程。如`Java`的`java.lang.Thread`就是一种主动对象。该模式下的主动对象会通过自己的特有的线程在合适的时机处理从外部接收到的异步消息。`Active Object`模式有时也被称为Actor模式。

### 参与的角色

* ActiveObject 定义了主动对象的接口

* ActiveObjectFactory 创建主动对象的类

* ActiveObjectImpl 主动对象的实现类

* Proxy 将方法调用转换为MethodRequest对象的类 （实现了ActiveObject的接口）

* SchedulerThread 调用execute方法处理MethodRequest对象的类

* ActivationQueue 按顺序保存MethodRequest对象的类

* MethodRequest 表示请求的抽象类

* Result 表示执行结果的抽象类

* FutureResult 在Future模式下表示执行结果的类

* RealResult 表示实际的执行结果的类

* Servant 执行实际处理的类（实现了ActiveObject接口）

执行flow：

> The proxy asks the schedulerThread to process the concrete MethodRequest and returns immediately (a `FutureResult` obejct if needed), then the request will be added into the queue of schedulerThread. Following the Producer-Consumer pattern, the request will the taken out and executed. During the execution phrase, the Servant gets involved and processes the actual task.

### 拓展

#### 到底做了什么事情

ActiveObject包中的所有角色相互协作，组成了一个主动对象：

* 定义了ActiveObject的API

* 接受异步消息：proxy将方法调用转换为MethodRequest角色后保存在ActivationQueue中

* 与调用者Client运行于不同的线程：由Scheduler角色提供

* 执行处理：由Servant角色单线程执行处理

* 返回的返回值： Future类型值

#### 运用模式时需要考虑问题的粒度

Active Object 模式组成要素较多，是一个非常庞大的模式。因此在运用该模式时，需要注意问题的粒度。问题的粒度指的是问题复杂度的大小，即解决问题的每个处理到底有多大。

Active Object 模式中无法忽略的Proxy创建ConcreteMethodRequest实例，以及与ActivationQueue交互时产生的性能开销。当问题粒度较小时，与使用ActivationQueue将执行的处理线程统一为一个相比，或许使用Guarded Suspension模式性能会更高。

#### 关于并发性

* Proxy 即使被多个线程调用也没有问题{{ concurrent }}

* Servant 只能被一个线程调用 {{ sequential }}

#### 主动对象之间的交互

本章以Client角色使用主动对象为例讲解了Active Object模式，但实际上也可以编写多个主动对象，然后让他们之间进行交互。

#### 通往分布式 -- 从跨越线程界线变为跨越计算机界线

在Active Object 模式中，“方法的调用”部分运行在Client角色的线程中，而“方法的执行”部分运行在Scheduler角色的线程中，这也是Chap 8 中讲解过的“调用与执行的分离”。

如果按线程分离开来，那么较为容易的是将运行线程的计算机也分离开，把执行invocation的机器与执行execution的机器分离开，然后用网络将它们连接起来。那网络之间相互传输的是什么呢？就是MethodRequest类型的对象和Result类型的对象。由于方法的调用和设置返回值都可以转换为对象这种“有形”的东西，所以可以通过网络交互。

Java中有一种与Active Object模式相关的技术叫作Remote Method Invocation（远程方法调用 RMI）。[RMI](https://docs.oracle.com/javase/tutorial/rmi/overview.html)是一种可以在本机调用，然后在网络远端计算机上执行方法的技术。为了能在网络间传输对象，RMI使用了Java的序列化（serialization）技术。

### 延伸：java.util.concurrent包与Active Object模式

| 类和接口    | 内容 |
| :----- | :----- |
| `ExecutorService`            | 用以提交请求的接口 （替代 -> SchedulerThread, ActivationQueue） |
| `Callable` | 将获取返回值的调用（call）抽象化后的接口（替代 -> MethodRequest 的具体类型） |
| `Runnable` | 将不获取返回值的调用（run）抽象化后的接口（替代 ->  MethodRequest 的具体类型|
| `Future` | 表示返回值的接口（替代 -> Result、FutureResult、RealResult）|

### 备注

* [RMI 和 RPC](https://zhuanlan.zhihu.com/p/188432060)