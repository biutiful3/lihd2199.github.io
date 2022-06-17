线程安全

------

在同一个群组里，我们无法让一个线程运行多个消费者，也无法让多个线程安全地共享一个消费者。按照规则，一个消费者使用一个线程。如果要在同一个消费者群组里运行多个消费者，需要让每个消费者运行在自己的线程里。最好是把消费者的逻辑封装在自己的对象里，然后使用Java的ExecutorService启动多个线程，使每个消费者运行在自己的线程上。



再均衡监听器

------

在为消费者分配新分区或移除旧分区时，可以通过消费者API执行一些应用程序代码，在调用subscribe()方法时传进去一个ConsumerRebalanceListener实例就可以了。ConsumerRebalanceListener有两个需要实现的方法。

(1) public void onPartitionsRevoked(Collection<TopicPartition> partitions)方法会在再均衡开始之前和消费者停止读取消息之后被调用。如果在这里提交偏移量，下一个接管分区的消费者就知道该从哪里开始读取了。

(2) public void onPartitionsAssigned(Collection<TopicPartition> partitions)方法会在重新分配分区之后和消费者开始读取消息之前被调用。





从特定偏移量处开始处理记录

------

如果你想从分区的起始位置开始读取消息，或者直接跳到分区的末尾开始读取消息，可以使用seekToBeginning(Collection<TopicPartition> tp)和seekToEnd(Collection<TopicPartition>tp)这两个方法。





如何退出

------

如果确定要退出循环，需要通过另一个线程调用consumer.wakeup()方法。要记住，consumer.wakeup()是消费者唯一一个可以从其他线程里安全调用的方法。

> 调用consumer.wakeup()可以退出poll()，并抛出WakeupException异常，或者如果调用consumer.wakeup()时线程没有等待轮询，那么异常将在下一轮调用poll()时抛出。我们不需要处理WakeupException，因为它只是用于跳出循环的一种方式。不过，在退出线程之前调用consumer.close()是很有必要的，它会提交任何还没有提交的东西，并向群组协调器发送消息，告知自己要离开群组，接下来就会触发再均衡，而不需要等待会话超时。

如果循环运行在主线程里，可以在ShutdownHook里调用该方法。

```java
Runtime.getRuntime().addShutdownHook(new Thread(){
    @Override
    public void run() {
        consumer.weakup();
    }
});
```

