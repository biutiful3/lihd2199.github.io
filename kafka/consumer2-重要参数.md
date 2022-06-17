heartbeat.interval.ms：发送心跳的时间间隔

> 从字面上看，它就是设置了心跳的间隔时间，但这个参数的真正作用是控制重平衡通知的频率。如果你想要消费者实例更迅速地得到通知，那么就可以给这个参数设置一个非常小的值，这样消费者就能更快地感知到重平衡已经开启了。

session.timeout.ms：指定了消费者在被认为死亡之前可以与服务器断开连接的时间，默认是3s

> 如果消费者没有在session.timeout.ms指定的时间内发送心跳给群组协调器，就被认为已经死亡，协调器就会触发再均衡，把它的分区分配给群组里的其他消费者。
>
> 该属性与heartbeat.interval.ms紧密相关。heartbeat.interval.ms指定了poll()方法向协调器发送心跳的频率，session.timeout.ms则指定了消费者可以多久不发送心跳。所以，一般需要同时修改这两个属性，heartbeat.interval.ms必须比session.timeout.ms小，一般是session.timeout.ms的三分之一。

auto.offset.reset：指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下（因消费者长时间失效，包含偏移量的记录已经过时并被删除）该作何处理，默认值是latest。

enable.auto.commit：消费者是否自动提交偏移量，默认值是true

max.poll.records：用于控制单次调用call()方法能够返回的记录数量

fetch.min.bytes：Consumer在一次拉取请求中能从Kafka中拉取的最小数据量，默认值为1（B）

> 如果返回给Consumer的数据量小于这个参数所配置的值，那么它就需要进行等待，直到数据量满足这个参数的配置大小。可以适当调大这个参数的值以提高一定的吞吐量，不过也会造成额外的延迟（latency），对于延迟敏感的应用可能就不可取了。

fetch.max.wait.ms：用于指定Kafka的等待时间，默认值为500（ms）

> 如果Kafka仅仅参考fetch.min.bytes参数的要求，那么有可能会一直阻塞等待而无法发送响应给 Consumer，显然这是不合理的。

fetch.max.bytes：Consumer在一次拉取请求中从Kafka中拉取的最大数据量，默认值为52428800（B）50MB

max.partition.fetch.bytes：指定了服务器从每个分区里返回给消费者的最大字节数，默认值是1MB

> 如果一个主题有20个分区和5个消费者，那么每个消费者需要至少4MB的可用内存来接收记录。在为消费者分配内存时，可以给它们多分配一些，因为如果群组里有消费者发生崩溃，剩下的消费者需要处理更多的分区。max.partition. fetch.bytes的值必须比broker能够接收的最大消息的字节数（通过max.message.size属性配置）大，否则消费者可能无法读取这些消息，导致消费者一直挂起重试。在设置该属性时，另一个需要考虑的因素是消费者处理数据的时间。消费者需要频繁调用poll()方法来避免会话过期和发生分区再均衡，如果单次调用poll()返回的数据太多，消费者需要更多的时间来处理，可能无法及时进行下一个轮询来避免会话过期。如果出现这种情况，可以把max.partition.fetch.bytes值改小，或者延长会话过期时间。

 receive.buffer.bytes和send.buffer.bytes：socket在读写数据时用到的TCP缓冲区也可以设置大小。如果它们被设为-1，就使用操作系统的默认值
