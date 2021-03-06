### 事务性写入

第二种实现端到端的恰好处理一次一致性语义的方法基于事务性写入。其思想是只将最近一次成功保存的检查点之前的计算结果写入到外部系统中去。这样就保证了在任务故障的情况下，端到端恰好处理一次语义。应用将被重置到最近一次的检查点，而在这个检查点之后并没有向外部系统发出任何计算结果。通过只有当检查点保存完成以后再写入数据这种方法，事务性的方法将不会遭受幂等性写入所遭受的重播不一致的问题。尽管如此，事务性写入却带来了延迟，因为只有在检查点完成以后，我们才能看到计算结果。

Flink提供了两种构建模块来实现事务性sink连接器：write-ahead-log（WAL，预写式日志）sink和两阶段提交sink。WAL式sink将会把所有计算结果写入到应用程序的状态中，等接到检查点完成的通知，才会将计算结果发送到sink系统。因为sink操作会把数据都缓存在状态后段，所以WAL可以使用在任何外部sink系统上。尽管如此，WAL还是无法提供刀枪不入的恰好处理一次语义的保证，再加上由于要缓存数据带来的状态后段的状态大小的问题，WAL模型并不十分完美。

与之形成对比的，2PC sink需要sink系统提供事务的支持或者可以模拟出事务特性的模块。对于每一个检查点，sink开始一个事务，然后将所有的接收到的数据都添加到事务中，并将这些数据写入到sink系统，但并没有提交（commit）它们。当事务接收到检查点完成的通知时，事务将被commit，数据将被真正的写入sink系统。这项机制主要依赖于一次sink可以在检查点完成之前开始事务，并在应用程序从一次故障中恢复以后再commit的能力。

2PC协议依赖于Flink的检查点机制。检查点屏障是开始一个新的事务的通知，所有操作符自己的检查点成功的通知是它们可以commit的投票，而作业管理器通知一个检查点成功的消息是commit事务的指令。于WAL sink形成对比的是，2PC sinks依赖于sink系统和sink本身的实现可以实现恰好处理一次语义。更多的，2PC sink不断的将数据写入到sink系统中，而WAL写模型就会有之前所述的问题。

|                | 不可重置的源 | 可重置的源                                               |
|----------------|--------------|----------------------------------------------------------|
| any sink       | at-most-once | at-least-once                                            |
| 幂等性sink     | at-most-once | exactly-once（当从任务失败中恢复时，存在暂时的不一致性） |
| 预写式日志sink | at-most-once | at-least-once                                            |
| 2PC sink       | at-most-once | exactly-once                                             |

