下载压缩包，解压缩
> tar -xzf kafka_2.11-0.9.0.0.tgz<br>
> cd kafka_2.11-0.9.0.0<br>
对kafka/config中的属性文件进行配置，详细配置可见http://www.inter12.org/archives/842


<b>配置server.properties</b><br>
> broker.id = 1<br>
每一个broker在集群中的唯一标示，要求是正数。在改变IP地址，不改变broker.id的话不会影响consumers<br>
log.dirs = /tmp/kafka-logs <br>
kafka数据的存放地址，多个地址的话用逗号分割 /tmp/kafka-logs-1，/tmp/kafka-logs-2<br>
zookeeper.connect = localhost:2181<br>
zookeeper集群的地址，可以是多个，多个之间用逗号分割 hostname1:port1,hostname2:port2,hostname3:port3<br>
host.name=localhost  改为  本机的ip地址：host.name=192.168.127.150<br>
否则组建集群时会报错Should not set log end offset on partition，当个参数在默认不配置的时候，绑定的是当前主机127.0.0.1，所以集群中主机之间进行相互备份的时候通过127.0.0.1找不到主机了<br>
Advertised.host.name = （服务器ip）：advertised.host.name=192.168.127.150<br>
需要设置服务器ip，否则可能报错,该字段的值是生产者和消费者使用的。如果没有设置，则会取host.name的值，默认情况下，该值为localhost，如果生产者拿到localhost这个值，只往本地发消息，必然会报错（因为本地没有kafka服务器）<br>

<b>配置producer.properties</b><br>
> metadata.broker.list=localhost:9092,localhost:9093,localhost:9094<br>
消费者获取消息元信息(topics, partitions and replicas)的地址,配置格式是：host1:port1,host2:port2，也可以在外面设置一个vip<br>
Request.required.acks = 0<br>
消息的确认模式<br>
 &nbsp;0：不保证消息的到达确认，只管发送，低延迟但是会出现消息的丢失，在某个server失败的情况下，有点像TCP<br>
 &nbsp;1：发送消息，并会等待leader 收到确认后，一定的可靠性<br>
 &nbsp; -1：发送消息，等待leader收到确认，并进行复制操作后，才返回，最高的可靠性<br>
producer.type = sync<br>
生产者的类型 async:异步执行消息的发送 sync：同步执行消息的发送<br><br>
serializer.class = kafka.serializer.DefaultEncoder<br>
消息体的系列化处理类 ，转化为字节流进行传输<br>

<b>配置consumer.properties</b><br>
> group.id = test-consumer-group<br>
Consumer归属的组ID，broker是根据group.id来判断是队列模式还是发布订阅模式<br>
zookeeper.connect = localhost:2182<br>
对于zookeeper集群的指定，可以是多个 hostname1:port1,hostname2:port2,hostname3:port3 必须和broker(server.properties)使用同样的zk配置<br>

启动zookeeper
> bin/zookeeper-server-start.sh config/zookeeper.properties

启动kafka
> bin/kafka-server-start.sh config/server.properties

创建一个主题（使用单个分区和一个副本）
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

启动consumer
> bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning

启动provider
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test

删除主题
> bin/kafka-topics.sh --delete --topic test --zookeeper localhost:2181 
用kafka-topics.sh的delete命令删除topic，会有两种情况：
如果当前topic没有使用过即没有传输过信息：可以彻底删除
如果当前topic有使用过即有过传输过信息：并没有真正删除topic只是把这个topic标记为删除（marked for deletion）。


要彻底把情况2中的topic删除必须把kafka中与当前topic相关的数据目录和zookeeper与当前topic相关的路径一并删除。
（1）删除kafka相关的数据目录
     目录位置为server.properties中设置的log.dirs位置
（2）删除zookeeper中关于当前topic的节点，打开zookeeper client，用ls查看目标节点路径，再用rmr命令删除
     rmr /brokers/topics/my_topic(rm可删除非空节点）
显示创建的主题
bin/kafka-topics.sh --list --zookeeper localhost:2181
在Kafka集群上创建备份因子为3，分区数为4的Topic：
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 4 --topic kafka


1.Kafka会保留消费过的数据，但是因为磁盘限制，不可能永久保留所有数据（实际 上也没必要），因此Kafka提供两种策略去删除旧数据。一是基于时间，二是基于partition文件大小。例如可以通过配置$KAFKA_HOME/config/server.properties，让Kafka删除一周前的数据（log.retention.hours=168），也可通过配置让Kafka在partition文件超过1GB时删除旧数据（log.retention.bytes=1073741824）
2.Kafka保证保证同一个consumer group里只有一个consumer会消费一条消息。实际上，Kafka保证的是稳定状态下每一个consumer实例只会消费某一个或多个特定 partition的数据，而某个partition的数据只会被某一个特定的consumer实例所消费。Kafka还允许不同consumer group同时消费同一条消息
