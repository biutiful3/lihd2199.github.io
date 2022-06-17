### 消息发送

------

1. 配置生产者客户端参数及创建相应的生产者实例
2. 构建待发送的消息
3. 发送消息
4. 关闭生产者实例

```java
private static final Properties PROPS = new Properties();
/**
* 设置参数
*/
static {
    PROPS.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, servers);
    PROPS.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, keyStringSerializer);
    PROPS.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, valueStringSerializer);
    PROPS.put(ProducerConfig.RETRIES_CONFIG, retries);
    PROPS.put(ProducerConfig.BATCH_SIZE_CONFIG, batchSize);
    PROPS.put(ProducerConfig.LINGER_MS_CONFIG, linger);
    PROPS.put(ProducerConfig.BUFFER_MEMORY_CONFIG, bufferMemory);
}

public static void main(String[] args) {

		//step1 初始化 KafkaProducer
    KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(PROPS);
		//step2 构建待发送的消息
    ProducerRecord<String, String> record = new ProducerRecord<>(
            topic, key, message);
  	//step3 发送消息
    kafkaProducer.send(record, (recordMetadata, exception) -> {
        if (null != exception) {
            log.error("kafka发送消息失败 ", exception);
            //todo
        }
    });
  	//step4 关闭生产者实例
 		kafkaProducer.close();

}
```

发送消息主要有三种模：

发后即忘（fire-and-forget）

```java
kafkaProducer.send(record);
```

同步（sync）

```java
kafkaProducer.send(record).get();
```

异步（async）

```java
kafkaProducer.send(record, (recordMetadata, exception) -> {
    if (null != exception) {
        log.error("kafka发送消息失败 ", exception);
        //todo
    }
})
```

### 必要参数

------

 bootstrap.servers：用来指定生产者客户端连接Kafka集群所需的broker地址清单

> 具体的内容格式为host1：port1，host2：port2，可以设置一个或多个地址，中间以逗号隔开，此参数的默认值为“”。注意这里并非需要所有的broker地址，因为生产者会从给定的broker里查找到其他broker的信息。不过建议至少要设置两个以上的broker 地址信息，当其中任意一个宕机时，生产者仍然可以连接到 Kafka集群上。

key.serializer 和 value.serializer：两个参数分别用来指定key和value序列化操作的序列化器，这两个参数无默认值



### 序列化

------

生产者需要用序列化器（Serializer）把对象转换成字节数组才能通过网络发送给Kafka。而在对侧，消费者需要用反序列化器（Deserializer）把从 Kafka 中收到的字节数组转换成相应的对象。

> 生产者使用的序列化器和消费者使用的反序列化器是需要一一对应的，如果生产者使用了某种序列化器，比如StringSerializer，而消费者使用了另一种序列化器，比如IntegerSerializer，那么是无法解析出想要的数据的。



### 分区器

------

如果消息ProducerRecord中没有指定partition字段，那么就需要依赖分区器，根据key这个字段来计算partition的值。分区器的作用就是为消息分配分区。Kafka中提供的默认分区器是org.apache.kafka.clients.producer.internals.DefaultPartitioner。

> 在默认分区器 DefaultPartitioner 的实现中，如果 key 不为 null，会对 key 进行哈希，拥有相同key的消息会被写入同一个分区。如果key为null，那么消息将会以轮询的方式发往主题内的各个可用分区。
>
> 通过实现org.apache.kafka.clients.producer.Partitioner接口实现自定义分区器
>
> 在不改变主题分区数量的情况下，key与分区之间的映射可以保持不变。不过，一旦主题中增加了分区，那么就难以保证key与分区之间的映射关系了。



### 拦截器

生产者拦截器既可以用来在消息发送前做一些准备工作，比如按照某个规则过滤不符合要求的消息、修改消息的内容等，也可以用来在发送回调逻辑前做一些定制化的需求，比如统计类工作。

> 通过实现org.apache.kafka.clients.producer.ProducerInterceptor接口自定义拦截器