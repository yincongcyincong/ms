# raft
![image](https://github.com/yincongcyincong/ms/blob/main/image/raft.png)

### 当最新的follower选主失败时，比如说
（term，logid） （2，4）（2，4）（2，4）（2，5）,当其中一个（2，4）选举成功，（2，5）这条数据就会被删除

### 当有个follower数据落后太多时（2，9）（2，3）
这时（2，3）会发送我需要（2，4），然后leader会重传（2，3）之后的数据

#### redis和raft分布式锁有什么区别
redis需要设置key过期来防止思索，zk是利用其session加临时序列节点天然防止线程挂掉引起死锁
