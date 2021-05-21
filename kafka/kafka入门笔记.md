# kafka入门笔记

## 黑马第一阶段

> 整体参考：https://www.pianshen.com/article/2642435158/

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

#### 2.4 Kafka--consumer高级API与低级API

> 参考： https://www.cnblogs.com/alexzhang92/p/10894800.html

* 高级API
  + 优点：写起来简单，不需要自行去管理offset，系统通过zookeeper自行管理，不需要管理分区、副本等情况，系统自动管理。消费者断线会自动根据上一次记录在zookeeper中的offset去接着获取数据（默认一分钟更新一次zookeeper中存的offset）
  + 缺点：不能自行控制offset、不能细化控制如分区、副本、zk等

* 低级API

  + 优点：开发者自己能控制offset，想从哪里读取就从哪里读取。自行控制连接分区，对分区自定义进行负载均衡，对zookeeper的依赖性降低（如对offset可以自行决定存储在哪里）

  + 缺点：太过复杂，需要自行控制offset，连接哪个分区，找到分区leader等。


高级Producer
```java
import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;
import kafka.serializer.StringEncoder;
 
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Properties;

public class KafkaProducer extends Thread {
    
    private String topic;
    
    private SimpleDateFormat sdf = new SimpleDateFormat("MM-dd hh:mm:ss");
    
    public KafkaProducer(String topic) {
        super();
        this.topic = topic;
    }
    
    @Override
    public void run() {
        Producer<String, String> producer = createProducer();
        long i = 0;
        while(true) {
            i++;
            long now = System.currentTimeMills();
            KeyedMessage<String, String> message = new KeyedMessage<String, String>(topic, sdf.format(new Date(now))+"_"+i+"");
            try {
            	Thread.sleep(1000);
        	} catch(InterruptedException e) {
            	e.printStackTrace();
        	}
        }

    }
    
    private Producer<String, String> createProducer() {
        Properties properties = new Properties();
        properties.put("metadata.broker.list", "192.168.110.81:9092, 192.168.110.82:9092, 192.168.110.83:9092");
        properties.put("serializer.class", StringEncoder.class.getName());
        properties.put("zookeeper.connect", "nnn1:2181,nnn2:2181,nslave1:2181");
        return new Producer<String,String>(new ProducerConfig(properties));
    }
}
```

高级consumer

```java
import kafka.consumer.Consumer;
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
 
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;

public class KafkaConsumer extends Thread {
    
    private String topic;
    
    private ConsumerConnector consumer;
    
    public KafkaConsumer(String topic) {
        super();
        this.topic = topic;
        consumer = createConsumer();
    }
    
    public void shutDown() {
        if (consumer != null) {
            consumer.shutdown();
        }
    }
    
    @Override
    public void run() {
        Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
        topicCountMap.put(topic, 1);
        
        Map<String, List<KafkaStream<byte[], byte[]>>> messageStream = 
            consumer.createMessageStreams(topicCountMap);
        KafkaStream<byte[], byte[]> stream = messageStream.get(topic).get(0);
        ConsumerIterator<byte[], byte[]>iterator = steam.iterator();
        while(iterator.hasNext()) {
            String message = new String(iterator.next().message());
            System.out.println(message);
        }        
    }
    
    private ConsumerConnector createConsumer() {
        Properties properties = new Properties();
        properties.put("zookeeper.connect", "nnn1:2181,nnn2:2181,nslave1:2181");
        properties.put("group.id", "testsecond");
        return Consumer.createJavaConsumerConnector(new ConsumerConfig(properties));
    }
}
```

低级API示例

```java
import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
 
import kafka.api.FetchRequest;
import kafka.api.FetchRequestBuilder;
import kafka.api.PartitionOffsetRequestInfo;
import kafka.cluster.BrokerEndPoint;
import kafka.common.ErrorMapping;
import kafka.common.TopicAndPartition;
import kafka.javaapi.FetchResponse;
import kafka.javaapi.OffsetResponse;
import kafka.javaapi.PartitionMetadata;
import kafka.javaapi.TopicMetadata;
import kafka.javaapi.TopicMetadataRequest;
import kafka.javaapi.consumer.SimpleConsumer;
import kafka.message.MessageAndOffset;

public class SimpleExample {
    
    private List<String> m_replicaBrokers = new ArrayList<String>();
    
    public SimpleExample() {
        m_replicaBrokers = new ArrayList<String>();
    }
    
    publc static void main(String[] args) {
        
        SimpleExample example = new SimpleExample();
        // 最大读取消息数量
        long maxReads = Long.parseLong("3");
        // 要订阅的topic
        String topic = "test1";
        // 要查找的分区
        int partition = Integer.parseInt("0");
        // broker节点的IP
        List<String> seeds = new ArrayList<String>();
        seeds.add("192.168.110.81");
        seeds.add("192.168.110.82");
        seeds.add("192.168.110.83");
        
        // 端口
        int port = Integer.parseInt("9092");
        try {
            example.run(maxReads, topic, partition, seeds, port);
        } catch(Exception e) {
            System.out.println("Oops:" + e);
            e.printStackTrace();
        }
	}
    
    pubic void run(long a_maxReads, String a_topic, int a_partition, List<String> a_seedBrokers, int a_port) throws Exception {
        // 获取指定Topic partition的元数据
        PartitionMetadata metadata = findLeader(a_seedBrokers, a_port, a_topic, a_partition);
        if (metadata == null) {
            System.out.println("Can't find metadata for Topic and Partition. Exiting");
            return;
        }
        
        if (metadata.leader() == null) {
            System.out.println("Can't find Leader for Topic and Partition. Exiting");
            return;
        }
        
        String leadBroker = metadata.leader().host();
        String clientName = "Client_" + a_topic + "_" + a_partition;
        
        SimpleConsumer simplerConsumer = new SimpleConsumer(leadBroker, a_port, 100000, 64*1024, clientName);
        
        long readOffset = getLastOffset(consumer, a_topic, a_partition, kafka.api.OffsetRequest.EarliestTime(), clientName);
        int numErrors = 0;
        while(a_maxReads > 0) {
            if (consumer == null) {
                consumer = new SimpleConsumer(leadBroker, a_port, 100000, 64*1024,clientName);
            }
            
            FetchRequest req = new FetchRequestBuilder().clientId(clientName).addFetch(a_topic, a_partition, readOffset, 10000).build();
            FetchResponse resp = new consumer.fetch(req);
            
            if (fetchResponse.hasError()) {
                numErrors++;
                //something went wrong
                short code = fetchResponse.errorCode(a_topic, a_partition);
                System.out.println("Error fetching data from the Broker:" +
                                  leadBroker + "Reason: " + code);
                if (numErrors > 5) {
                    break;
                }
                
                if (code == ErrorMapping.OffsetOutOfRangeCode()) {
                    // We asked for an invalid offset.For simple case ask for 
                    // the last element to reset
                    readOffset = getLastOffsetconsumer, a_topic, a_partition, kafka.api.OffsetRequest.LatestTime(), clientName);
                    continue;
                }
                consumer.close();
                consumer == null;
                leadBroker = findNewLeader(leadBroker, a_topic, a_partition, a_port);
                continue;
            }
            
            numErrors = 0;
            
            long numRead = 0;
            for (MessageAndOffset messageAndOffset : fetchResponse.messageSet(a_topic, a_partition)) {
                long currentOffset = messageAndOffset.offset();
                if (currentOffset < readOffset) {
                    System.out.println("Found an old offset: " + currentOffset + " Expecting: " + readOffset);
                    continue;
                }
                
                readOffset = messageAndOffset.nextOffset();
                ByteBuffer payload = messageAndOffset.message().payload();
                
                byte[] bytes = new byte[payload.limit()];
                payload.get(bytes);
                System.out.println(String.valueOf(messageAndOffset()) + ": " + new String(bytes, "UTF-8"));
                numRead++;
                a_maxReads--;
            }
            
            if (numRead == 0) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ie) {
                    ie.printStackTrace();
                }
            }
        }
        if (consumer != null) {
            consumer.close();
        }
    }
    
    public static long getLastOffset(SimpleConsumer consumer, String topic, int partition, long whichTime, String clientName) {
        
        TopicAndPartition topicAndPartition = new TopicAndPartition(topic, partition);
        Map<TopicAndPartition, PartitionOffsetRequestInfo> requestInfo = new HashMap<TopicAndPartition, PartitionOffsetRequestInfo>();
        requestInfo.put(topicAndPartition, new PartitionOffsetRequestInfo(whichTime, 1));
         
        kafka.javaapi.OffsetRequest request = new kafka.javaapi.OffsetRequest(requestInfo, kafka.api.OffsetRequest.CurrentVersion(), clientName);
        OffsetResponse response = consumer.getOffsetsBefore(request);
 
        if (response.hasError()) {
            System.out.println("Error fetching data Offset Data the Broker. Reason: " + response.errorCode(topic, partition));
            return 0;
        }
        long[] offsets = response.offsets(topic, partition);
        return offsets[0];
        
    }
    
        private String findNewLeader(String a_oldLeader, String a_topic, int a_partition, int a_port) throws Exception {
        for (int i = 0; i < 3; i++) {
            boolean goToSleep = false;
            PartitionMetadata metadata = findLeader(m_replicaBrokers, a_port, a_topic, a_partition);
            if (metadata == null) {
                goToSleep = true;
            } else if (metadata.leader() == null) {
                goToSleep = true;
            } else if (a_oldLeader.equalsIgnoreCase(metadata.leader().host()) && i == 0) {
                // first time through if the leader hasn't changed give
                // ZooKeeper a second to recover
                // second time, assume the broker did recover before failover,
                // or it was a non-Broker issue
                //
                goToSleep = true;
            } else {
                return metadata.leader().host();
            }
            if (goToSleep) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ie) {
                }
            }
        }
        System.out.println("Unable to find new leader after Broker failure. Exiting");
        throw new Exception("Unable to find new leader after Broker failure. Exiting");
    }
 
    private PartitionMetadata findLeader(List<String> a_seedBrokers, int a_port, String a_topic, int a_partition) {
        PartitionMetadata returnMetaData = null;
        loop: for (String seed : a_seedBrokers) {
            SimpleConsumer consumer = null;
            try {
                consumer = new SimpleConsumer(seed, a_port, 100000, 64 * 1024, "leaderLookup");
                List<String> topics = Collections.singletonList(a_topic);
                TopicMetadataRequest req = new TopicMetadataRequest(topics);
                kafka.javaapi.TopicMetadataResponse resp = consumer.send(req);
 
                List<TopicMetadata> metaData = resp.topicsMetadata();
                for (TopicMetadata item : metaData) {
                    for (PartitionMetadata part : item.partitionsMetadata()) {
                        if (part.partitionId() == a_partition) {
                            returnMetaData = part;
                            break loop;
                        }
                    }
                }
            } catch (Exception e) {
                System.out.println("Error communicating with Broker [" + seed + "] to find Leader for [" + a_topic + ", " + a_partition + "] Reason: " + e);
            } finally {
                if (consumer != null)
                    consumer.close();
            }
        }
        if (returnMetaData != null) {
            m_replicaBrokers.clear();
            for (BrokerEndPoint replica : returnMetaData.replicas()) {
                m_replicaBrokers.add(replica.host());
            }
        }
        return returnMetaData;
    }
}
```

#### 2.5 Kafka原理

##### 2.5.1 Leader和Follower

​		在Kafka中，每个topic的分区可以拥有若干副本。Kafka会自动在每个副本上备份数据，当一个节点down掉时，数据依旧是可用的。在创建topic的时候，Kafka会将每个分区的Leader均匀地分配在每个broker上。所有的读写操作都是由Leader来完成，而所有的follower都复制leader的日志数据文件。当leader出现故障时，follower会被选举为leader。

​		Kafka判断节点是否存活：

​		1）节点必须可以维护和Zookeeper的连接，Zookeeper通过心跳机制检查每个节点的连接；

​		2） 如果节点是follower，它必须能及时同步leader的写操作，延时不能太久。

#####	2.5.2 AR、ISR、OSR

> ​		每个分区会有多个副本，副本又分为Leader副本和Follower副本，Leader副本负责读写，Follower副本只负责同步，即只负责备份。当Leader副本down掉后，会从其他Follower副本中选举一个作为Leader副本。具体可参考：https://www.cnblogs.com/aspirant/p/9179045.html

​		在实际环境中，leader有可能会出现故障，所以Kafka会选举出一个新的leader。接下来说明涉及到的几个概念。Kafka中，可以把follower按照不同状态分为三类---AR、ISR、OSR。

* 分区所有副本称为[AR] （Assigned Replicas----已分配的副本）
* 所有与leader副本保持一定程度同步的副本（包括leader副本在内）组成[ISR] (In-Sync Replicas---在同步中的副本)
* 由于follower副本同步滞后过多的副本（不包括leader副本）组成[OSR] (Out-of-Sync Replias)
* AR = ISR + OSR
* 正常情况下，所有的follower副本都应该与leader副本保持同步，即AR = ISR, OSR集合为空

##### 2.5.3 Leader选举，Controller介绍

> ​		Kafka控制器管理着整个集群中分区以及副本的状态，控制器的选举需要依赖于Zookeeper，在Kafka集群启动的时候，都会先去访问ZK中的这个节点，如果不存在Broker就会创建这个节点，先到先得称为Controller，其他Broker当访问这个节点时，如果读取到的brokerid不等于-1，那么说明已经被选举出来了。那么获取的controller会如下：

```sh
[zk: localhost:2181(CONNECTED) 3] get /controller 
{"version":1,"brokerid":0,"timestamp":"1608173427409"}

#其中version为固定的版本号1
#brokerid为控制器的broker节点id
#timestamp为改节点被选举成Controller的时间
```

​		在任何时候，集群中都只会存在一个Controller，ZK中还维护着一个永久节点 --controller_epoch，该节点记录着控制器的变更次数，这个节点初始值为1，每选举一次Controller，那么该值+1。

​		Controller的职责：

* 监听分区的变化
* 监听主题相关的变化
* 监听Broker的变化
* 从ZK中获取Broker、Topic、Partition相关的元数据
* 启动并管理分区状态和副本状态
* 更新集群的元数据信息，并同步给其他的Broker
* 如果开启了自动优先副本选举，那么会后台启动一个任务用来自动维护优先副本的均衡

> 可参考：https://blog.csdn.net/shufangreal/article/details/111375955

##### 2.5.4 Kafka-Leader负载均衡

> ​		当一个broker停止或者crashes时，所有本来将它作为leader的分区将会把leader转移到其他broker上去，极端情况下，会导致同一个leader管理多个分区，导致负载不均衡，同时当这个broker重启时，如果这个broker不再是任何分区的leader,kafka的client也不会从这个broker来读取消息，从而导致资源的浪费。
>
> ​		kafka中有一个被称为优先副本（preferred replicas）的概念。如果一个分区有3个副本，且这3个副本的优先级别分别为0,1,2，根据优先副本的概念，0会作为leader 。当0节点的broker挂掉时，会启动1这个节点broker当做leader。当0节点的broker再次启动后，会自动恢复为此partition的leader。不会导致负载不均衡和资源浪费，这就是leader的均衡机制。

在配置文件conf/ server.properties中配置开启（默认就是开启）

```java
auto.leader.rebalance.enable true
```



##### 2.5.5 Kafka生产、消费数据流程

###### 2.5.5.1 Kafka生产数据流程

* broker进程上的leader将消息写入到本地log中
* follower从leader上拉取消息，写入到本地log，并向leader发送ACK
* leader接收到所有的ISR中的Replica的ACK后，并向生产者返回ACK

###### 2.5.5.2 Kafka数据消费流程

![image-20210520110517972](D:\文档\笔记\kafka\kafka入门笔记.assets\image-20210520110517972.png)

##### 2.5.6 Kafka数据存储形式（物理存储）

> 可参考： https://www.cnblogs.com/zhengzhaoxiang/p/13977382.html
>
> 问题：普通消息和包装消息

* 一个Topic由多个分区组成
* 一个分区(partition)由多个segment组成
* 一个segment（段）由多个文件组成（log、index、timeindex）



+ 文件管理：

​		保留数据时Kafka的一个基本特性，Kafka不会一直保留数据，也不会等所有消费者读取了消息才删除消息（消息消费情况是由消费者记录）。默认情况下，以segment为单位进行管理，默认大小为10M，每个片段(log)包含1GB或一周数据，以较小的为准。当前正在写入数据的片段叫做活跃片段，活跃片段不会被删除。

+ 文件格式：

  ​		Kafka的消息和偏移量保存在文件里。保存在磁盘上的数据格式与从生产者发送过来的或发送给消费者的消息格式是一样的。这样方便进行磁盘存储和网络传输，也方便Kafka使用零拷贝技术给消费者发送消息。问题：普通消息和包装消息？

![tmp](D:\文档\笔记\kafka\kafka入门笔记.assets\tmp.png)

>
>
>

##### 2.5.7 Kafka消息不丢失机制

> https://zhuanlan.zhihu.com/p/354772550

![tmp](D:\文档\笔记\kafka\kafka入门笔记.assets\tmp.jpg)

* Broker

  ​		Broker丢失消息的原因：**Kafka将数据异步批量存储到磁盘中，即按照一定的消息量和时间间隔进行刷盘。**

  > ​		这是由Liunx操作系统决定的。当将数据存储到Liunx操作系统中，会先存储到页缓存(Page cache)中，按照时间或者其他条件进行刷盘（从page cache到file)，或者通过fsync命令强制刷盘。数据在page cache中时，如果系统挂掉，数据会丢失。
  >
  > ​		刷盘触发条件有三：
  >
  > * 主动调用sync或fsync函数
  > * 可用内存低于阈值
  > * dirty data时间达到阈值。dirty是pagecache的一个标识位，当有数据写入到pageCache时，pagecache被标注为dirty，数据刷盘后，dirty标志清除。

  **kafka解决办法 ：通过producer与broker协同处理，也就是ack机制。**

  ​		具体如下：当ack为-1时，当所有的follower从leader中复制数据后，才会返回ack。此时数据已经写入磁盘，所以机器停电后，数据也不会丢失。

* Producer

  ​		Producer丢失数据，发生在生产者的客户端。为了提升效率，减少I/O，producer在发送数据时，可以将多个请求进行合并后发送。同时，我们还会将生产者改造为异步方式，客户端通过callback来处理消息发送失败或者超时情况。那么则会产生以下两种异常情况。

  + 那么数据在缓存在内存的时候，就可能会丢失，如下图所示。

  ![生产者数据丢失](D:\文档\笔记\kafka\kafka入门笔记.assets\生产者数据丢失.jpg)

  + 客户端发送完消息后，通过调用如果消息是多线程异步产生的，则线程会被挂起等待，占用内存，造成程序奔溃，消息丢失。

    ![生产者异步发送消息导致数据丢失](D:\文档\笔记\kafka\kafka入门笔记.assets\生产者异步发送消息导致数据丢失.jpg)

  + 解决思路：

    > - 同步：发送一批数据给Kafka后，等待Kafka返回结果
    > - 异步：发送一批数据给Kafka后，只是提供一个回调函数

* Consumer

  Consumer消费消息的步骤：接收消息、处理消息、反馈“处理完毕”(commited)

  Consumer消费方式主要分为两种：自动提交offset，手动提交offset

  + 自动提交offset：根据一定的时间间隔，将接收到的消息进行commit。commit过程和消费消息的过程是异步的。也就是说，可能存在消费过程未成功（比如抛出异常），commit消息已经提交了。此时消息就丢失了。示例代码如下，当insertIntoDB(record)失败后，消息将会出现丢失。

    ```java
    roperties props = new Properties();  
    props.put("bootstrap.servers", "localhost:9092");  
    props.put("group.id", "test");  
    // 自动提交开关
    props.put("enable.auto.commit", "true");
    // 自动提交时间间隔
    props.put("auto.commit.interval.ms", "1000");
    props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");  
    props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");  
    
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);  
    consumer.subscribe(Arrays.asList("foo", "bar"));  
    while (true) {
        // 调用poll后，1000ms后，消息状态会被改为 commited
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record: records) {
            insertIntoDB(record); // 将消息入库，时间可能会超过1000ms
        }
    }
    ```
    
  + 手动提交offset：将提交类型改为手动以后，可以保证消息“至少被消费一次”(at least once)。但是此时可能出现重复消费的情况。

    ```java
    Properties props = new Properties();  
    props.put("bootstrap.servers", "localhost:9092");  
    props.put("group.id", "test");  
    
    // 关闭自动提交，改为手动提交
    // 关闭自动提交，改为手动提交  
    props.put("enable.auto.commit", "false");  
    props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");  
    props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");  
    
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);  
    consumer.subscribe(Arrays.asList("foo", "bar"));  
    
    final int minBatchSize = 200;  
    List<ConsumerRecord<String, String>> buffer = new ArrayList<>();  
    
    while (true) {
        // 调用poll后，不会进行auto commit
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record: records) {
            buffer.add(record);
        }
        if (buffer.size() >= minBatchSize) {
            // 开启Spring事务
            insertIntoDb(buffer); 
            // 所有消息消费完毕后，才进行commit操作
            // 会出现重复消费的情况，当insertIntoDb()进行了一半，也就是消息已经消费了，但是未进行提交时，程序中断了，那么再次启动时，会出现重复消费的情况
            consumer.commitSync();
            // 事务提交
            buffer.clear();
        }
    }
    ```

    **如何处理消息重复消费（Flink提供Exactly-Once保障---二阶段提交（分布式事务））**

    ​		可以使用低级API，自己管理offset，和数据一起存在MySQL中，使用事务提交数据和offset，当失败的时候，一起失败，就可以保证消息不被重复消费。

    

  ##### 2.5.8 Kafka数据积压

  * 数据积压指的是一些外部I/O、一些比较耗时的操作（Full GC)，就会造成partition中一直存在未被消费，就会产生数据积压
  * 若有监控系统，则尽快处理 

  数据积压常见的场景：

  > https://zhuanlan.zhihu.com/p/312089762

  ##### 2.5.9 Kafka日志清理

  
