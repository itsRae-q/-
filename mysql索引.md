# mysql索引

### 索引模型
    1.哈希索引
    哈希函数+拉链法解决哈希冲突
    等值查询很快，通过哈希函数计算即可，但由于索引是无序的，做范围查询的时候需要遍历全部，效率很低
    
    2.有序数组
    递增排列，用二分法等值查询，有序保证范围查询效率，更新效率低，需要移动后续元素
    
    3.B+树
    多叉树，非叶子节点只存下一层指针，不存数据->树高小->磁盘IO少
    
### InnoDB索引模型
#### 主键索引-非主键索引
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  `k` varchar(20),
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

维护了两棵索引树 id和c
```

* * *

>select * from t where id=1;
>搜索id索引的B+树
>select * from t where c=1;
>搜索c索引的B+树+回表搜索id索引树
>尽量使用主键查询

    为什么建表的时候一般都设一个自增主键bigint呢？
    1.依次递增，避免索引数据页出现分裂或者合并操作
    2.二级索引数叶子节点存主键值，bigint占8个字节比较省空间，优于一般的业务主键

* * *

#### 覆盖索引
    一个例子
    select * from t where c between 3 and 5;
    
    1) c索引树上找到3记录，获取节点存的主键id值
    2）查找主键索引树数据
    3）取索引c中3的下一条记录
    4）回表...
    

* * *


    使用覆盖索引可以避免回表(性能优化) 
    select id from t where c=3;
    索引c叶子节点存了主键id值不用回表
    性能优化：高频请求建立覆盖索引(联合索引)
    
#### 最左前缀匹配+索引下推
    
    最左匹配，以最左边的为起点任何连续的索引都能匹配上。同时遇到范围查询(>、<、between、like)就会停止匹配
    
    构建一颗B+树只能根据一个值来构建，因此数据库依据联合索引最左的字段来构建B+树，如构建(c,d)的联合索引，则c是有序的，d无法保证有序，只有在c确定的情况下d才是有序的，因此当a为范围查询时，b就不会再走索引
    
    小tips：
        建立联合索引的时候通过调整顺序可以少建索引，如要查c,同时也需要查c,d，则建立c,d的联合索引，不建立d,c的
        如果c字段比d字段大，则建立c,d较好，因为如果建立d,c的话在建一个c索引，c索引树占用空间更大
        
        ———————— todo: explain分析最左匹配下选用的索引————————
       

* * *

`select * from t where k like "12%" and c=10;`（假设建立了k,c的联合索引）

    mysql5.6之前，会走k索引，将拿到的值一一回表去查，不会按照c过滤，但其实(c,k)联合索引中已经可以过滤c!=10的结果了，这部分数据不用回表再查----索引下推优化
    
    
#### 唯一索引和普通索引
    查询: 普通索引查到值后还会顺序遍历叶子节点直到不满足的值出现，而唯一索引找到就可以直接返回，但InnoDB是按数据页为单位读数据的，因此两者查询效率差别不大
    

* * *
    
    更新：由于普通索引可以用change buffer，更新时不必每次都将数据从磁盘读入内存，效率更高
    当要插入的数据页不在内存中时，唯一索引需要将数据页读入内存，判断是否有冲突，然后插入；而对于普通索引来说，只需要将更新记录在change buffer中即可，避免了随机磁盘访问，当后续访问到该数据页会进行merge操作，将变更应用于数据页，后台线程也会定期merge
    ----- 写多读少的场景适合普通索引-----
    
#### 前缀索引-倒序存储-哈希字段
    mysql支持创建索引的时候指定前缀长度
    `alter table t add index idx_k(k(6))`
    创建索引的时候只取前6位
    前缀索引可以节省空间，但索引区分度下降，需要不停回表去查完整数据，同时也无法使用覆盖索引
    
    
### SQL优化-Explan学习
```sql
explain select * from sku_cache_tab where sku_id="3000153181_2000634859";
```
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p6![image](https://user-images.githubusercontent.com/46525758/127452675-013b9038-b23f-499f-9c0c-9dd4a73a9f3a.png)

* * *
#### id列
    多个select中,id越大越先执行
    id相同，在上面的先执行
    
    
#### select_type列
    1) SIMPLE：简单查询，不包含子查询
    explain select * from sku_cache_tab where sku_id="3000153181_2000634859";
    
    2) PRIMARY: 外部主查询
    
    3）derived: from后的子查询（衍生表）
    
    4）subquery: from前的子查询

先关闭临时表优化小选项：`set session optimizer_switch = `derived_merge=off`;
`
执行` explain select (select 1 from sku_cache_tab where id=1) from (select * from sku_cache_tab where sku_id="446875364_0") t1;`

evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p10![image](https://user-images.githubusercontent.com/46525758/127452776-80a03ec6-c596-45cc-ae34-7fe4255c8d42.png)

    id越大越先执行，先执行的是from后面的`select * from  sku_cache_tab where sku_id="446875364_0"`，属于from后的子查询类型
    然后执行select后面的`select 1 from sku_cache_tab where id=1`，属于from前的子查询
    最后执行最外部的select..from..，属于外部主查询
    5）union: 联合查询
    
#### table列
    正在访问哪张表
    
#### type列
    比较重要，判断当前sql的性能
    null > system > const > eq_ref > ref > range > index > all
    
    尽量保证tyoe列在range及以上
    
    1）null
    explain select min(id) from sku_cache_tab;
    
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p13![image](https://user-images.githubusercontent.com/46525758/127452802-5f0c39ba-2307-44fb-ac7f-f5532f162b9b.png)

    使用聚合函数对主键直接进行操作，直接从索引树获取
    
    2）system
    直接合一条记录匹配
    
    3) const
    主键索引或唯一索引和常量进行比较
    explain select * from sku_cache_tab where sku_id="3000153181_2000634859";

evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p14![image](https://user-images.githubusercontent.com/46525758/127452819-a8f59830-316d-4cd4-924e-3518025ec0cd.png)

    4) eq_ref
    连接查询时，用本表的主键进行关联
    explain select * from inbound_v2_tab a left join inbound_sku_list_v2_tab b on a.inbound_id=b.id;
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p15![image](https://user-images.githubusercontent.com/46525758/127452834-dbd29817-72c8-4838-a3ae-5d9cf3dd9251.png)

    5) ref
    简单查询：索引是普通索引
    explain select * from sku_cache_tab where shopid=12313132;
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p16![image](https://user-images.githubusercontent.com/46525758/127452865-ae2ba5cd-7d16-47d1-8688-f2e38484f880.png)

    复查查询：用本表的普通索引进行连接
    explain select * from inbound_v2_tab a left join inbound_sku_list_v2_tab b on a.inbound_id=b.inbound_id;

evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p17![image](https://user-images.githubusercontent.com/46525758/127452888-6fa69be9-73ce-4aef-a3fc-c43846b7a3a4.png)

    6) range
    索引列上的范围查找
    explain select * from sku_cache_tab where id>100;
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p18![image](https://user-images.githubusercontent.com/46525758/127452911-66abf126-3d48-4c72-85ec-3a54ee732404.png)

    7）index
    查询所有的记录，但是直接从索引树上获取的
    explain select category_id_l1, category_id_l2 from sku_cache_tab ;
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p19![image](https://user-images.githubusercontent.com/46525758/127452929-9abe3063-2701-49fc-8dc7-51b6dfe0a866.png)

    category_id_l1, category_id_l2是联合索引，可以直接从索引树拿到
    8）All
    全表扫描，需要优化
    explain select * from sku_cache_tab where status=1;
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p20![image](https://user-images.githubusercontent.com/46525758/127452946-5829dbde-71f6-481b-ba85-612f841cd874.png)


#### possible keys
    查询中可能用到的索引
    mysql内部优化器会进行判断，如果走索引性能比全表查还蛮，则全表查
    explain select * from sku_cache_tab where country like "I%";
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p21![image](https://user-images.githubusercontent.com/46525758/127452968-71adfaa5-2d12-4fe1-941c-688e62018292.png)

    因为走索引还需要在进行回表捞数据，如果索引区分度很差，则性能不如全表扫描
    
#### rows
    可能要查询的数据
    
#### key_len
    键的长度，可以判断出联合索引命中了哪几个
    explain select * from sku_cache_tab where category_id_l1=123 and category_id_l3=234;

evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p22![image](https://user-images.githubusercontent.com/46525758/127452996-ce0f9a0a-b09e-4ecc-9d09-4582472bf288.png)

    可以看出没有命中category_id_l3
     varchar(n)-> 4n+2?
     
 #### extra列
     是否使用了覆盖索引，文件排序等
     
     1. using index
     使用到了覆盖索引
     explain select category_id_l1, category_id_l2, category_id_l3 from sku_cache_tab where category_id_l1=123132;
evernotecid://B48FD0DA-C02D-4121-889A-411E48BFD9BB/appyinxiangcom/36139734/ENResource/p23![image](https://user-images.githubusercontent.com/46525758/127453014-7de5dee0-d62f-4325-888d-198b88df25a2.png)
 
     2.using where
     使用了索引进行范围查找
     
     3.using index condition
     where命中索引，建议使用覆盖索引进行优化
     
     4. using temporary
     在没有索引的列上进行去重操作，会创建临时表进行操作
     explain select distinct status from sku_cache_tab;
     
     5. using filesort

#### Explain优化实践 TODO
#### trace工具使用 todo
#### sql优化
