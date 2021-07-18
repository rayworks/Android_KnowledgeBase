# 笔记说明

笔记内容摘选自[图灵2017版的《图解Java多线程设计模式》](https://www.ituring.com.cn/book/1812)。

按章节对应索引如下：

[序章1　Java线程](./0Java%20thread.md)

[第1章　Single Threaded Execution模式——能通过这座桥的只有一个人](./1Single-Threaded-Execution%20pattern.md)

[第2章　Immutable模式——想破坏也破坏不了](./2Immutable.md)

[第3章　Guarded Suspension模式——等我准备好哦](./3GuardedSuspensionPattern.md)

[第4章　Balking模式——不需要就算了](./4BalkingPattern.md)

[第5章　Producer-Consumer模式——我来做，你来用](./5Producer-Consumer%20pattern.md)

[第6章　Read-Write Lock模式——大家一起读没问题，但读的时候不要写哦](./6Read-Write%20Lock%20pattern.md)

[第7章　Thread-Per-Message模式——这项工作就交给你了](./7Thread-Per-MessagePattern.md)

[第8章　Worker Thread模式——工作没来就一直等，工作来了就干活](./8WorkerThreadPattern.md)

[第9章　Future模式——先给您提货单](./9FuturePattern.md)

[第10章 Two-Phase Termination模式——先收拾房间再睡觉](./10Two-PhraseTerminationPattern.md)

[第11章  Thread-Specific Storage模式——一个线程一个储物柜](./11Thread-Specific-StoragePattern.md)

[第12章 Active Object模式——接收异步消息的主动对象](./12ActiveObjectPattern.md)

另外节选了书中的[部分Sample代码](./src)。作者在讲解某个模式时，一般先会尝试先用自己的方式实现，然后同时对比利用了 `Java concurrent` 包中(JUC)相关的类以后的代码实现。这无疑大大减缓了一般读者对`concurrent`包学习的距离感，也让读者能逐渐理会到系统类库设计时所做的考量。

更多的随书资源，请移步[图书主页](https://www.ituring.com.cn/book/1812)。