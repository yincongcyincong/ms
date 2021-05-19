# raft
![image](https://github.com/yincongcyincong/ms/blob/main/image/raft.png)

### 当最新的follower选主失败时，比如说
（term，logid） （2，4）（2，4）（2，4）（2，5）,当其中一个（2，4）选举成功，（2，5）这条数据就会被删除

### 当有个follower数据落后太多时（2，9）（2，3）
这时（2，3）会发送我需要（2，4），然后leader会重传（2，3）之后的数据

#### redis和raft分布式锁有什么区别
redis需要设置key过期来防止思索，zk是利用其session加临时序列节点天然防止线程挂掉引起死锁

#### leader :
  
提交上一任期(term) leader的msg成为leader后发送一个空entries的msg到follower，让所有follower提交上一个term的msg；   
保证读的一致性 读请求默认发送到leader节点，收到请求之后确保自己仍然是当前系统的leader，判断的方法是根据heartbeat判断自己仍然能获取到大多数的follower的响应。   
heartbeat time 发送心跳，附带appendmsg 收到resp检查是否符合大多数，提交log entry   
election time 这个时间还在使用，每隔这个时间就发送消息pb.MsgCheckQuorum 检查是否能连上所有的follower    
#### preCandidate

follower切换成为candidate之后会增加系统的term，但如果该节点无法联系上系统中的大多数节点那么这次状态切换会导致term毫无意义的增大。因此在转换为candidate之前，需要发送prevote消息，消息中带有index和term，用于跟follower比较，确保entry够新；发送这个消息并且获取足够的响应才能成为candidate。    

#### candidate

收到MsgApp,heartbeat,snap消息则退回follower状态；若收到MsgPreVoteResp,则检查投票情况，超过半数成为leader   

#### follower

follower可以proxy的模式下工作，将收到的client请求route到leader。
收到prevote消息，判断是否在leader lease内；如果自己还能收到leader的消息，拒绝投票。
收到vote 消息则重置自己的election timer避免自己超时成为candidate 避免无谓的竞争。
处理heartbeat更新commitIndex并重置election timer，超时则切换为precandidate
处理msgApp消息，追加日志。
