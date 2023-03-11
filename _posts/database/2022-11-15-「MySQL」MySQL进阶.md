---
layout:     post
title:      "「MySQL」 MySQL进阶"
subtitle:   "MySQL进阶"
date:       2022-07-31 12:00:00
author:     "yosin"
header-img: "img/post-bg-html-css.jpg"
katex: true
tags:
    - MySQL
---

## SQL 进阶

### SQL优化

#### 插入数据

- insert优化

如果我们需要一次性往数据库表中插入多条记录，可以从以下三个方面进行优化：

1. 批量插入数据

   ```sql
   Insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry');
   ```

2. 手动控制事务

   ``` sql
   start transaction; 
   insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry'); 
   insert into tb_test values(4,'Tom'),(5,'Cat'),(6,'Jerry'); 
   insert into tb_test values(7,'Tom'),(8,'Cat'),(9,'Jerry'); 
   commit;
   ```

3. 主键顺序插入，性能要高于乱序插入。



- load指令：大批量插入数据

#### 主键优化

在InnoDB引擎中，数据行是记录在逻辑结构 page 页中的，而每一个页的大小是固定的，默认16K。那也就意味着， 一个页中所存储的行也是有限的，如果插入的数据行row在该页存储不小，将会存储到下一个页中，页与页之间会通过指针连接。

##### 顺序插入

①. 从磁盘中申请页， 主键顺序插入

![image-20230303110323912](/img/Database/advanced/image-20230303110323912.png)

②. 第一个页没有满，继续往第一页插入

![image-20230303110346708](/img/Database/advanced/image-20230303110346708.png)

③. 当第一个也写满之后，再写入第二个页，页与页之间会通过指针连接

![image-20230303110405085](/img/Database/advanced/image-20230303110405085.png)

④. 当第二页写满了，再往第三页写入![image-20230303110456881](/img/Database/advanced/image-20230303110456881.png)

##### 乱序插入：页分裂

①. 加入1#,2#页都已经写满了，存放了如图所示的数据![image-20230303110540876](/img/Database/advanced/image-20230303110540876.png)

②. 此时再插入id为50的记录，不会直接写入新的页。因为索引结构的叶子节点是有顺序的。按照顺序，应该存储在47之后。

![image-20230303110628578](/img/Database/advanced/image-20230303110628578.png)

![image-20230303110643892](/img/Database/advanced/image-20230303110643892.png)

但是47所在的1#页，已经写满了，存储不了50对应的数据了。 那么此时会开辟一个新的页 3#。

![image-20230303110705961](/img/Database/advanced/image-20230303110705961.png)

但是并不会直接将50存入3#页，而是会将1#页后一半的数据，移动到3#页，然后在3#页，插入50。

![image-20230303110726645](/img/Database/advanced/image-20230303110726645.png)

移动数据，并插入id为50的数据之后，那么此时，这三个页之间的数据顺序是有问题的。 1#的下一个页，应该是3#， 3#的下一个页是2#。 所以，此时，需要重新设置链表指针。

![image-20230303110809939](/img/Database/advanced/image-20230303110809939.png)

##### 删除页：页合并

目前表中已有数据的索引结构(叶子节点)如下：

![image-20230303110904077](/img/Database/advanced/image-20230303110904077.png)

当我们对已有数据进行删除时，具体的效果如下:

当删除一行记录时，实际上记录并没有被物理删除，只是记录被标记（flaged）为删除并且它的空间

变得允许被其他记录声明使用。

![image-20230303110931146](/img/Database/advanced/image-20230303110931146.png)

当我们继续删除2#的数据记录![image-20230303110948485](/img/Database/advanced/image-20230303110948485.png)

当页中删除的记录达到 MERGE_THRESHOLD（默认为页的50%），InnoDB会开始寻找最靠近的页（前

或后）看看是否可以将两个页合并以优化空间使用。

![image-20230303111014543](/img/Database/advanced/image-20230303111014543.png)

删除数据，并将页合并之后，再次插入新的数据21，则直接插入3#页![image-20230303111040632](/img/Database/advanced/image-20230303111040632.png)

#### Order by优化

MySQL的排序，有两种方式：

- Using filesort : 通过表的索引或全表扫描，读取满足条件的数据行，然后在排序缓冲区sort

buffer中完成排序操作，所有不是通过索引直接返回排序结果的排序都叫 FileSort 排序。

- Using index : 通过有序索引顺序扫描直接返回有序数据，这种情况即为 using index，不需要

额外排序，操作效率高。

对于以上的两种排序方式，Using index的性能高，而Using filesort的性能低，我们在优化排序

操作时，尽量要优化为 Using index。

> 排序时,也需要满足最左前缀法则,否则也会出现 filesort。因为在创建索引的时候， age是第一个字段，phone是第二个字段，所以排序时，也就该按照这个顺序来，否则就会出现 Usingfilesort。
>
> 创建索引时，如果未指定顺序，默认都是按照升序排序的，而查询时，一个升序，一个降序，此时就会出现Using filesort。

order by优化原则:

A. 根据排序字段建立合适的索引，多字段排序时，也遵循最左前缀法则。

B. 尽量使用覆盖索引。

C. 多字段排序, 一个升序一个降序，此时需要注意联合索引在创建时的规则（ASC/DESC）。

D. 如果不可避免的出现filesort，大数据量排序时，可以适当增大排序缓冲区大小sort_buffer_size(默认256k)。 

#### group by优化

> 在分组操作中，我们需要通过以下两点进行优化，以提升性能：
>
> A. 在分组操作时，可以通过索引来提高效率。
>
> B. 分组操作时，索引的使用也是满足最左前缀法则的。

#### limit优化

在数据量比较大时，如果进行limit分页查询，在查询时，越往后，分页查询效率越低。查询排序的代价非常大。

优化思路: 一般分页查询时，通过创建 覆盖索引 能够比较好地提高性能，可以通过覆盖索引加子查询形式进行优化。

```sql
select * from tb_sku t , (select id from tb_sku order by id limit 2000000,10) a where t.id = a.id;
```

#### count优化

- MyISAM 引擎把一个表的总行数存在了磁盘上，因此执行 count(*) 的时候会直接返回这个数，效率很高； 但是如果是带条件的count，MyISAM也慢。

- InnoDB 引擎就麻烦了，它执行 count(*) 的时候，需要把数据一行一行地从引擎里面读出来，然后累积计数。

| count用法   | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| count(主键) | InnoDB 引擎会遍历整张表，把每一行的 主键id 值都取出来，返回给服务层。服务层拿到主键后，直接按行进行累加(主键不可能为null) |
| count(字段) | 没有not null 约束 : InnoDB 引擎会遍历整张表把每一行的字段值都取出来，返回给服务层，服务层判断是否为null，不为null，计数累加。有not null 约束：InnoDB 引擎会遍历整张表把每一行的字段值都取出来，返回给服务层，直接按行进行累加。 |
| count(数字) | InnoDB 引擎遍历整张表，但不取值。服务层对于返回的每一行，放一个数字“1”进去，直接按行进行累加。 |
| count(*)    | InnoDB引擎并不会把全部字段取出来，而是专门做了优化，不取值，服务层直接按行进行累加。 |

> 按照效率排序的话，count(字段) < count(主键 id) < count(1) ≈ count(\**)，所以尽量使用 count(*\*)。 

#### update优化

```sql
-- id是主键 有主键索引 设置的是行锁
update course set name = 'javaEE' where id = 1 ;

-- name字段没有索引 设置的是表锁
update course set name = 'SpringBoot' where name = 'PHP' ;
```

> 当我们开启多个事务，在执行上述的SQL时，我们发现行锁升级为了表锁。 导致该update语句的性能
>
> 大大降低。
>
> InnoDB的行锁是针对索引加的锁，不是针对记录加的锁 ,并且该索引不能失效，否则会从行锁升级为表锁 。

### 锁

MySQL中的锁，按照锁的粒度分，分为以下三类：

- 全局锁：锁定数据库中的所有表。

- 表级锁：每次操作锁住整张表。

- 行级锁：每次操作锁住对应的行数据

#### 全局锁

全局锁就是对整个数据库实例加锁，加锁后整个实例就处于只读状态，后续的DML的写语句，DDL语句，已经更新操作的事务提交语句都将被阻塞。

其典型的使用场景是做全库的逻辑备份，对所有的表进行锁定，从而获取一致性视图，保证数据的完整性。

![image-20230303152853425](/img/Database/advanced/image-20230303152853425.png)

对数据库进行进行逻辑备份之前，先对整个数据库加上全局锁，一旦加了全局锁之后，其他的DDL、DML全部都处于阻塞状态，但是可以执行DQL语句，也就是处于只读状态，而数据备份就是查询操作。

那么数据在进行逻辑备份的过程中，数据库中的数据就是不会发生变化的，这样就保证了数据的一致性和完整性。

```sql
-- 案例：备份数据库
-- 加全局锁
flush tables with read lock ;

-- 备份数据 (mysqldump不是mysql命令，要在shell中输入不是mysql输入)
-- 表名 > 路径
mysqldump -uroot –p1234 itcast > itcast.sql

-- 释放锁
unlock tables ;
```

数据库中加全局锁，是一个比较重的操作，存在以下问题：

- 如果在主库上备份，那么在备份期间都不能执行更新，业务基本上就得停摆。

- 如果在从库上备份，那么在备份期间从库不能执行主库同步过来的二进制日志（binlog），会导

致主从延迟。

在InnoDB引擎中，我们可以在备份时加上参数 \--single-transaction 参数来完成不加锁的一致性数据备份。

```sql
mysqldump --single-transaction -uroot –p123456 itcast > itcast.sql
```

#### 表级锁

##### 表锁

```sql
-- 加锁：lock tables 表名... read/write。

-- 释放锁：unlock tables / 客户端断开连接 。
```

- 表共享读锁（read lock）   

 左侧为客户端一，对指定表加了读锁，不会影响右侧客户端二的读，但是会阻塞右侧客户端的写。

![image-20230303154537138](/img/Database/advanced/image-20230303154537138.png)

- 表独占写锁（write lock）

左侧为客户端一，对指定表加了写锁，会阻塞右侧客户端的读和写。

![image-20230303154805452](/img/Database/advanced/image-20230303154805452.png)

> 结论: 读锁不会阻塞其他客户端的读，但是会阻塞写。写锁既会阻塞其他客户端的读，又会阻塞
>
> 其他客户端的写。

##### 元数据锁

meta data lock , 元数据锁，简写MDL。

MDL加锁过程是系统自动控制，无需显式使用，在访问一张表的时候会自动加上。MDL锁主要作用是维护表元数据的数据一致性，在表上有活动事务的时候，不可以对元数据进行写入操作。为了避免DML与DDL冲突，保证读写的正确性。

这里的元数据，大家可以简单理解为就是一张表的表结构。 也就是说，某一张表涉及到未提交的事务时，是不能够修改这张表的表结构的。

在MySQL5.5中引入了MDL，当对一张表进行增删改查的时候，加MDL读锁(共享)；当对表结构进行变更操作的时候，加MDL写锁(排他)。

| SQL语句                                        | 加锁类型                               | 说明                                             |
| ---------------------------------------------- | -------------------------------------- | ------------------------------------------------ |
| lock tables xxx read /write                    | SHARED_READ_ONLY /SHARED_NO_READ_WRITE |                                                  |
| select 、select ...lock in share mode          | SHARED_READ                            | 与SHARED_READ、SHARED_WRITE兼容，与EXCLUSIVE互斥 |
| insert 、update、delete、select ... for update | SHARED_WRITE                           | 与SHARED_READ、SHARED_WRITE兼容，与EXCLUSIVE互斥 |
| alter table ...                                | EXCLUSIVE                              | 与其他的MDL都互斥                                |

##### 意向锁

为了避免DML在执行时，加的行锁与表锁的冲突，在InnoDB中引入了意向锁，使得表锁不用检查每行数据是否加锁，使用意向锁来减少表锁的检查。

假如没有意向锁，客户端一对表加了行锁后，当客户端二，想对这张表加表锁时，会检查当前表是否有对应的行锁，如果没有，则添加表锁，此时就会从第一行数据，检查到最后一行数据，效率较低。

有了意向锁之后 :客户端一，在执行DML操作时，会对涉及的行加行锁，同时也会对该表加上意向锁。而其他客户端，在对这张表加表锁的时候，会根据该表上所加的意向锁来判定是否可以成功加表锁，而不用逐行判断行锁情况了。

- 意向共享锁(IS): 由语句select ... lock in share mode添加 。 与 表锁共享锁(read)兼容，与表锁排他锁(write)互斥。

- 意向排他锁(IX): 由insert、update、delete、select...for update添加 。与表锁共享锁(read)及排他锁(write)都互斥，意向锁之间不会互斥。

> 一旦事务提交了，意向共享锁、意向排他锁，都会自动释放。

#### 行级锁

行级锁，每次操作锁住对应的行数据。锁定粒度最小，发生锁冲突的概率最低，并发度最高。应用在InnoDB存储引擎中。

InnoDB的数据是基于索引组织的，行锁是通过对索引上的索引项加锁来实现的，而不是对记录加的锁。对于行级锁，主要分为以下三类：

- 行锁（Record Lock）：锁定单个行记录的锁，防止其他事务对此行进行update和delete。在RC、RR隔离级别下都支持。

![image-20230303190541295](/img/Database/advanced/image-20230303190541295.png)

- 间隙锁（Gap Lock）：锁定索引记录间隙（不含该记录），确保索引记录间隙不变，防止其他事务在这个间隙进行insert，产生幻读。在RR隔离级别下都支持。

![image-20230303190604320](/img/Database/advanced/image-20230303190604320.png)

##### 行锁

```sql
-- 查看行锁
select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from performance_schema.data_locks;
```



InnoDB实现了以下两种类型的行锁：

- 共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排它锁。

- 排他锁（X）：允许获取排他锁的事务更新数据，阻止其他事务获得相同数据集的共享锁和排他锁

常见的SQL语句，在执行时，所加的行锁如下：

| SQL                           | 锁类型     | 说明                                     |
| ----------------------------- | ---------- | ---------------------------------------- |
| INSERT ...                    | 排他锁     | 自动加锁                                 |
| UPDATE ...                    | 排他锁     | 自动加锁                                 |
| DELETE ...                    | 排他锁     | 自动加锁                                 |
| SELECT（正常）                | 不加任何锁 |                                          |
| SELECT ... LOCK IN SHARE MODE | 共享锁     | 需要手动在SELECT之后加LOCK IN SHARE MODE |
| SELECT ... FOR UPDATE         | 排他锁     | 需要手动在SELECT之后加FOR UPDATE         |

默认情况下，InnoDB在 REPEATABLE READ事务隔离级别运行，InnoDB使用 next\-key 锁进行搜索和索引扫描，以防止幻读。

- 针对唯一索引进行检索时，对已存在的记录进行等值匹配时，将会自动优化为行锁。

- InnoDB的行锁是针对于索引加的锁，不通过索引条件检索数据，那么InnoDB将对表中的所有记录加锁，此时 就会升级为表锁。

##### 间隙锁和临键锁

默认情况下，InnoDB在 REPEATABLE READ事务隔离级别运行，InnoDB使用 next\-key 锁进行搜索和索引扫描，以防止幻读。

- 索引上的等值查询(唯一索引)，给不存在的记录加锁时, 优化为间隙锁 。
- 索引上的等值查询(非唯一普通索引)，向右遍历时最后一个值不满足查询需求时，next-keylock 退化为间隙锁。

- 索引上的范围查询(唯一索引)\--会访问到不满足条件的第一个值为止。

> 注意：间隙锁唯一目的是防止其他事务插入间隙。间隙锁可以共存，一个事务采用的间隙锁不会阻止另一个事务在同一间隙上采用间隙锁。



