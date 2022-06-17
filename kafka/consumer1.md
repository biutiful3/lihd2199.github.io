### 消费逻辑

------

1. 配置消费者客户端参数及创建相应的消费者实例；
2. 订阅主题；
3. 拉取消息并消费；
4. 提交消费位移；
5. 关闭消费者实例；

```java
private static final Properties PROPS = new Properties();

/**
 * 设置参数
 */
static {
    PROPS.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, servers);
    PROPS.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, keyStringSerializer);
    PROPS.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, valueStringSerializer);
    PROPS.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
}

public static void main(String[] args) {

    //step1 初始化 KafkaConsumer
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(PROPS);
    //step2 订阅主题
    consumer.subscribe(Arrays.asList(topic));

    try {
        while (true) {
            //step3 拉取消息并消费  可以指定时间，也可以立刻返回
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
          	//step4 遍历处理消息
            for (ConsumerRecord<String, String> record : records) {
                log.info(record.key());
                log.info(record.value());
                log.info(record.topic());
                log.info(String.valueOf(record.partition()));
                log.info(String.valueOf(record.offset()));
            }
        }
    } catch (Exception e) {
        log.error("获取消息出错", e);
    } finally {
        //step5 关闭consumer，会直接触发再均衡
        consumer.close();
    }

}
```

### 必要参数

------

bootstrap.servers：指 定 连 接 Kafka 集 群 所需 的 broker 地 址 清 单

> 具 体 内 容 形 式 为host1：port1，host2：post，可以设置一个或多个地址，中间用逗号隔开，参数的默认值为“”。这里并非需要设置集群中全部的broker地址，消费者会从现有的配置中查找到全部的Kafka集群成员。这里设置两个以上的broker地址信息，当其中任意一个宕机时，消费者仍然可以连接到Kafka集群上。

group.id：消费者隶属的消费组的名称，默认值为“”

key.deserializer 和 value.deserializer：与生产者客户端KafkaProducer中的key.serializer和value.serializer参数对应

> 消费者从broker端获取的消息格式都是字节数组（byte[]）类型，所以需要执行相应的反序列化操作才能还原成原有的对象格式。这两个参数分别用来指定消息中key和value所需反序列化操作的反序列化器，这两个参数无默认值，这里必须填写反序列化器类的全限定名。



