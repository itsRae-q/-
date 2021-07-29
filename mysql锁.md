# mysql锁
## 隔离级别
## MVCC
## 表锁和行锁
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

## 幻读
定义: 幻读指的是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行

| session1 |  session2|
| --- | --- |
| begin ;|  |
| select * from t where d=5 for update ;(5,5,5) |  |
||insert into t values (1,1,5);|
|select * from t where d=5 for update ;(5,5,5),(1,1,5)||

1.innodb设为可重复度的隔离级别时，由于加了间隙锁，是不会出现幻读的，此处幻读的基于只加行锁假设的；
2. 幻读在当前读情况下才出现，快照读是不会去同步当前表最新数据的，因此无法查到其他事务插入的数据

幻读带来的问题：
1. 破坏语义
   上例session1中第一条语句是将d=5的所有记录锁住，但session却插入了d=5的数据，也可以通过id去更新该行数据，即新插入的d=5的数据没有被锁住
2. 数据一致性问题

| session1 |session2  |
| --- | --- |
| begin ; |  |
| select * from t where d=5 for update ; |  |
|update t set c=100 where d=5;||
||begin;|
||insert into t values (1,1,5);|
||update t set c=200 where id=1;|
||commit;|
|commit;||

表里的数据应该为(id=5,c=100,d=5),(id=1,c=200,d=5)
写入binlog语句为
```sql
insert into t values (1,1,5);
update t set c=200 where id=1;
update t set c=100 where d=5;
```
binlog执行结果为(id=5,c=100,d=5),(id=1,c=100,d=5)
id=1这一行记录与表里数据不一致
导致幻读的原因：锁住的是行，但是操作的是行间的间隙

## 间隙锁

间隙锁(Gap Lock)，锁的就是两个值之间的空隙
执行 select * from t where d=5 for update 的时候，就不止是给数据库中已有的 6 个记录加上了行锁，还同时加了 7 个间隙锁（由于d列没有索引，会导致全表扫描）

>注意：间隙锁之间不冲突，跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作

间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间

间隙锁的引入其实扩大的加锁范围，同时间隙锁间是不冲突的，容易引起死锁


# 加锁实践
以下加锁分析都基于可重复度隔离级别
>加锁原则
>原则 1：加锁的基本单位是 next-key lock。
>原则 2：查找过程中访问到的对象才会加锁。
>优化 1：索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。
>优化 2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。
>一个 bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止
>（高版本已修复，wms用的版本为5.7.29，还存在该问题）

### 1. 唯一索引等值查询
#### 1) 查找到
| session1 |session2  |
| --- | --- |
| begin; |  |
| select * from t where id=5 for update; (只锁id=5的记录)|  |
|  |begin;  |
|  | insert into t values(6,6,6); 不阻塞|

id为主键,session1的id=5命中,next-key lock退化为行锁，只锁id=5这一行,session2中插入id=6的记录，由于间隙(5,10)没有被锁，因此不阻塞

#### 2） 未找到

| session1 |session2  |session3 |
| --- | --- | --- |
| begin; |  | |
| select * from t where id=7 for update; (锁(5,10)的间隙)|  |
|  |begin;  |
|  | insert into t values(8，8，8); 阻塞|
|  | |begin;|
|||select * from t where id=10 for update; (ok)|

加锁单位为next-key-lock, session1加(5,10], 由于id=10不满足条件，根据优化2，退化为间隙锁(5,10),因此session2阻塞,session3成功

### 非唯一索引等值查询

| session1 |session2  |
| --- | --- |
| begin; |  |
| select * from t where c=5 for update; 锁(0,5],(5,10)|  |
|  | begin; |
|  | insert into t values(7,7,7) 阻塞 |
||update t set d=6 where id=5; 阻塞|

c=5加next-key lock锁(0,5], 由于是非唯一索引，因此需要继续向右遍历，查到c=10,加next-key lock 5,10],由于c=10不满足条件，退化为间隙锁(5,10)

然而，对于以下场景：

| session1 |session2  |
| --- | --- |
| begin; |  |
| select id from t where c=5 lock in share mode; 锁(0,5],(5,10)||
|  | update t set d=6 where id=5; (ok) |

此时session2不会被阻塞，原因：
1. 锁是加在索引上的，不是记录行上
2. 只有访问到的对象才会加锁
对于session1的sql, 由于只检索id字段，且为lock in share mode, 而c列对应的索引树的叶子节点存储着主键值，相当于覆盖索引，不需要访问主键索引，因此此时只是在c列上加锁，不会锁主键列，因此session2的sql不会阻塞

> 注意：加锁进行更新数据时，要么加写锁，如果加读锁需要绕开覆盖索引

非唯一索引相同值的间隙
```sql
insert into t values(30,10,30);
```

| session1 |session2  |session3  |session4  |
| --- | --- | --- | --- |
| begin; |  |  |  |
| select * from t where c=10 for update; 锁住(c=5, id=5)到(c=15,id=15)这个间隙及中间的记录 |  |  |  |
|  | begin; |  |  |
|  | insert into t values(4,5,50); (ok)没有锁住(c=5,id=4这个间隙) |  |  |
|  |  |begin;  |  |
|  |  | insert into t values(6,5,50); (阻塞) |  |
|  |  |  | begin; |
|  |  |  |select * from t where id=10 for update;(阻塞)其他id不阻塞  |

对于索引c来说，索引树叶子节点大致如下：
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p4![image](https://user-images.githubusercontent.com/46525758/127452290-e3212fc5-662d-407d-a7ef-41ffa02140a2.png)


对于索引c来说，session1中sql锁住了(5,10],(10,15)这个间隙，二级索引c叶子节点存储主键值，而二级索引叶子节点是有序的，对于session2中sql来说，(c=5,id=4)没有在这个间隙内，可以插入，但是session3中的(id=6,c=5)在间隙内，被阻塞。另外，session1中sql还给主键索引加了id=10的行锁，因此session4中的语句被阻塞，插入其他id时是正常的

### 唯一索引范围查询

| session1 |session2  |session3  |
| --- | --- | --- |
| begin; |  |  |
|select * from t where id>=10 and id<11 for update; 加锁10, (10,15]  |  |  |begin;||
|  |insert into t values(8,8,8) (ok) |  |
|  |insert into t values(13,13,13) 阻塞 |  |
|||begin;|
|||select * from t where id=15 for update;(阻塞)|

对于session1的sql, 首先等值查询id=10, 唯一索引等值查询找到时会退化为行锁,此时只锁id=10这一行记录，接下来进行范围查找，向右遍历，找到id=15这一条记录不满足条件，加next-key-lock (10,15],此处注意只有等值查询向右遍历不满足条件时才会退化为间隙锁，而范围查询不会，因此最终锁为id=10的行锁和(10,15]

### 非唯一索引范围查询

| session1 |session2  |session3  |
| --- | --- | --- |
| begin; |  |  |
|select * from t where c>=10 and c<11 for update; 加锁(5,10], (10,15]  |  |  |
||begin;||
||insert into t values(8,8,8) 阻塞||
|||begin;|
|||select * from t where id=15 for update;(阻塞)|

与唯一索引不同，在进行c=10这一等值查询的时候，会加next-key lock (5,10], 接下来进行范围查找与之前一致，锁(10,15]

### 唯一索引bug加锁

| session1 |session2  |session3  |
| --- | --- | --- |
| begin; |  |  |
|select * from t where id>10 and id<=15 for update; 加锁(10,15], (15,20]  |  |  |
||begin;||
||select * from t where id=20 for update; 阻塞||
|||begin;|
|||insert into t values(16,16,16); 阻塞|

唯一索引扫描到id=15这一条记录时，满足条件，因此会继续向右扫描，扫描到id=20这一条记录，加next-key lock (15,20],其实扫描到id=15就知道后面不满足了，可以停止扫描，减少(15,20]这一个锁

### limit语句可以减少加锁范围

| session1 |session2  |
| --- | --- |
| begin; |  |
|  select * from t where c=10 for update limit 2;|  |
|  | begin; |
|  | insert into t values(11,11,11); |

与非唯一索引等值查询中最后一个例子进行比较，由于加了limit 2的限制，扫描到c=10的节点后已经找到了2个，因此减少了（10，15）的间隙锁，此时加锁如下：
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p5![image](https://user-images.githubusercontent.com/46525758/127452428-91f32fc5-3087-4a0d-8b81-c604748a2c27.png)



# 死锁
#### 间隙锁不冲突引起死锁
1. 例如这样的业务逻辑：如果不存在就插入，存在则跳过

| session1 |session2  |
| --- | --- |
|begin;  |  |
|select * from t where id=9 for uodate; (锁住(5,10)间隙) |  |
|  |begin;  |
|  |select * from t where id=9 for uodate; (间隙锁不冲突，锁住(5,10间隙)) |
| insert into t values(9,9,9) 阻塞 |  |
||insert into t values(9,9,9) 死锁|


2. next-key lock实际上是分两步加的

| session1 |session2  |
| --- | --- |
|begin;  |  |
|select * from t where c=10 for update;   |  |
|  | begin; |
|  | select * from t where c=10 for update; (阻塞) |
|insert into t values(8,8,8); 死锁  |  |

session1中sql锁住(5,10],(10,15)，session2中要加同样的锁，先执行next-key lock(5,10],分两步，第一步加间隙锁(5,10),由于间隙锁不冲突，这一步成功，第二步加c=10行锁，被阻塞，session1要插入c=8的记录，对应的间隙(5,10)被session2锁住，造成死锁。

上面两例都是 两个session获取到同一个间隙锁，然后向该间隙插入数据导致的死锁。
