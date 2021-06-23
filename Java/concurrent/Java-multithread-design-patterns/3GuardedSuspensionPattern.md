# Guarded Suspension Pattern

## Chap3

### 介绍

Guarded Suspension 模式，如果执行时碰到问题，就让执行处理的线程等待。这一模式让线程等待来保证实例的安全性。

### 拓展

* 等待端的逻辑：

```
while(!ready){
    使用wait进行等待
}
执行目标处理
```

* 唤醒端的处理

```
ready = true;
notifyAll();
```

* 使用 `java.util.concurrent.LinkedBlockingQueue`

LinkedBlockingQueue 属于线程安全的队列，类属的 `take` 和 `put` 方法已经考虑了互斥处理。它在实现中已经使用了Guarded Suspension 模式，能保证线程安全。

