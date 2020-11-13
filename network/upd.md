# udp
#### udp包头
![image](https://github.com/yincongcyincong/ms/blob/main/image/udp_header.jpeg)   
头部校验和字段：占16比特。内容是根据IP头部计算得到的校验和码。计算方法是：对头部中每个16比特进行二进制反码求和。（和ICMP、IGMP、TCP、UDP不同，IP不对头部后的数据进行校验）
#### udp发送
UDP面向报文形式, 系统是不会缓冲的, 也不会做优化的, Send的时候, 就会直接Send到网络上, 对方收不收到也不管, 所以这块数据总是能够能一包一包的形式接收到, 而不会出现前一个包跟后一个包都写到缓冲然后一起Send.
