#### redis的string怎么ttl的
其实有三种不同的删除策略：   
（1）：立即删除。在设置键的过期时间时，创建一个回调事件，当过期时间达到时，由时间处理器自动执行键的删除操作。   
（2）：惰性删除。键过期了就过期了，不管。每次从dict字典中按key取值时，先检查此key是否已经过期，如果过期了就删除它，并返回nil，如果没过期，就返回键值。    
（3）：定时删除。每隔一段时间，对expires字典进行检查，删除里面的过期键。    
可以看到，第二种为被动删除，第一种和第三种为主动删除，且第一种实时性更高。下面对这三种删除策略进行具体分析。    


#### redis的string怎么存储的

#### 处理hash冲突的算法
开放定值法：    

也叫再散列法，当关键字key的哈希地址p=H（key）出现冲突时，以p为基础，产生另一个哈希地址p1，如果p1仍然冲突，再以p为基础，产生另一个哈希地址p2，…，直到找出一个不冲突的哈希地址pi，将相应元素存入其中。

通常都是用以下公式计算：Hi=（H（key）+di）% m i=1，2，…，n

其中H（key）为哈希函数，m 为表长，di称为增量序列。增量序列的取值方式不同，相应的再散列方式也不同。主要有三种：线性探测再散列（冲突发生时，顺序查看表中下一单元，直到找出一个空单元或查遍全表），二次探测再散列（冲突发生时，在表的左右进行跳跃式探测，直到找到空单元），伪随机探测再散列。

链地址法：   

这种方法的基本思想是将所有哈希地址为i的元素构成一个称为同义词链的单链表，并将单链表的头指针存在哈希表的第i个单元中，因而查找、插入和删除主要在同义词链中进行。链地址法适用于经常进行插入和删除的情况。像之前看到的HashMap就是利用这种方法解决Hash冲突的。

再哈希法：   

多写几个哈希函数，算出来一个hashcode重复的就用另一个哈希函数算，直到算出来不一样。


#### redis集群模式主挂了怎选从
通过投票机制来判断master是否不可用，参与投票的是所有的master，所有的master之间维持着心跳，如果一半以上的master确定某个master失联，则集群认为该master挂掉，此时发生主从切换。    

通过选举机制来确定哪一个从节点升级为master。选举的依据依次是：网络连接正常->5秒内回复过INFO命令->10*down-after-milliseconds内与主连接过的->从服务器优先级->复制偏移量->运行id较小的。选出之后通过slaveif no ont将该从服务器升为新master。   