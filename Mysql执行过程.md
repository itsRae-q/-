## SQL执行过程

#### Mysql基本架构
![image](https://user-images.githubusercontent.com/46525758/130794543-410ef705-2a96-484d-8812-a9a1a76f22d2.png)

    1.连接器
    连接器负责跟客户端建立连接、获取权限、维持和管理连接
    
    客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数 wait_timeout 控制的，默认值是 8 小时。
    
```sql
show processlist; #获取当前连接
```
![image](https://user-images.githubusercontent.com/46525758/130794620-668086c9-68d8-49ff-9ec8-3c8a3e67090e.png)

    2.优化器
    优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序
    
    3.执行器
    根据表引擎提供的接口获取数据或操作数据
    
#### redo log
    在InnoDB存储引擎层
    
    为什么需要redo log呢？
    每次更新都写磁盘需要随机IO，代价较大，将更新的数据线存到内存缓存中，提高效率，但如果mysql宕机了，怎么保证内存里的数据不会被丢弃？此时就需要记录每次更新的日志，并写到磁盘中，该日志就是redo log,redo log记录的是"物理级别"上的页修改操作，比如页号xxx、偏移量yyy写入了'zzz'数据.
    
    当向MySQL写用户数据时，先写redo log，然后redo log根据"某种方式"持久化到磁盘，变成redo log file，用户数据则在"buffer"中(比如数据页、索引页)。如果发生宕机，则读取磁盘上的 redo log file 进行数据的恢复。即MySQL 事务的持久性是通过 redo log 来实现的。在系统比较空闲的时候，将这个操作记录更新到磁盘里面（redo log日志还是在磁盘存储，只不过是顺序写不用随机IO，空闲时的操作是取更新磁盘数据？）
    
    InnoDB 的 redo log 是固定大小的，当写满的时候会清除最开始的记录
    
    
#### binlog归档日志
    在server层
    
    binlog三种格式：
    

| 格式 |说明  |
| --- | --- |
| STATMENT |每一条会修改数据的sql语句会记录到binlog中。优点：记录的日志比较少，性能较好，缺点：某些情况下会导致master-slave中的数据不一致，比如走的索引不一样导致更新结果不同或者获取当前日志的函数等  |
|ROW  |不记录每一条SQL语句的上下文信息，仅需记录哪条数据被修改了，修改成了什么样子了。优点：数据一致，缺点：会产生大量的日志，尤其是alter table的时候会让日志暴涨  |
| MIXED | 一般的复制使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog|

```sql
show variables like 'binlog_format';
```
![image](https://user-images.githubusercontent.com/46525758/130794723-17c8e283-a35d-42ee-80f8-92d573356bbd.png)


#### 更新语句执行过程
`update T set c=c+1 where ID=2;`

    浅色框表示是在 InnoDB 内部执行的，深色框表示是在执行器中执行的

![image](https://user-images.githubusercontent.com/46525758/130794780-c97f40bf-1801-4ff5-bbc9-8fa580cf5f3b.png)

    更新时采用了WAL技术（Write-Ahead Logging）,先写日志（Redo Log），在更新磁盘数据（将随机磁盘IO优化为顺序磁盘IO）
    
    Redo Log文件是固定大小的

![image](https://user-images.githubusercontent.com/46525758/130794847-263f3636-77ce-4185-9858-eac91786730c.png)

    write pos和check point之间是空闲部分，当写满的时候则不能在进行更新操作了，需要擦除的redo log数据，具体来讲
    可以看出，这里redo log的写入采用了两阶段提交，目的是让两个日志在逻辑上保持一致状态。
    
    如果不用两阶段提交，更新语句将id值由1变为2，写了redolog,然后Mysql崩了，此时没有写binlog,当mysql重启后，可以根据redo log将id变为2这条数据恢复，但是之后如果用这个binlog恢复数据库的话会缺少此部分更新，即根据binlog复制出来的数据库id值为1，导致数据不一致
    
    两阶段提交如何恢复数据？
    顺序扫描redo log
    1. 如果是在redo log里事务是完整的，即处于commit状态，说明redo log和bin log都写入成功，直接提交事务
    2. 如果redo log里事务状态的prepare，则需要用redo log中XID数据字段去Bin log找对应事务
        a)如果存在且完整则提交事务
        b)如果不存在或者未写完则回滚
   
     此处注意点：
     1. 为什么redo log处于commit状态时，异常恢复的时候要提交事务，不回滚？
     因为此时bin log已经写入完成，之后可能被用于同步从库或者恢复数据表等，所以需要提交该事务
     
     2.只用redo log，不用binlog可以吗
       不行
       a) redo log是循环写入，历史数据会被抹掉
       b) binlog有其他用处（高可用基础（待看），消费binlog更新数据等）
     
     3. redo log是什么时候写入的呢？
     redo log可能存在三个位置：
     1）redo log buffer中，即Mysql内存进程中
     2）文件系统的page cache里
     3）持久化到磁盘中
     写入redo log buffer和文件系统较快，刷磁盘就比较慢了
     
     因此mysql提供了innodb_flush_log_at_trx_commit 参数用于控制刷盘的时机，有三种取值：
     1）设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中
     2）设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘
     3）设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache
     InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。

![image](https://user-images.githubusercontent.com/46525758/130794898-12a30b35-0841-404d-8e9f-b33c7abd7cf8.png)

`show variables like 'innodb_flush_log_at_trx_commit';
`

![image](https://user-images.githubusercontent.com/46525758/130794947-d705a6f2-3183-49c8-bc91-600a42a2c050.png)

    当innodb_flush_log_at_trx_commit 设置成 1，那么 redo log 在 prepare 阶段就要持久化一次，因为在Mysql异常恢复的时候需要用到redo log+binlog
    

* * *

    binlog写入时机
    
    事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中

    一个事务的 binlog 是不能被拆开的，因此不论这个事务多大，也要确保一次性写入
    
    通过参数sync_binlog 控制binlog写入时机：
    1.sync_binlog=0 的时候，表示每次提交事务都只 write，不 fsync；
    2.sync_binlog=1 的时候，表示每次提交事务都会执行 fsync；
    3.sync_binlog=N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。
