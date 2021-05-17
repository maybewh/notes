# kafka入门笔记

## 黑马第一阶段



## 黑马第二阶段

### 1. 分区和副本机制

#### 1.1 生产者分区写入策略

​	生产者写入消息到topic，Kafka将依据不同策略将数据分配到不同的分区中，具体策略如下：

 	1. 轮询分区策略
 	2. 随机分区策略
 	3. 按key分区分配策略
 	4. 自定义分区策略

##### 1.1.1 轮询策略

​		一个分区一个分区接着放。例如：有3个分区，那么产生的第一个消息会放在第一个分区里面，产生的第二消息放在第二个分区里面，产生的第三条消息放在第三个分区，产生的第四条消息放在第一个分区，以此类推。如下图所示：

![img](http://www.louisvv.com/wp-content/uploads/2019/12/20191229155854_61528.png)

* 默认策略，最大限度保证消息分配均匀
* 如果在生产消息时，key为null，则使用轮询策略均衡地分区

##### 1.1.2 随机策略（不用）

​		每次都将消息随机分配到每个分区，在较早的版本中，默认使用该策略，但后续发现轮询策略更优，所以不再使用该策略。

##### 1.1.3  按key分配策略

​		key可以定义为一些含有特殊含义的字符串，比如客户代码、部门编号或业务ID等，同一个key会进入同一个分区。实现该策略的partition计算规则为：

```java
// 1. 先计算key的hashCode，再使用hashCode对分区数取余
List<PatitionInfo> partitions = cluster.partitionForTopic(topic);
return Math.abs(key.hashCode() % partitions.size());
```

##### 1.1.4 地理分区策略

​	该策略主要针对跨城市、跨国家的大集群。

#####	1.1.5 kafka乱序问题

​		kafka乱序问题指的是全局乱序，局部partition是有序的。

![image-20210516211242061](C:\Users\王晖\AppData\Roaming\Typora\typora-user-images\image-20210516211242061.png)

##### 1.1.6 自定义分区策略

​	继承**partitioner**类，实现接口中的partition()方法和close()方法即可。

```java
 /**
     * Compute the partition for the given record.
     *
     * topic、key、keyBytes、value和valueBytes都属于消息数据,cluster则是集群信息（比如当前Kafka集群共有多少主题、多少	 * Broker等）
     *
     * @param topic The topic name
     * @param key The key to partition on (or null if no key)
     * @param keyBytes The serialized key to partition on( or null if no key)
     * @param value The value to partition on or null
     * @param valueBytes The serialized value to partition on or null
     * @param cluster The current cluster metadata
     */
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster); 

    /**
     * This is called when partitioner is closed.
     */
    public void close();
```

### 2. 消费组Consumer Group Rebalance 

​		消费组概念存在于kafka的发布、订阅模型。一个消费组可以有多个消费者实例，而这些消费者实例共享一个id，即group id。而一个消费组中，每个分区只能被一个消费者订阅。

​		具体参考：https://www.cnblogs.com/listenfwind/p/12662968.html

####	2.1 rebalance触发的时机以及缺点

* 触发时机：
  + 消费者数量发生变化
    * 某个消费者crash（挂掉）
    * 新增消费者
  + topic数量发生变化
    * 某个topic被删除
  + partition发生了变化：新增或者删除partition

* 不良影响：所有消费者将不能消费数据

#### 2.2 消费者分区分配策略

可参考：https://my.oschina.net/u/4262150/blog/3274346

##### 2.2.1 Range范围分配策略

​		Range范围分配策略是kafka默认的分配策略，它确保每个消费者消费的分区数量是均衡的。

​		注意：Range范围分配策略是针对每个Topic的。

配置：

​		配置消费者的partition.assignment.strategy为org.apache.kafka.clients.consumer.RangeAssignor

​		算法公式：n  = 分区数量 / 消费者线程数量  m = 分区数量 % 消费者线程数量，前m个消费者分配到n+1个分区，剩余的消费者分配到N个分区

##### 2.2.2 轮询策略

RoundRobin策略的原理是将**消费组内所有消费者以及消费者所订阅的所有topic的partition按照字典序排序**，然后通过轮询算法逐个将分区以此分配给每个消费者。

##### 2.2.3 Sticky分配策略

​	该分配策略主要实现两个目的：

1.  分区的分配尽可能的均匀

2. 分区的分配尽可能的与上次分配保持一致

​    如果这两个目的发生冲突，优先实现第一个目的。

为什么要尽量保持和上次分配的分区？

​		**因为对于同一个分区而言，有可能之前的消费者和新指派的消费者不是同一个，对于之前消费者进行到一半的处理还要在新指派的消费者中再次处理一遍，这时会浪费系统资源。而使用Sticky策略就可以让分配策略具备一定的“粘性”,尽可能地让前后两次分配相同，进而减少系统资源操作的损耗。**	

####	2.3 副本机制

​		主要目的：冗余备份，避免数据丢失

##### 2.3.1 producer的ACKs参数

​		对副本关系比较大的就是ACKs，producer配置的acks参数表示当生产者生产消息的时候，写到副本的要求严格程度。它决定生产者如何在性能和可靠性之间做取舍。

* Acks为0(单分区单副本)，不等待broker确认，直接发送一条数据，性能最高。如果leader已经死亡，producer不知道，继续发送的消息就会丢失。
* 当生产者ACK配置为1时，生产者会等待leader成功接收数据并确认后才会发送下一条数据。如果leader已经死亡，但是follower尚未复制，数据就会丢失。
* 当ACK配置为-1或者all时，意味着producer得到follwer确认，才发送下一条数据。这种情况持久性最好，但是延时性最差。

