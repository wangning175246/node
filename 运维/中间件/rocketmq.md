> https://github.com/apache/rocketmq/blob/master/docs/cn/architecture.md

#### NameServer

- NameServer 中维护着 Producer 集群、Broker     集群、 Consumer 集群的服务状态。通过定时发送心跳数据包进行维护更新各个服务的状态。
- 当有新的Producer     加入集群时，通过上报自身的服务信息，及获取各个 Broker Master的信息（Broker 地址、Topic、Queue     等信息），这样就可以决定把对应的Topic消息存储到那个Broker、哪个Queue 上。Consumer 同理。
- NameServer     可以部署多个，多个NameServer互相独立，不会交换消息。Producer、Broker、Consumer 启动的时候都需要指定多个     NameServer，各个服务的信息会同时注册到多个 NameServer 上，从而能到达高可用。

NameServer直接启动即可。

#### Brocker

Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave的对应关系通过指定相同的BrokerName，不同的BrokerId来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与Name Server集群中的所有节点建立长连接，定时注册Topic信息到所有Name Server。

![](images\rocketmq\image-20200315122201255-1608818829064.png)

#### Topic

消息存储架构图中主要有下面三个跟消息存储相关的文件构成。

(1) CommitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；

(2) ConsumeQueue：消息消费队列，引入的目的主要是提高消息消费的性能，由于RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。Consumer即可根据ConsumeQueue来查找待消费的消息。其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。consumequeue文件可以看成是基于topic的commitlog索引文件，故consumequeue文件夹的组织方式如下：topic/queue/file三层组织结构，具体存储路径为：$HOME/store/consumequeue/{topic}/{queueId}/{fileName}。同样consumequeue文件采取定长设计，每一个条目共20个字节，分别为8字节的commitlog物理偏移量、4字节的消息长度、8字节tag hashcode，单个文件由30W个条目组成，可以像数组一样随机访问每一个条目，每个ConsumeQueue文件大小约5.72M；

(3) IndexFile：IndexFile（索引文件）提供了一种可以通过key或时间区间来查询消息的方法。Index文件的存储位置是：$HOME \store\index${fileName}，文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引，IndexFile的底层存储设计为在文件系统中实现HashMap结构，故rocketmq的索引文件其底层实现为hash索引。

在上面的RocketMQ的消息存储整体架构图中可以看出，RocketMQ采用的是混合型的存储结构，即为Broker单个实例下所有的队列共用一个日志数据文件（即为CommitLog）来存储。RocketMQ的混合型存储结构(多个Topic的消息实体内容都存储于一个CommitLog中)针对Producer和Consumer分别采用了数据和索引部分相分离的存储结构，Producer发送消息至Broker端，然后Broker端使用同步或者异步的方式对消息刷盘持久化，保存至CommitLog中。只要消息被刷盘持久化至磁盘文件CommitLog中，那么Producer发送的消息就不会丢失。正因为如此，Consumer也就肯定有机会去消费这条消息。当无法拉取到消息后，可以等下一次消息拉取，同时服务端也支持长轮询模式，如果一个消息拉取请求未拉取到消息，Broker允许等待30s的时间，只要这段时间内有新消息到达，将直接返回给消费端。这里，RocketMQ的具体做法是，使用Broker端的后台服务线程—ReputMessageService不停地分发请求并异步构建ConsumeQueue（逻辑消费队列）和IndexFile（索引文件）数据。

Tag

> https://www.codercto.com/a/90483.html

#### 基础命令

mqadmin topicList -c -n 127.0.0.1:9876 查看的是所有的topic

mqadmin clusterList -c -n 127.0.0.1:9876 查看集群的信息，broker的信息

mqadmin statsAll -n 127.0.0.1:9876

mqadmin consumerProgress -g ConsumerES -n 127.0.0.1:9876 查看消费这的信息。

#### 基础性能指标

TPS 服务器每秒处理的事务数，

写入TPS 每秒可以发送到消息队列中的的最大消息 

消费TPS 每秒可以从消息队列中消费的消息数

<img src="images\rocketmq\clip_image001.png" alt="enter image description here" style="zoom: 100%;" />