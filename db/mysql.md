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
