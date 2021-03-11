
第一、建连耗时问题。对于长连接来讲，连接包括 TCP 握手和信令握手，需要 2RTT。而通过 QUIC 协议可以做到协议层 0RTT 握手，总共能节省一个 RTT。   
  ![image](https://github.com/yincongcyincong/ms/blob/main/image/quic-hand.png)
  ![image](https://github.com/yincongcyincong/ms/blob/main/image/tls-hand.png)

第二、连接保活。TCP 使用 WIFI 切换 4G，或者是 4G 切换 WIFI 会导致连接中断，影响连接保活率。而 QUIC 协议支持连接迁移，网络切换无需重连，所以当 IP 发生变化，仍然可以保持长连接，不需要中断。    

第三、头部阻塞。单个 TCP 连接，队头报文丢失，会影响队列中所有信令时延。在 QUIC 协议中基于 UDP 传输，创建多 Stream，所以队头报文丢失，不会影响其他信令，传输耗时也会有所改善。    

第四、弱网优化。经过数据统计分析，在丢包率比较高的链路上，由于端上的拥塞控制算法依赖丢包检测，导致耗时较高。基于 QUIC 协议，可以在应用层面定制优化一些算法，例如使用像 BBR 等拥塞控制算法，弱网条件下抗丢包能力比较强。   

