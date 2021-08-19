# SQL优化
## order by
```sql
explain select * from inbound_sku_list_v2_tab where inbound_id="INPUTHA2020090700002" order by expected_count;
```
![image](https://user-images.githubusercontent.com/46525758/129135251-8d412ff8-055e-476c-98e8-c4c01e87fc72.png)


    inbound_id是varchar(32), key_len=130,用到了inbound_id索引
    extra: Using filesort- 需要排序
    
    MySQL 会给每个线程分配一块内存用于排序，称为 sort_buffer。
    
    执行过程：
    1）初始化sort_buffer
    2) 根据索引找到第一个满足条件的值，拿到主键id
    3）根据主键id回表查询，取出证行数据放入sort_buffer
    4) 去下一个值...
    5）对sort_buffer中的数据进行排序
    （排序的数量小于sort_buffer大小，内部排序，大于则需要利用磁盘临时文件进行外部排序（一般为归并排序））
    

* * *

    rowid排序
    
    如果单行数据太大，则sort_buffer中可存放的数据越少，则会分成很多临时文件进行排序，效率低，此时mysql会只把主键和待排序字段放入sort_buffer
    
    mysql参数: max_length_for_sort_data 控制用于排序的行数据的长度 (test环境该值为1024)，单行长度大于该值则采用rowid排序
    
    执行过程：
    1.初始化sort_buffer,只放带排序字段和主键id
    2.根据索引找到第一条数据主键id
    3.回表，将带排序字段+id放入sort_buffer
    4.继续查下一个...
    5.sort_buffer中数据排序
    6.按照排好的id再次回表查询完整数据
    --- 多了一次回表操作
    
    优化点：待排序字段加入联合索引中避免进行排序
    例如：
```sql
explain select * from inbound_sku_list_v2_tab where inbound_id="INPUTHA2020090700002" order by sku_id;
```
![image](https://user-images.githubusercontent.com/46525758/129135351-b37c990a-9a4e-4753-93d2-2d30c108ce16.png)


    由于该表上有inbound_id_sku_id的联合索引，当根据inbound_id索引找到节点时，sku_id本身就已经排好序了，因此不用再排序，此时执行过程如下：
    1.找到满足条件的第一个节点及主键id
    2.回表取完整数据
    3.继续找下一个...
    
    但如果where条件中是范围查询，则还是需要排序的，因为联合索引中sku_id不是全局有序的
    
```sql
explain select * from inbound_sku_list_v2_tab where inbound_id in ("INPUTHA2020090700002", "INPUTHA2020090800011") order by sku_id;
```
![image](https://user-images.githubusercontent.com/46525758/129135682-f512583c-7284-4653-a3b1-257dfa8fb665.png)

```sql
explain select * from inbound_sku_list_v2_tab where inbound_id like "IN%" order by inbound_id;
```
![image](https://user-images.githubusercontent.com/46525758/129135387-ee33924e-0228-4f7c-a7a4-b4ee19e63066.png)

    此处由于inbound_id的like查询区分度太低，mysql优化器会进行全表扫描，不走索引，因为也会进行文件排序

## join
```sql
explain select * from inbound_v2_tab a left join inbound_sku_list_v2_tab b on a.inbound_id=b.inbound_id;
```
![image](https://user-images.githubusercontent.com/46525758/129135405-2593e06d-fe6c-4712-b575-caa505fad03d.png)

    NLG算法
    执行过程：
    1.从inbound_v2_tab中取一行数据
    2.去除inbound_id，去inbound_sku_list_v2_tab按照索引查找
    3.组合结果，循环。。。
    
    其中，inbound_v2_tab称为驱动表，inbound_sku_list_v2_tab为被驱动表，连接时用到了被驱动表的索引-NLJ
    
    怎么选择驱动表呢？
    驱动表全表查询，被驱动表走索引查询
    假设驱动表行数是N，被驱动表行数是M
    全表扫描驱动表，行数为N，然后对于每一个驱动板的数据，按照索引扫描被驱动表log(M),然后回表查被驱动表数据log(M),因此总复杂度为：N+N*2*log(M),可以看出N越小越好，因此需要选用小表做驱动表
    
```sql
explain select * from inbound_v2_tab a left join inbound_sku_list_v2_tab b on a.mtime=b.mtime;
```
![image](https://user-images.githubusercontent.com/46525758/129135454-19332318-5e55-4064-8394-f8641a3f9826.png)

    BNLG算法:关联的键没有索引时，会启用BNLG算法，创建一个join buffer缓冲区，将驱动表中的数据存进来，再和被驱动表数据进行比较，减少了磁盘IO过程，时间复杂度一样但是效率更高（如果join buffer放不下整个驱动板则会分段放）
    

* * *


    略看
    Multi-Range Read优化
    尽量使用顺序读盘
    在回表的过程中，非主键索引叶子节点的主键值查出来不是有序的，MRR是对查出来的主键值进行了一次排序（如果想要稳定地使用 MRR 优化的话，需要设置set optimizer_switch="mrr_cost_based=off，否则优化器可能会不使用MRR）
    在NLG算法中会将驱动板数据查出来放到join buffer中进行排序，在查被驱动表
    
    
    

## Group By
    原理是先排序后分组


## 分页查询
```sql
explain select * from inbound_sku_list_v2_tab limit 10000, 10;
```
![image](https://user-images.githubusercontent.com/46525758/129135484-23a50ec3-ce8b-4fc1-bac7-e0a142efef56.png)

获取10010条数据并舍弃前10000条

```sql
explain select * from inbound_sku_list_v2_tab a inner join (select id from inbound_sku_list_v2_tab limit 10000,10) b on a.id=b.id;
```
![image](https://user-images.githubusercontent.com/46525758/129135518-c2a93365-924d-4de1-9aed-7bf3982929d1.png)

    1.id越大越先执行，即先执行
    select id from inbound_sku_list_v2_tab limit 10000,10
    type是index，表明是直接从索引树上取值，然后limit 10000,10，此时取出10条id
    2.id相同在上面的先执行，即执行的是临时表
    select * from (select id from inbound_sku_list_v2_tab limit 10000,10)
    在进行连表查询的时候，mysql优化器会将小表作为驱动表，因此会先执行该衍生表，type是ALL，会进行全表扫描，但此时只会扫描10条id记录
    3.执行连表查询，对2中扫描到的每一行记录，去inbound_sku_list_v2_tab查询，此时会走主键索引，因此type是eq-ref
    
    总体来说，优化后其实是将ALL类型的全表扫描优化为index类型
    
    注意，在没有order by的情况下，select * from ... 和select id from ...搜索出的数据是不一致的
    
## in和exist

## count
```
explain select count(*) from inbound_sku_list_v2_tab;
```
![image](https://user-images.githubusercontent.com/46525758/129135536-6b434964-f0d8-42cd-be45-d55fe658e342.png)

    在count(*)的时候，是去sku索引进行遍历，计算叶子节点总数的。主键索引叶子节点存的是数据，而非主键索引存的是主键id，显然非主键索引树更小，因此是取非主键索引查
    
    为什么不直接将总数存起来呢？
    因为innodb的MVCC机制，同一时刻不同事务查询表的总数是不确定的，例如在RR隔离级别下：
    

| session A | session B |
| --- | --- | 
| begin; |  |  |
| select count(*) from t |  |  
|  | begin; |  
| | insert into t;| 
| | select count(*) from t;| 
|select count(*) from t;(应该返回10000)|select count(*) from t;（应该返回10001）|

    
    对于频繁需要查总数的表，由于count(*)也是需要遍历索引树上所有节点的，为了性能更好可以将表的总数缓存起来
    
    1. 用redis缓存
    由于写缓存和写表不是原子操作，因此会出现不一致的情况
    例如：查询计数及最近100条记录
    

|时刻| session A |session B  |
|---| --- | --- |
|T1| redis计数+1 |  |
|T2|  | 读redis计数，去表中查100条记录 |
|T3|表中插入数据  |  |

    sessionB在T2时刻读总数的时候数据还没有插入，但是已经读到了+1后的总数及新插入的记录
    
    2. 用单独的表存总数
    利用事务隔离性可以解决上面的问题
    

| 时刻 | sessionA | sessionB  |
| --- | --- | --- |
|  | begin; |  |
| T1 | 表中计数+1 |  |
|  |  | begin; |
|T2||读表中计数；查表中100条记录|
||插入数据；||
||commit;||

    在SessionB T2时刻读表中计数的时候，由于隔离性，在SessionA提交之前，SessionB看不到计数增加和数据插入，保证了计数和查询数据的统一性（快照度保证统一性，当前读会被阻塞，在事务提交后查到的数据也是统一的）
    
## SQL注意点
    1.不能再索引列上做计算，函数，类型转换(会导致索引失效)
    
```sql
explain select * from inbound_sku_list_v2_tab where id+1=1130265;
;
```
![image](https://user-images.githubusercontent.com/46525758/129135569-5bd013d6-ce85-4483-9a22-60f1f6df0e5f.png)

    函数运算会破坏索引有序性，但其实即使不破坏mysql也不走
    
    2.尽量使用覆盖索引
    
    3.使用不等于导致全表扫描
    这个该怎么优化？
    
```sql
explain select * from inbound_v2_tab where inbound_id != "INPUTHA2020090700002";
```
![image](https://user-images.githubusercontent.com/46525758/129135589-d117ac9a-aaf9-4fa4-a44e-9d8c99af49b5.png)

    4.使用is null, is not null会导致全表扫描
    
    5. like通配符在前面导致全表扫描
    
    6. 类型转换导致全部扫描
```sql
explain select * from inbound_v2_tab where supplier="846005";   (1)

explain select * from inbound_v2_tab where supplier=846005;     (2)
```

![image](https://user-images.githubusercontent.com/46525758/129135614-8412f3a2-0b10-4442-81b6-182fa04da825.png)
![image](https://user-images.githubusercontent.com/46525758/129135617-b986bc0a-4799-4c60-be20-84db3a4dfa25.png)

    在Mysql中，字符串和数字进行比较的时候，是将字符串转为数字
    因此在（2）中需要遍历所有行
    
    7.大in操作或者宽范围查找可能不走索引
    切分，多次查询
    
    8. 连接查询时需要在关联字段上建索引，且保证类型一致，避免索引失效
   

## 倒序索引（Descending Indexes）
    假设表t中字段a,b建立联合索引idx_a_b, 进行单字段排序时，会找到联合索引树中最右边的节点，然后依次向左遍历获取结果
    
```
select a,b from t order by a desc;
```
![image](https://user-images.githubusercontent.com/46525758/129996334-11e50427-8c82-4f23-9258-63eb4b8729bb.png)

    如果创建的是升序索引，那么对于组合字段的排序，会有以下情况：
    
```sql
explain select inbound_id, sku_id from inbound_sku_list_v2_tab order by inbound_id asc , sku_id asc ;
```
![image](https://user-images.githubusercontent.com/46525758/129996380-b566ea3a-54b3-4f9a-ba87-51bf85c12339.png)

    此时不需要filesort,因为联合索引按照Inbound_id升序，且inbound_id一定时sku_id升序排列的
    
 ```sql
explain select inbound_id, sku_id from inbound_sku_list_v2_tab order by inbound_id desc , sku_id desc ;
```
![image](https://user-images.githubusercontent.com/46525758/129996415-c1ddf9ec-2d75-4705-89a0-e451b1a59f16.png)
    
    此时也不需要filesort，找到最后一个节点从右向左遍历极即可

 ```sql
explain select inbound_id, sku_id from inbound_sku_list_v2_tab order by inbound_id asc , sku_id desc ;
```
![image](https://user-images.githubusercontent.com/46525758/129996449-2321458e-f6be-4bc5-b999-c5baf605433b.png)

    此时需要filesort，因为inbound_id确定时，sku_id不是倒序的
    
    如果建立倒序索引则可以避免多字段排序顺序不同时需要filesort的情况
    
[倒序索引](https://dev.mysql.com/doc/refman/8.0/en/descending-indexes.html)
![image](https://user-images.githubusercontent.com/46525758/129996481-7e765950-9f78-4880-adfe-34f8755331dd.png)

    mysql8.0版本支持该功能
    
![image](https://user-images.githubusercontent.com/46525758/129996527-3bd54756-82df-49d1-a797-fa65a27144db.png)

    对于不支持倒序索引的可以怎么处理呢？
    例如:
```sql
explain select inbound_id, sku_id from inbound_sku_list_v2_tab order by inbound_id asc , sku_id desc ;
```
    1.先写sql按inbound_id,sku_id升序捞数据
    2.构造空栈，写入第一条数据
    3.读入下一行，
    a.如果新一行中a值与上一行相同，将新一行入栈；
    b.如果新一行中a值与上一行不同，则将栈中的所有数据行依次出栈并输出，直到栈清空；然后新一行入栈。
