# Rabbitmq

Broker：简单来说就是消息队列服务器实体。   
Exchange：消息交换机，它指定消息按什么规则，路由到哪个队列。    
Queue：消息队列载体，每个消息都会被投入到一个或多个队列。   
Binding：绑定，它的作用就是把exchange和queue按照路由规则绑定起来。   
Routing Key：路由关键字，exchange根据这个关键字进行消息投递。    
vhost：虚拟主机，一个broker里可以开设多个vhost，用作不同用户的权限分离。    
producer：消息生产者，就是投递消息的程序。   
consumer：消息消费者，就是接受消息的程序。   
channel：消息通道，在客户端的每个连接里，可建立多个channel，每个channel代表一个会话任务。   
ExchangeType：fanout,direct,topic,headers


1.队列元数据：队列名称和它们的属性（是否可持久化，是否自动删除）
2.交换器元数据：交换器名称、类型、属性（是否持久化）
3.绑定元数据：一张表格，展示如何将消息路由到队列
4.vhost元数据：vhost内的队列、交换器和绑定提供命名空间和安全属性。

镜像队列：消息和元数据所有节点都有

和kafka一样先记日志，等内存缓冲区满了就落盘，所有节点数据都落磁盘

有ack，没有offset

consumer是被推送数据

镜像队列有主队列，发消息先路由到主队列，然后到从队列，consumer只读主队列

mnesia数据库选主
