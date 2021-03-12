# Rabbitmq

 ![image](https://github.com/yincongcyincong/ms/blob/main/image/rabbitmq.png)   


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
