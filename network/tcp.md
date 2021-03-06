# tcp协议：

#### tcp包头：
  ![image](https://github.com/yincongcyincong/ms/blob/main/image/tcp_header.png)
URG：表示紧急指针是否有效；   
ACK：表示确认号是否有效，携带ACK标志的数据报文段为确认报文段；    
PSH：提示接收端的应用程序应该立即从TCP接受缓冲区中读走数据，为接受后续数据腾出空间；   
RST：表示要求对方重新建立连接，携带RST标志位的TCP报文段成为复位报文段；    
SYN：表示请求建立一个连接，携带SYN标志的TCP报文段为同步报文段；    
FIN：通知对方本端要关闭了，带FIN标志的TCP报文段为结束报文段。

#### 拥塞避免：
为了防止cwnd增加过快而导致网络拥塞，所以需要设置一个慢开始门限ssthresh状态变量（我也不知道这个到底是什么，就认为他是一个拥塞控制的标识）,它的用法：
![image](https://github.com/yincongcyincong/ms/blob/main/image/load_avoid.png)
 1. 当cwnd < ssthresh,使用慢启动算法。   
 2. 当cwnd > ssthresh,使用拥塞控制算法，停用慢启动算法。   
 3. 当cwnd = ssthresh，这两个算法都可以。   

#### 快重传：
  ![image](https://github.com/yincongcyincong/ms/blob/main/image/quick_send.png)
  快重传算法要求首先接收方收到一个失序的报文段后就立刻发出重复确认，而不要等待自己发送数据时才进行捎带确认。接收方成功的接受了发送方发送来的M1、M2并且分别给发送了ACK，现在接收方没有收到M3，而接收到了M4，显然接收方不能确认M4，因为M4是失序的报文段。如果根据可靠性传输原理接收方什么都不做，但是按照快速重传算法，在收到M4、M5等报文段的时候，不断重复的向发送方发送M2的ACK,如果接收方一连收到三个重复的ACK,那么发送方不必等待重传计时器到期，由于发送方尽早重传未被确认的报文段。


#### 快恢复：
![image](https://github.com/yincongcyincong/ms/blob/main/image/quick_recover.png)
  1. 当发送发连续接收到三个确认时，就执行乘法减小算法，把慢启动开始门限（ssthresh）减半，但是接下来并不执行慢开始算法。
  2. 此时不执行慢启动算法，而是把cwnd设置为ssthresh的一半， 然后执行拥塞避免算法，使拥塞窗口缓慢增大。

![image](https://github.com/yincongcyincong/ms/blob/main/image/three_hello.png)
![image](https://github.com/yincongcyincong/ms/blob/main/image/four_goodbye.png)
#### 【问题1】为什么连接的时候是三次握手，关闭的时候却是四次握手？
答：因为当Server端收到Client端的SYN连接请求报文后，可以直接发送SYN+ACK报文。其中ACK报文是用来应答的，SYN报文是用来同步的。但是关闭连接时，当Server端收到FIN报文时，很可能并不会立即关闭SOCKET，所以只能先回复一个ACK报文，告诉Client端，"你发的FIN报文我收到了"。只有等到我Server端所有的报文都发送完了，我才能发送FIN报文，因此不能一起发送。故需要四步握手。

#### 【问题2】为什么TIME_WAIT状态需要经过2MSL(最大报文段生存时间)才能返回到CLOSE状态？
答：虽然按道理，四个报文都发送完毕，我们可以直接进入CLOSE状态了，但是我们必须假象网络是不可靠的，有可以最后一个ACK丢失。所以TIME_WAIT状态就是用来重发可能丢失的ACK报文。在Client发送出最后的ACK回复，但该ACK可能丢失。Server如果没有收到ACK，将不断重复发送FIN片段。所以Client不能立即关闭，它必须确认Server接收到了该ACK。Client会在发送出ACK之后进入到TIME_WAIT状态。Client会设置一个计时器，等待2MSL的时间。如果在该时间内再次收到FIN，那么Client会重发ACK并再次等待2MSL。所谓的2MSL是两倍的MSL(Maximum Segment Lifetime)。MSL指一个片段在网络中最大的存活时间，2MSL就是一个发送和一个回复所需的最大时间。如果直到2MSL，Client都没有再次收到FIN，那么Client推断ACK已经被成功接收，则结束TCP连接。

#### tcp粘包
TCP是面向流的, 流, 要说明就像河水一样, 只要有水, 就会一直流向低处, 不会间断. TCP为了提高传输效率, 发送数据的时候, 并不是直接发送数据到网路, 而是先暂存到系统缓冲, 超过时间或者缓冲满了, 才把缓冲区的内容发送出去, 这样, 就可以有效提高发送效率. 所以会造成所谓的粘包, 即前一份Send的数据跟后一份Send的数据可能会暂存到缓冲当中, 然后一起发送.

#### close wait状态
被断开连接放，主动发fin包没有发，因为忙于读和写

#### tcp异常断开
客户端程序崩溃或异常退出：服务端read时会报connection rest by peer。   
断电重启：服务端发送心跳信息时，会收到客户端的RST消息，调用read时会报connection rest by peer。    
断电或网络中断：服务端发送心跳信息后超时。   
