### 整体架构

------

![](./img/producer架构图.jpeg)

​		整个生产者客户端由两个线程协调运行，这两个线程分别为主线程和Sender线程（发送线程）。在主线程中由KafkaProducer创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息收集器（RecordAccumulator）中。Sender 线程负责从RecordAccumulator中获取消息并将其发送到Kafka中。

> RecordAccumulator 主要用来缓存消息以便 Sender 线程可以批量发送，进而减少网络传输的资源消耗以提升性能。RecordAccumulator 缓存的大小可以通过生产者客户端参数buffer.memory 配置，默认值为 33554432B，即 32MB。

```java
ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches
```

​		主线程中发送过来的消息都会被追加到RecordAccumulator的某个双端队列（Deque）中，在 RecordAccumulator 的内部为每个分区都维护了一个双端队列，队列中的内容就是ProducerBatch，即 Deque＜ProducerBatch＞。消息写入缓存时，追加到双端队列的尾部；Sender读取消息时，从双端队列的头部读取。

> 注意ProducerBatch不是ProducerRecord，ProducerBatch中可以包含一至多个 ProducerRecord。通俗地说，ProducerRecord 是生产者中创建的消息，而ProducerBatch是指一个消息批次，ProducerRecord会被包含在ProducerBatch中，这样可以使字节的使用更加紧凑。将较小的ProducerRecord拼凑成一个较大的ProducerBatch，也可以减少网络请求的次数以提升整体的吞吐量。

​		当一条消息（ProducerRecord）流入RecordAccumulator时，会先寻找与消息分区所对应的双端队列（如果没有则新建），再从这个双端队列的尾部获取一个 ProducerBatch（如果没有则新建），查看 ProducerBatch 中是否还可以写入这个ProducerRecord，如果可以则写入，如果不可以则需要创建一个新的ProducerBatch。

> 在新建ProducerBatch时评估这条消息的大小是否超过batch.size参数的大小，如果不超过，那么就以 batch.size 参数的大小来创建ProducerBatch，这样在使用完这段内存区域之后，可以通过BufferPool 的管理来进行复用；如果超过，那么就以评估的大小来创建ProducerBatch，这段内存区域不会被复用。

​		Sender 从 RecordAccumulator 中获取缓存的消息之后，会进一步将原本＜分区，Deque＜ProducerBatch＞＞的保存形式转变成＜Node，List＜ ProducerBatch＞的形式，其中Node表示Kafka集群的broker节点

> 对于网络连接来说，生产者客户端是与具体的broker节点建立的连接，也就是向具体的 broker 节点发送消息，而并不关心消息属于哪一个分区；而对于KafkaProducer的应用逻辑而言，我们只关注向哪个分区中发送哪些消息，所以在这里需要做一个应用逻辑层面到网络I/O层面的转换。

​		请求在从Sender线程发往Kafka之前还会保存到InFlightRequests中，InFlightRequests保存对象的具体形式为Map＜NodeId，Deque＜Request＞＞，它的主要作用是缓存了已经发出去但还没有收到响应的请求（NodeId 是一个 String类型，表示节点的 id 编号）。