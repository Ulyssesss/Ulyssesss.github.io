---
title: MySQL 索引
date: 2018-12-17 21:50:51
tags:
  - MySQL
categories:
  - 技术
---

索引是对数据库表中一列或多列的值进行排序的一种结构，使用索引可以快速访问数据库表中的特定信息。索引就像书的目录，通过目录可以快速搜索到想要查找的内容。

MySQL 中索引可以分为 B+tree 索引和哈希索引，B+tree 索引又可以分为聚集索引和普通索引。





<!-- more -->

## 聚集索引

InnoDB 存储引擎表是索引组织表，聚集索引是一种索引组织形式，索引键值的逻辑顺序决定表中数据行的物理存储顺序。聚集索引叶子节点存放表中所有行的数据，索引即数据。

创建表时，要显式地创建主键（聚集索引），如果不显式创建，InnoDB 会选择第一个不包含 null 值的唯一索引作为主键。如果唯一索引也没有的话，会默认生成一个 6 字节的 rowid 作为主键。

使用自增列做主键，可以保证写入数据顺序自增，进而很大程度提高存取效率。



## 普通索引

普通索引在叶子节点不包含行的数据，只存自己本身的键值和主键的值，检索数据时通过普通索引叶子节点上的主键来获取想要的行记录。

以下为两种创建普通索引的语法及查看表中索引的语法：

```sql
alter table table_name add index index_name ( 索引字段 );
create index index_name on table_name ( 索引字段 );
show index index_name from table_name；
```



## SQL 优化

通过 explain 命令可以查看 SQL 的执行计划，如 `explain select * from t where name = 'AAA'` 。

返回结果中 type 如果为 all 则表示全表扫描，key 为索引使用情况，rows 表示 SQL 执行过程中扫描的行数（估算值）。extra 列中如果有 using filesort 或 using temporary 关键字则会影响数据库性能。另外 MySQL 5.7 后执行计划会默认添加 filtered 列，表示返回结果占需要扫描行数的百分比。



### SQL 优化思路

1. 检查表中各列的数据类型是否合理，遵循最小、最合适的原则。
2. 检查表中碎片是否整理。
3. 查看执行计划，检查索引的使用情况，没有用到索引则考虑创建。
4. 创建索引前需要依据选择性来判断字段是否适合创建索引。

> 选择性指不重复的索引值与数据表记录总数的比值，选择性越高查找时过滤掉的行就更多，查询效率也就越高。
>
> 一般经常被查询的列、经常用于表连接的列和经常用于排序分组的列可以考虑建索引。

5. 创建索引后再吃查看执行计划，对比两次结果查看效率是否提高。



## ICP、MRR 和 BKA

ICP 即 Index Condition Pushdown，是 MySQL 使用索引从表中检索数据的一种优化方式，从 MySQL 5.6 开始支持，默认开启。使用 ICP 优化时，执行计划中 extra 会显示 Using index condition。

ICP 开启后，如果 Where 条件可以使用索引，MySQL 会把过滤操作放到存储引擎层，过滤后再把满足的行从表中读取出来，以减少访问基表的次数和 Server 层访问存储引擎的次数。（5.6 前遍历索引定位到行，返回给 Server 层后再过滤。）

MRR 即 Multi-Range Read Optimization，作用是把普通索引叶子节点上找到的主键值集合先存储到 read_rnd_buffer 中（生产环境可在 4 ~ 8 MB 间调整），进行排序后在利用排序完毕的主键值集合访问表，由随机 IO 转为顺序 IO 降低开销。使用 MRR 优化时，执行计划中 extra 显示 Using MRR。

MRR 特性也是从 5.6 之后才有的，通过 optimizer_switch 参数中的两个选项控制。一个是 mrr，控制是否开启；另一个是 mrr_cost_based，开启时表示根据优化代价选择是否使用，关闭时表示强制使用。

BKA 即 Batched Key Access，是提高 join 表连接性能的算法，作用是读取被连接表时使用顺序 IO。通过 join buffer 收集第一个操作对象生成的相关列值，排序后通过 MRR 传给引擎层做索引查找。

BKA 通过 optimizer_switch 参数中 batched_key_access 选项来控制，默认关闭，开启需要保证强制 MRR。



## 唯一索引

唯一索引是约束条件的一种，不允许重复的值，但对 null 没有限制。主键只能有一个，但唯一索引可以有多个。

创建唯一索引的语法为 `alter table table_name add unique ( 索引字段 )` 。



## 覆盖索引

覆盖索引，即只需要通过索引就可以返回查询需要的数据，而不必查到索引后再到表中查询，大幅减少 IO 操作，效率极高。执行计划中 extra 会显示 Using index。

比如要通过 name 字段查询满足条件的主键 id，而 name 为普通索引字段，由于普通索引中包含主键值，所以就使用了覆盖索引。



## 前缀索引

对于 BLOB、TEXT 或很长的 VARCHAR 类型，可以为它们的前几个字符建立索引，即前缀索引。前缀索引更小，查询也会更快。缺点是不能再 order by、group by 中使用，也不能用作覆盖索引。

创建前缀索引的语法为 `alter table table_name add key(column_name(prefix_length))` 。



## 联合索引

联合索引即在表中两个或两个以上的列上创建的索引，通过索引中附加的列可以缩小检索的范围，更快地搜索数据。创建语法和创建普通索引一样，如 `create index idx_c1_c2 on t (c1, c2)` ，一般把选择性高的列放在前面。

使用联合索引，需要满足左前缀原则，查询语句可以只使用索引的一部分，但必须从最左侧开始。



## 哈希索引

哈希索引采用哈希算法，将键值转换为哈希值，只能进行等值查询，不可用于排序、模糊查询和范围查询。InnoDB 中哈希索引是自适应的，会根据使用情况自动生成，不能人为干预。



## 总结

索引有如下优点：

* 提高数据检索效率
* 提高聚合函数效率
* 提高排序效率
* 使用覆盖索引可以通过索引直接返回数据



创建索引时要避免如下场景：

* 字段选择性低，如性别、状态
* 对字段的查询很少
* 字段为 BLOB、TEXT 等大数据类型
* 字段允许为 NULL



使用不到索引的情况：

* 通过索引扫描的行数超过全表 30%，优化器会转为全表扫描
* 联合索引中第一个查询条件不是最左索引列
* 两个单列索引分别用于检索和排序，只能用到一个索引（查询最多使用一个索引，可以考虑联合索引）
* 查询字段上有索引，但使用了函数运算