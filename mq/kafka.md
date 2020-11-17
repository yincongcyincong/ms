# kafka

Producer:生产者    
Broker：实体机器，多个broker选出一个controller进行管理    
Consumer：生产者，拉数据    
Topic：主题,一个topic有多个partition，每个partition有多个副本   
Partition:会被选主，只有主会接受外部请求，其他的follower只同步数据    

所有的isr是否同步leader的数据然后成功是需要配置的，默认是不同步就成功   

多个consumer会收到同一条消息，在一个consumer group里面的consumer不会收到同一条    

partition挂了，只有isr里面的partition能成为leader    

元数据（topic信息；分区信息：topic有哪些分区，哪些副本，分别在哪台brocker上，哪个是leader；consumer信息等）储存在zk上   

kafka0拷贝，顺序读写   

没有consumer的ack，offset需要自己储存   

保证数据完整性用binlog，进内存，然后内存刷到磁盘上，顺序写文件，所以比较快    

or = isr+osr    
hw：只有写入的数据被 同步到 所有的ISR中的 副本后，数据才认为已提交，HW更新到该位置，HW之前的数据才可以被消费者访问，保证 没有 同步完成的数据不会被消费者 访问到   

__consumer_offsets：kafka内置topic，记录消费者组的元数据和offset   

#### 如何保证消息不丢
1 副本机制 2 producer确认机制 3 leader选取机制 4 消费者手动提交

#### 消费者消费机制
at most once  先提交再消费  最多消费一次 存在数据丢失可能   

at least once 先消费再提交  最少消费一次 存在重复消费可能（kafka默认）    

exaclty once 通过事务等机制保证一定消费且只一次    

#### 生产者同步机制
acks=0 ： 消息发送出去就认为已经成功了，不会等待任何来自服务器的响应；    
acks=1 ： 只要集群的首领节点收到消息，生产者就会收到一个来自服务器成功响应；    
acks=all ：只有当所有参与复制的节点全部收到消息时，生产者才会收到一个来自服务器的成功响应。    

#### isr和osr
ISR中的副本都要同步leader中的数据，只有都同步完成了数据才认为是成功提交了，成功提交之后才能供外界访问。    
在这个同步的过程中，数据即使已经写入也不能被外界访问，这个过程是通过LEO-HW机制来实现的。   

OSR内的副本是否同步了leader的数据，不影响数据的提交，OSR内的follower尽力的去同步leader，可能数据版本会落后。   
最开始所有的副本都在ISR中，在kafka工作的过程中，如果某个副本同步速度慢于replica.lag.time.max.ms指定的阈值，则被踢出ISR存入OSR，如果后续速度恢复可以回到ISR中。   

LogEndOffset：分区的最新的数据的offset，当数据写入leader后，LEO就立即执行该最新数据。相当于最新数据标识位。   

HighWatermark：只有写入的数据被同步到所有的ISR中的副本后，数据才认为已提交，HW更新到该位置，HW之前的数据才可以被消费者访问，保证没有同步完成的数据不会被消费者访问到。相当于所有副本同步数据标识位。    
在leader宕机后，只能从ISR列表中选取新的leader，无论ISR中哪个副本被选为新的leader，它都知道HW之前的数据，可以保证在切换了leader后，消费者可以继续看到HW之前已经提交的数据。    
所以LEO代表已经写入的最新数据位置，而HW表示已经同步完成的数据，只有HW之前的数据才能被外界访问。   

如果leader宕机，选出了新的leader，而新的leader并不能保证已经完全同步了之前leader的所有数据，只能保证HW之前的数据是同步过的，此时所有的follower都要将数据截断到HW的位置，再和新的leader同步数据，来保证数据一致。
当宕机的leader恢复，发现新的leader中的数据和自己持有的数据不一致，此时宕机的leader会将自己的数据截断到宕机之前的hw位置，然后同步新leader的数据。宕机的leader活过来也像follower一样同步数据，来保证数据的一致性。    

#### leader选举
当leader宕机时会选择ISR中的一个follower成为新的leader，如果ISR中的所有副本都宕机，怎么办？    

有如下配置可以解决此问题：   

unclean.leader.election.enable=false    

策略1：必须等待ISR列表中的副本活过来才选择其成为leader继续工作。   

unclean.leader.election.enable=true   

策略2：选择任何一个活过来的副本，成为leader继续工作，此follower可能不在ISR中。    

策略1，可靠性有保证，但是可用性低，只有最后挂了leader活过来kafka才能恢复。   

策略2，可用性高，可靠性没有保证，任何一个副本活过来就可以继续工作，但是有可能存在数据不一致的情况。    
