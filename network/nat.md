#### Full-cone NAT（also known as one-to-one NAT）
一旦一个内网地址 (iAddr:iPort) 被映射到一个外部地址 (eAddr:ePort), 来自 iAddr:iPort 的任何数据包将通过 eAddr:ePort 发送。

任何外部主机能够通过eAddr:ePort这个地址发送数据包到iAddr:iPort.

#### Address-restricted-cone NAT
一旦一个内网地址 (iAddr:iPort) 被映射到一个外部地址 (eAddr:ePort), 来自 iAddr:iPort 的任何数据包将通过 eAddr:ePort 发送.

仅只有接收到主机(iAddr:iPort)通过eAddr:ePort发送的数据包的外部主机通过该主机的任何端口发送到eAddr:ePort的数据包才能够被正确的转发到iAddr:iPort.也就是说主机有关端口无关.

#### Port-restricted cone NAT
类似于address restricted cone NAT, 但是端口号有限制.

一旦一个内网地址 (iAddr:iPort) 被映射到一个外部地址 (eAddr:ePort), 来自 iAddr:iPort 的任何数据包将通过 eAddr:ePort 发送.

仅只有接收到主机(iAddr:iPort)通过eAddr:ePort发送的数据包的外部主机通过该主机的相同端口发送到eAddr:ePort的数据包才能够被正确的转发到iAddr:iPort.

#### Symmetric NAT
来自相同内部ip和port发送到相同目的地ip和port的请求被映射到唯一的外部ip和port地址；如果相同的内部主机采用相同的ip和port地址发送到不同的目的地，那么重新分配映射地址。

只有先前收到内部主机发送的包的外部主机才能够发送返回包到内部主机。

针对前面三种NAT类型（即cone NAT）只要通信双方彼此知道对方的内部地址和外部地址的映射关系，然后通过UDP打洞的方式就可以建立相互连接的通信；但是第四种也就是Symmetric NAT的话由于每次向不同目的地发送数据包时采用不同的外部地址，也就没办法通过直接的方式建立P2P连接。

|   | 全锥型 | ip限制锥型 | 端口限制锥型  |  对称型 |   
|  ----  | ----  | ----  | ----  | ----  |
| 全锥型 | 可以 |  可以 |  可以 | 可以 |
| ip限制锥型 | 可以 |  可以 |  可以 | 可以 |
| 端口限制锥型  | 可以 |  可以 |  可以 | 不可以 |
| 对称型  | 可以 |  可以 |  不可以 | 不可以 |
