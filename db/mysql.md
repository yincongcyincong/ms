# mysql

#### 双1配置

innodb_flush_log_at_trx_commit    
如果innodb_flush_log_at_trx_commit设置为0：log buffer将每秒一次地写入log file中，并且log file的flush(刷到磁盘)操作同时进行.该模式下，在事务提交的时候，不会主动触发写入磁盘的操作;    
如果innodb_flush_log_at_trx_commit设置为1：每次事务提交时MySQL都会把log buffer的数据写入log file，并且flush(刷到磁盘)中去;    
如果innodb_flush_log_at_trx_commit设置为2：每次事务提交时MySQL都会把log buffer的数据写入log file，但是flush(刷到磁盘)操作并不会同时进行。该模式下,MySQL会每秒执行一次 flush(刷到磁盘)操作。   

注意：由于进程调度策略问题,这个"每秒执行一次 flush(刷到磁盘)操作"并不是保证100%的"每秒"。   

sync_binlog   
sync_binlog 的默认值是0，像操作系统刷其他文件的机制一样，MySQL不会同步到磁盘中去而是依赖操作系统来刷新binary log。   
当sync_binlog =N (N>0) ，MySQL 在每写 N次 二进制日志binary log时，会使用fdatasync()函数将它的写二进制日志binary log同步到磁盘中去。    

注意：如果启用了autocommit，那么每一个语句statement就会有一次写操作；否则每个事务对应一个写操作。    

二、性能    
两个参数在不同值时对db的纯写入的影响表现如下：    
测试场景1   
innodb_flush_log_at_trx_commit=2    
sync_binlog=1000    

测试场景2   
innodb_flush_log_at_trx_commit=1    
sync_binlog=1000    

测试场景3   
innodb_flush_log_at_trx_commit=1    
sync_binlog=1   

测试场景4   
innodb_flush_log_at_trx_commit=1    
sync_binlog=1000    

测试场景5   
innodb_flush_log_at_trx_commit=2    
sync_binlog=1000    

在以上5个场景下的TPS分别为:    
场景1            41000    
场景2           33000   
场景3           26000   
场景4           33000   

由此可见，当两个参数设置为双1的时候，写入性能最差，sync_binlog=N (N>1 ) innodb_flush_log_at_trx_commit=2 时，(在当前模式下)MySQL的写操作才能达到最高性能。    

#### B树的性质
1、定义任意非叶子结点最多只有M个儿子，且M>2；   
2、根结点的儿子数为[2, M]；     
3、除根结点以外的非叶子结点的儿子数为[M/2, M]；    
4、每个结点存放至少M/2-1（取上整）和至多M-1个关键字；（至少2个关键字）    
5、非叶子结点的关键字个数=指向儿子的指针个数-1；     
6、非叶子结点的关键字：K[1], K[2], …, K[M-1]；且K[i] < K[i+1]；     
7、非叶子结点的指针：P[1], P[2], …, P[M]；其中P[1]指向关键字小于K[1]的子树，P[M]指向关键字大于K[M-1]的子树，其它P[i]指向关键字属于(K[i-1], K[i])的子树；    
8、所有叶子结点位于同一层；    

#### b+树优点
所有叶子结点的路径相同 io次数相同    
每次加载一页    
范围查询优秀    
相比b-叶子结点能存更多索引    

#### 聚簇索引
所谓聚簇索引，就是指主索引文件和数据文件为同一份文件，聚簇索引主要用在Innodb存储引擎中。在该索引实现方式中B+Tree的叶子节点上的data就是数据本身，key为主键，如果是一般索引的话，data便会指向对应的主索引，在B+Tree的每个叶子节点增加一个指向相邻叶子节点的指针，就形成了带有顺序访问指针的B+Tree。做这个优化的目的是为了提高区间访问的性能，例如图4中如果要查询key为从18到49的所有数据记录，当找到18后，只需顺着节点和指针顺序遍历就可以一次性访问到所有数据节点，极大提到了区间查询效率。

#### 非聚簇索引
非聚簇索引就是指B+Tree的叶子节点上的data，并不是数据本身，而是数据存放的地址。主索引和辅助索引没啥区别，只是主索引中的key一定得是唯一的。主要用在MyISAM存储引擎中

#### innodb和myisam区别
1. InnoDB支持事务，MyISAM不支持，对于InnoDB每一条SQL语言都默认封装成事务，自动提交，这样会影响速度，所以最好把多条SQL语言放在begin和commit之间，组成一个事务；    
2. InnoDB支持外键，而MyISAM不支持。对一个包含外键的InnoDB表转为MYISAM会失败；    
3. InnoDB是聚集索引，使用B+Tree作为索引结构，数据文件是和（主键）索引绑在一起的（表数据文件本身就是按B+Tree组织的一个索引结构），必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。    
4. InnoDB不保存表的具体行数，执行select count(*) from table时需要全表扫描。而MyISAM用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快；    
5. Innodb不支持全文索引，而MyISAM支持全文索引，查询效率上MyISAM要高；5.7以后的InnoDB支持全文索引了    
6. MyISAM表格可以被压缩后进行查询操作   
7. InnoDB支持表、行(默认)级锁，而MyISAM支持表级锁   

#### 索引类型
Mysql目前主要有以下几种索引类型：FULLTEXT，HASH，BTREE，RTREE。

1. FULLTEXT   
即为全文索引，目前只有MyISAM引擎支持。其可以在CREATE TABLE ，ALTER TABLE ，CREATE INDEX 使用，不过目前只有 CHAR、VARCHAR ，TEXT 列上可以创建全文索引。    
全文索引并不是和MyISAM一起诞生的，它的出现是为了解决WHERE name LIKE “%word%"这类针对文本的模糊查询效率较低的问题。    
2. HASH   
由于HASH的唯一（几乎100%的唯一）及类似键值对的形式，很适合作为索引。    
HASH索引可以一次定位，不需要像树形索引那样逐层查找,因此具有极高的效率。但是，这种高效是有条件的，即只在“=”和“in”条件下高效，对于范围查询、排序及组合索引仍然效率不高。   
3. BTREE    
BTREE索引就是一种将索引值按一定的算法，存入一个树形的数据结构中（二叉树），每次查询都是从树的入口root开始，依次遍历node，获取leaf。这是MySQL里默认和最常用的索引类型。    
4. RTREE    
RTREE在MySQL很少使用，仅支持geometry数据类型，支持该类型的存储引擎只有MyISAM、BDb、InnoDb、NDb、Archive几种   

普通索引：仅加速查询    
唯一索引：加速查询 + 列值唯一（可以有null）   
主键索引：加速查询 + 列值唯一（不可以有null）+ 表中只有一个    
组合索引：多列值组成一个索引，专门用于组合搜索，其效率大于索引合并   
全文索引：对文本的内容进行分词，进行搜索    

#### innodb锁类型：（for update）
record lock：记录锁，也就是仅仅锁着单独的一行    
gap lock：区间锁，仅仅锁住一个区间(注意这里的区间都是开区间，也就是不包括边界值。   
next-key lock：record lock+gap lock，所以next-key lock也就半开半闭区间，且是下界开，上界闭。左开右闭[8,9),如果有唯一索引，就是record lock    
共享锁【S锁】   
又称读锁，若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S锁。    
这保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。   
 
排他锁【X锁】   
又称写锁。若事务T对数据对象A加上X锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的锁。    
这保证了其他事务在T释放A上的锁之前不能再读取和修改A。    


#### innodb引擎的4大特性
插入缓冲（insert buffer）,二次写(double write),自适应哈希索引(ahi),预读(read ahead)

#### 索引失效的场景
1.	like 以%开头，索引无效；当like前缀没有%，后缀有%时，索引有效。   
2.	or语句前后没有同时使用索引。当or左右查询字段只有一个是索引，该索引失效，只有当or左右查询字段均为索引时，才会生效。    
3.	组合索引，不是使用第一列索引，索引失效。    
4.	数据类型出现隐式转化。如varchar不加单引号的话可能会自动转换为int型，使索引无效，产生全表扫描。    
5.	在索引列上使用 IS NULL 或 IS NOT NULL操作。索引是不索引空值的，所以这样的操作不能使用索引，可以用其他的办法处理，例如：数字类型，判断大于0，字符串类型设置一个默认值，判断是否等于默认值即可。    
6.	在索引字段上使用not，<>，!=。不等于操作符是永远不会用到索引的，因此对它的处理只会产生全表扫描。 优化方法： key<>0 改为 key>0 or key<0。   
7.	对索引字段进行计算操作。    
8.	在索引字段上使用函数。   
9.	当全表扫描速度比索引速度快时，mysql会使用全表扫描，此时索引失效。   

#### where和having都可以使用的场景
1）select addtime,name from dw_users where addtime> 1500000000   
2）select addtime,name from dw_users having addtime> 1500000000    
解释：上面的having可以用的前提是我已经筛选出了addtime字段，在这种情况下和where的效果是等效的，但是如果我没有select addtime就会报错！！因为having是从前面筛选的字段再筛选，而where是从数据表中的字段直接进行的筛选的。    
2. 只可以用where，不可以用having的情况    
1） select addtime,name from dw_users where addtime> 1500000000    
2） select phone,name from dw_users having addtime> 1500000000 //报错！！！因为前面并没有筛选出addtime字段    
3. 只可以用having，不可以用where情况   
查询每种category_id商品的价格平均值，获取平均价格大于100元的商品信息   
1）select category_id , avg(price) as ag from dw_goods group by goods_category having ag > 100   
2）select category_id , avg(price) as ag from dw_goods where ag>100 group by goods_category //报错！！因为from dw_goods 这张数据表里面没有ag这个字段    
注意:where 后面要跟的是数据表里的字段，如果我把ag换成avg(price)也是错误的！因为表里没有该字段。而having只是根据前面查询出来的是什么就可以后面接什么。   

#### mysql的复制原理大致如下
(1)首先，mysql主库在事务提交时会把数据库变更作为事件Events记录在二进制文件binlog中；mysql主库上的sys_binlog控制binlog日志刷新到磁盘。   
(2)主库推送二进制文件binlog中的事件到从库的中继日志relay log,之后从库根据中继日志重做数据库变更操作。通过逻辑复制，以此来达到数据一致。   
Mysql通过3个线程来完成主从库之间的数据复制：其中BinLog Dump线程跑在主库上，I/O线程和SQl线程跑在从库上。当从库启动复制（start slave）时，首先创建I/O线程连接主库，主库随后创建Binlog Dump线程读取数据库事件并发给I/O线程，I/O线程获取到数据库事件更新到从库的中继日志Realy log中去，之后从库上的SQl线程读取中继日志relay log 中更新的数据库事件并应用。   

Innodb的mvcc多版本并发控制用undolog控制，比如说你读取了一条数据，其他的事物再改，你读取的数据不变，甚至不是事务的修改，你读取的不会再变    
