# Chap5

## Producer-Consumer Pattern

### 介绍

`Producer` 生产者，指的是生成数据的线程；`Consumer` 消费者，指的是使用数据的线程。
生产者安全地将数据交给消费者。看似简单的操作，但生产者和消费者以不同的线程运行时，两者之间的的速度差异便会引起问题。

Producer-Consumer模式在生产者和消费者之间引入了一个“桥梁角色”，它用以消除线程中的处理速度的差异。

一般来说，该模式中的生产者和消费者可以用多个。而当两者都只有一个的时候，我们称之为“Pipe模式”(操作系统中的pipe)。

### 参与的角色

```
---------              ---------             ---------
Producer                Channel              Consumer
-channel                *put()               -channel
                        *take()
---------              ---------             ---------
    |                     ◇                      |
    |                     |                      |
    |                     | Contains ↓           |
    |                     ↓                      |
    | Creates →        ---------           ← Use |
    |——————>           |  Data |          <——————|
                       ---------
```
### 拓展

`Channel` 角色实现线程间的互斥处理，确保Producer正确地将`Data`传递给Consumer。

#### 直接传递的问题

Producer-Consumer模式为了从Producer向Consumer传递Data，在中间设置了一个Channel的角色。继续探讨下：
* 直接调用方法
  Consumer想要获取Data数据而执行某些处理，如果Producer直接调用Consumer的方法，那么执行处理的就不是Consumer的线程，而是Producer的线程了。

* 数据插入到Channel中
  Producer将Data直接传递给Channel之后无需等待Consumer对Data的处理，可以立即开始准备下一个Data，即可以在Channel能存储的范围之内持续创建Data。

#### Channel剩余空间的问题

Channel中存储的实例个数一般是有上限的，如果使用Guarded suspension模式采用LinkedList作为存储的容器，那么存储个数可能就没有上限了。或许得过了很长一段时间后，会报出内存不足，也就无法创建Data实例。

#### 以什么顺序访存Data
Channel 角色从Producer接收Data，并传递给Consumer。当Channel有多个Data时，应该按什么顺序传递呢？

* 队列(Queue) - 先接收先传递
* 栈(Stack) - 后接收先传递
* 优先队列(Priority Queue)- "优先"的先传递

#### Consumer 只有一个时会怎么样

Consumer只有一个，即处理Channel中Data的线程只有一个。如果Consumer有多个，还要注意不能让Consumer线程之间相互影响；而如果只有一个Consumer线程，相当于将多线程的处理放进了单线程中。

在Swing（JFC）框架中，事件处理部分就是利用的这种方法（多个Producer对应一个Consumer）。执行Swing事件处理的线程成为事件分发线程（Event dispatching thread）。这个线程相当于从Channel的事件队列中取出事件并进行处理的Consumer角色。事件分发线程只有一个。这样的设计使得事件处理的程序变得简单高效。

### 延伸1 理解InterruptedException 异常

#### 可能耗费时间但可以取消

Java多线程中注意到有些方法后面加了 `throws InterruptedException`，则表明方法（或是该方法进一步调用的方法中）可能会抛出 `InterruptedException` 异常。这有两层意思：
* 是"耗费时间"的方法
* 是"可以取消"的方法

#### 加了throwsInterruptedException的方法

Java标准类库中加了throwsInterruptedException的典型方法主要有：
* `java.lang.Object`类的`wait`方法
* `java.lang.Thread`类的`sleep`方法
* `java.lang.Thread`类的`join`方法

#### sleep 和 interrupt方法
以sleep方法为例，线程A执行了以下的方法：
`Thread.sleep(6000);`
假定想取消线程A的暂停状态，由其它线程调用线程的实例方法：
`A.interrupt();`

当执行interrupt方法时，线程并不需要获取Thread A实例的锁。interrupt方法被调用后，正在sleep的线程会终止暂停状态，抛出`InterruptedException` 异常。这样线程A的控制权就会转移到捕获该异常的catch语句块中（想想可作为线程退出的方式之一?!）。

#### interrupt方法只是改变了中断状态

也许有人会认为“当调用interrupt方法时，调用对象的线程会抛出`InterruptedException` 异常”，这其实是一种误解。实际上interrupt方法只是改变了中断状态而已。所谓的中断状态（interrupted status）是一种用于表示线程是否被中断的状态。

当线程A执行了sleep方法而停止运行时，另一线程调用了A.interrupt方法。这时，线程A确实会抛出`InterruptedException` 异常，但这其实是sleep方法内部对线程的中断状态进行了检查，而抛出的异常。

如果没有调用sleep，wait，join等方法，且没有编写检查线程中断状态并抛出异常的逻辑代码，就不会抛出`InterruptedException` 异常。

`isInterrupted()`方法用以检查线程的中断状态，它不改变中断状态的值。

`interrupted()`方法检查并且清除线程的中断状态。区别于`interrupt`方法，后者是让线程中断的方法。

### 延伸2 java.util.concurrent包和Producer-Consumer模式

#### java.util.concurrent包中的队列

java.util.concurrent包提供了包括BlockingQueue接口以及实现类，它们相当于Producer-Consumer模式中的Channel角色。

* BlockingQueue接口 - 阻塞队列

* ArrayBlockingQueue类 - 基于数组的BlockingQueue

* LinkedBlockingQueue类 - 基于链表的BlockingQueue

* PriorityBlockingQueue类 - 带有优先级的BlockingQueue

* DelayQueue类 - 一定时间之后才可以take的BlockingQueue

* SynchronousQueue类 - 直接传递的BlockingQueue

* ConcurrentLinkedQueue类 - 元素个数没有最大限制的线程安全队列

#### 使用java.util.concurrent.Exchanger类交换缓冲区






