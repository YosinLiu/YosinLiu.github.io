---
layout:     post
title:      "「MySQL」 MySQL基础"
subtitle:   "MySQL基础"
date:       2022-04-27 12:00:00
author:     "yosin"
header-img: "img/post-bg-html-css.jpg"
katex: true
tags:
    - MySQL
---



## MySQL基础

### 事务

事务是一组操作的集合，它是一个不可分割的工作单位，事务会把所有的操作作为一个整体一起向系统提交或撤销操作请求，即这些操作要么同时成功，要么同时失败。

#### 事务操作

```mysql
-- 事务控制的两种方式

-- 方式一：手动设置事务的提交方式
-- 查看/设置事务提交方式
SELECT @@autocommit ;
SET @@autocommit = 0 ; -- 0是手动提交；1是自动提交

-- 方式二
-- 开启事务
START TRANSACTION 或 BEGIN ;

-- 提交事务
commit
-- 回滚事务
rollback

-- 案例
-- 开启事务 start transaction 
-- 1. 查询张三余额 
select * from account where name = '张三'; 
-- 2. 张三的余额减少1000 
update account set money = money - 1000 where name = '张三'; 
-- 3. 李四的余额增加1000 
update account set money = money + 1000 where name = '李四'; 
-- 如果正常执行完毕, 则提交事务 
commit; 
-- 如果执行过程中报错, 则回滚事务 
-- rollback;
```

#### 事务的四大特性（ACID)

- 原子性（Atomicity）：事务是不可分割的最小操作单元，要么全部成功，要么全部失败。

- 一致性（Consistency）：事务完成时，必须使所有的数据都保持一致状态。

- 隔离性（Isolation）：数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环境下运行。

- 持久性（Durability）：事务一旦提交或回滚，它对数据库中的数据的改变就是永久的。

#### 并发事务问题

1. 脏读：一个事务读到另外一个事务还没有提交的数据。

![image-20230302104942263](/img/Database/basic/image-20230302104942263.png)

2. 不可重复读：一个事务先后读取同一条记录，但两次读取的数据不同，称之为不可重复读。

![image-20230302105013507](/img/Database/basic/image-20230302105013507.png)

3. 幻读：一个事务按照条件查询数据时，没有对应的数据行，但是在插入数据时，又发现这行数据已经存在，好像出现了 "幻影"。

![image-20230302105911220](/img/Database/basic/image-20230302105911220.png)

#### 事务的隔离级别

> 注意：事务隔离级别越高，数据越安全，但是性能越低。

| 隔离级别                    | 脏读 | 不可重复读 | 幻读 |
| --------------------------- | ---- | ---------- | ---- |
| Read Uncommitted            | √    | √          | √    |
| Read Committed (Oracle默认) | ×    | √          | √    |
| Repeatable Read (MySQL默认) | ×    | ×          | √    |
| Serializable                | ×    | ×          | ×    |

```sql
-- 查看事务隔离级别
SELECT @@TRANSACTION_ISOLATION;

-- 设置事务隔离级别
-- session:当前窗口 global:客户端所有窗口 
-- 读未提交、读已提交、可重复读、串行化
SET [ SESSION | GLOBAL ] TRANSACTION ISOLATION LEVEL { READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE }

-- 幻读的案例
set global transaction isolation level repeatable read
-- 并发进程一
start transaction
-- 并发进程二
start transaction
-- 并发进程一 查询操作 结果为empty set
select * from account where id = 3;
-- 并发进程二 插入操作 操作成功
insert into account (id,name,money) value (3,'雪豹',2000);
commit
-- 并发进程一 插入操作 报错主键重复
insert into account (id,name,money) value (3,'芝士雪豹',5000);
-- 并发进程一 查询操作 结果为empty set
select * from account where id = 3;
```

### 存储引擎

> InnoDB引擎与MyISAM引擎的区别 ? 
>
> ①. InnoDB引擎, 支持事务, 而MyISAM不支持。
>
> ②. InnoDB引擎, 支持行锁和表锁, 而MyISAM仅支持表锁, 不支持行锁。
>
> ③. InnoDB引擎, 支持外键, 而MyISAM是不支持的。

InnoDB: 是Mysql的默认存储引擎，支持事务、外键。如果应用对事务的完整性有比较高的要

求，在并发条件下要求数据的一致性，数据操作除了插入和查询之外，还包含很多的更新、删除操

作，那么InnoDB存储引擎是比较合适的选择。

MyISAM ： 如果应用是以读操作和插入操作为主，只有很少的更新和删除操作，并且对事务的完

整性、并发性要求不是很高，那么选择这个存储引擎是非常合适的。

MEMORY：将所有数据保存在内存中，访问速度快，通常用于临时表及缓存。MEMORY的缺陷就是

对表的大小有限制，太大的表无法缓存在内存中，而且无法保障数据的安全性。

### 索引

| 优点                                                         | 缺点                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 提高数据检索的效率，降低数据库。                             | 索引列也是要占用空间的。                                     |
| 通过索引列对数据进行排序，降低数据排序的成本，降低CPU的消耗。 | 索引大大提高了查询效率，同时却也降低更新表的速度，如对表进行INSERT、UPDATE、DELETE时，效率降低。 |

MySQL的索引是在存储引擎层实现的，不同的存储引擎有不同的索引结构，主要包含以下几种：

| 索引结构             | 描述                                                         |
| -------------------- | ------------------------------------------------------------ |
| B+Tree索引           | 最常见的索引类型，大部分引擎都支持 B+ 树索引                 |
| Hash索引             | 底层数据结构是用哈希表实现的, 只有精确匹配索引列的查询才有效, 不支持范围查询 |
| R-tree(空间索引)     | 空间索引是MyISAM引擎的一个特殊索引类型，主要用于地理空间数据类型，通常使用较少 |
| Full\-text(全文索引) | 是一种通过建立倒排索引,快速匹配文档的方式。类似于Lucene,Solr,ES |

#### B-tree

B\-Tree，B树是一种多叉路衡查找树，相对于二叉树，B树每个节点可以有多个分支，即多叉。以一颗最大度数（max\-degree）为5(5阶)的b\-tree为例，那这个B树每个节点最多存储4个key，5个指针：

![image-20230302153236022](/img/Database/basic/image-20230302153236022.png)

特点：

- 5阶的B树，每一个节点最多存储4个key，对应5个指针。

- 一旦节点存储的key数量到达5，就会裂变，中间元素向上分裂。

- 在B树中，非叶子节点和叶子节点都会存放数据。

#### B+tree

![image-20230302154217391](/img/Database/basic/image-20230302154217391.png)

我们可以看到，两部分：

- 绿色框框起来的部分，是索引部分，仅仅起到索引数据的作用，不存储数据。

- 红色框框起来的部分，是数据存储部分，在其叶子节点中要存储具体的数据。
- 叶子节点以链表的形式连接。

MySQL中B+tree的优化

MySQL索引数据结构对经典的B+Tree进行了优化。在原B+Tree的基础上，增加一个指向相邻叶子节点的链表指针，就形成了带有顺序指针的B+Tree，提高区间访问的性能，利于排序。

![image-20230302154539318](/img/Database/basic/image-20230302154539318.png)

#### Hash

哈希索引就是采用一定的hash算法，将键值换算成新的hash值，映射到对应的槽位上，然后存储在

hash表中。 如果两个(或多个)键值，映射到一个相同的槽位上，他们就产生了hash冲突（也称为hash碰撞），可以通过链表来解决。

![image-20230302160926354](/img/Database/basic/image-20230302160926354.png)

特点：

A. Hash索引只能用于对等比较(=，in)，不支持范围查询（between，>，< ，...）

B. 无法利用索引完成排序操作

C. 查询效率高，通常(不存在hash冲突的情况)只需要一次检索就可以了，效率通常要高于B+tree索 引

> 思考题： 为什么InnoDB存储引擎选择使用B+tree索引结构?
>
> A. 相对于二叉树，层级更少，搜索效率高；
>
> B. 对于B\-tree，无论是叶子节点还是非叶子节点，都会保存数据，这样导致一页中存储的键值减少，指针跟着减少，要同样保存大量数据，只能增加树的高度，导致性能降低；
>
> C. 相对Hash索引，B+tree支持范围匹配及排序操作；

#### 索引分类

在MySQL数据库，将索引的具体类型主要分为以下几类：主键索引、唯一索引、常规索引、全文索引。

| 分类     | 含义                                                 | 特点                     | 关键字   |
| -------- | ---------------------------------------------------- | ------------------------ | -------- |
| 主键索引 | 针对表中主键创建的索引                               | 默认自动创建, 只能有一个 | PRIMARY  |
| 唯一索引 | 避免同一个表中某数据列中的值重复                     | 可以有多个               | UNIQUE   |
| 常规索引 | 快速定位特定数据                                     | 可以有多个               |          |
| 全文索引 | 全文索引查找的是文本中的关键词，而不是比较索引中的值 | 可以有多个               | FULLTEXT |



在InnoDB存储引擎中，根据索引的存储形式，又可以分为以下两种：聚集索引，二级索引

| 分类     | 含义                                                       | 特点                |
| -------- | ---------------------------------------------------------- | ------------------- |
| 聚集索引 | 将数据存储与索引放到了一块，索引结构的叶子节点保存了行数据 | 必须有,而且只有一个 |
| 二级索引 | 将数据与索引分开存储，索引结构的叶子节点关联的是对应的主键 | 可以存在多个        |

聚集索引选取规则:

如果存在主键，主键索引就是聚集索引。

如果不存在主键，将使用第一个唯一（UNIQUE）索引作为聚集索引。

如果表没有主键，或没有合适的唯一索引，则InnoDB会自动生成一个rowid作为隐藏的聚集索引。

> 聚集索引中叶子节点是行的内容
>
> 二级索引中叶子节点是主键->id

![image-20230302165546953](/img/Database/basic/image-20230302165546953.png)

> 回表查询： 这种先到二级索引中查找数据，找到主键值，然后再到聚集索引中根据主键值，获取数据的方式，就称之为回表查询。
>
> 案例：select * from account where name = 'Kit';
>
> ①. 由于是根据name字段进行查询，所以先根据name='Arm'到name字段的二级索引中进行匹配查找。但是在二级索引中只能查找到 Arm 对应的主键值 10。
>
> ②. 由于查询返回的数据是*，所以此时，还需要根据主键值10，到聚集索引中查找10对应的记录，最终找到10对应的行row。 
>
> ③. 最终拿到这一行的数据，直接返回即可。



> 思考：
>
> 1.以下两条SQL语句，那个执行效率高? 为什么?
>
> A. select * from user where id = 10 ;
>
> B. select * from user where name = 'Arm' ;
>
> 备注: id为主键，name字段创建的有索引；
>
> 解答：
>
> A 语句的执行性能要高于B 语句。
>
> 因为A语句直接走聚集索引，直接返回数据。 而B语句需要先查询name字段的二级索引，然
>
> 后再查询聚集索引，也就是需要进行回表查询。
>
> 
>
> 2.InnoDB主键索引的B+tree高度为多高呢?
>
> 假设:
>
> 一
>
> 行数据大小为1k，
>
> 一
>
> 页中可以存储16行这样的数据。InnoDB的指针占用6个字节的空
>
> 间，主键即使为bigint，占用字节数为8。
>
> 高度为2：
>
> n * 8 + (n + 1) *6 = 16 *1024 , 算出n约为 1170
>
> 1171* 16 = 18736 （KB)
>
> 也就是说，如果树的高度为2，则可以存储 18000 多条记录。
>
> 高度为3：
>
> 1171 * 1171 * 16 = 21939856（KB)
>
> 也就是说，如果树的高度为3，则可以存储 2200w 左右的记录。

#### 语法

```sql
-- 创建索引
CREATE [ UNIQUE | FULLTEXT ] INDEX index_name ON table_name ( index_col_name,... ) ;

-- 查看索引
SHOW INDEX FROM table_name ;

-- 删除索引
DROP INDEX index_name ON table_name ;

--案例：name字段为姓名字段，该字段的值可能会重复，为该字段创建索引。
CREATE INDEX idx_user_name ON tb_user(name);

-- 案例：为profession、age、status创建联合索引。
CREATE INDEX idx_user_pro_age_sta ON tb_user(profession,age,status);
```

#### 最左前缀法则 及 索引失效

如果索引了多列（联合索引），要遵守最左前缀法则。

- 最左前缀法则指的是查询从索引的最左列开始，并且不跳过索引中的列。（存在就可以，不必按照顺序）

如果跳过了第一列，索引全部失效；如果跳跃某一列，索引将会部分失效(后面的字段索引失效)。

- 联合索引中，出现范围查询(>,<)，范围查询右侧的列索引失效。当范围查询使用>= 或 <= 时，索引不失效
- 不要在索引列上进行运算操作， 索引将失效。

```sql
-- 字符串截取函数 从第10位开始截取2位
explain select * from tb_user where substring(phone,10,2) = '15';
```

- 字符串类型字段使用时，不加引号，索引将失效。如上述‘15’的引号没有加，索引就会失效。

- 如果仅仅是尾部模糊匹配，索引不会失效。如果是头部模糊匹配，索引失效。

- 当or连接的条件，左右两侧字段都有索引时，索引才会生效。 如果or前的条件中的列有索引，而后面的列中没有索引，那么涉及的索引都不会被用到。

- MySQL在查询时，会评估使用索引的效率与走全表扫描的效率，如果走全表扫描更快，则放弃

  索引，走全表扫描。 因为索引是用来索引少量数据的，如果通过索引查询返回大批量的数据，则还不

  如走全表扫描来的快，此时索引就会失效。

#### 覆盖索引

1.覆盖索引是一种数据查询方式，不是索引类型
2.在索引数据结构中，通过索引值可以直接找到要查询字段的值，而不需要通过主键值回表查询，那么就叫覆盖索引
3.查询的字段被使用到的索引树全部覆盖到

![image-20230302210250311](/img/Database/basic/image-20230302210250311.png)

> 虽然是根据name字段查询，查询二级索引，但是由于查询返回在字段为 id，name，在name的二级索
>
> 引中，这两个值都是可以直接获取到的，因为覆盖索引，所以不需要回表查询，性能高。

#### 前缀索引

当字段类型为字符串（varchar，text，longtext等）时，有时候需要索引很长的字符串，这会让索引变得很大，查询时，浪费大量的磁盘IO， 影响查询效率。此时可以只将字符串的一部分前缀，建立索引，这样可以大大节约索引空间，从而提高索引效率。

```sql
-- 语法
create index idx_xxxx on table_name(column(n)) ;

--为tb_user表的email字段，建立长度为5的前缀索引。
create index idx_email_5 on tb_user(email(5));
```

![image-20230302212105970](/img/Database/basic/image-20230302212105970.png)

#### 联合索引

> 相比于单列索引，对于多个字段的查询，可以更好的避免回表查询。
>
> 在业务场景中，如果存在多个查询条件，考虑针对于查询字段建立索引时，建议建立联合索引，
>
> 而非单列索引。

![image-20230302212135834](/img/Database/basic/image-20230302212135834.png)