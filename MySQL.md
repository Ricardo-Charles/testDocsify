# 进阶学习

[参考文献](https://www.cnblogs.com/developer_chan/p/9234769.html)

# 事务
```mysql
# 开启一个事务
START TRANSACTION;
# 多条 SQL 语句
SQL1,SQL2...
## 提交事务
COMMIT;

# 事务回滚
ROLLBACK;
如果部分操作发生问题，映射到事务开启前。

```
数据库事务的四大特性: ACID

- 原子性（Atomicity） ： 一个事务里面的所有操作只能全部成功 , 如果一个事务里面有任何一个操作失败了, 那么回滚到事务最开始的地方, 重新执行这个事务
通过undo log 来实现的: 
所谓的回滚就是根据回滚日志做逆向操作，比如delete的逆向操作为insert，insert的逆向操作为delete
- 持久性（Durability）： 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。
通过redo log 重做日志实现: 比如 MySQL 实例挂了或宕机了，重启时，InnoDB存储引擎会使用redo log恢复数据，保证数据的持久性与完整性。
- 隔离性（Isolation）： 多个事务并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间是独立的；
通过(读写锁+MVCC)来实现的
- 一致性（Consistency）： 例如转账业务中，无论事务是否成功，转账者和收款人的总额应该是不变的；执行事务前后，数据库的数据保持前后一致，前后数据不会变化, 
通过原子性，持久性，隔离性来实现
# 事务的隔离级别


可能引发的问题
- 不可重复读侧重于修改，幻读侧重于新增或删除。

| 问题           | 现象                                                         |
| -------------- | ------------------------------------------------------------ |
| **脏读**       | **是指在一个事务处理过程中读取了另一个未提交的事务中的数据 , 导致两次查询结果不一致** |
| **不可重复读** | **是指在一个事务处理过程中读取了另一个事务中修改并已提交的数据, 导致两次查询结果不一致** |
| **幻读**       | 它发生在一个事务A读取了用户表的几行数据，接着另一个并发事务B插入了一些数据时。在随后的查询中，事务A就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。 |


|      | 隔离级别             | 简介 | 出现脏读 | 出现不可重复读 | 出现幻读 | 数据库默认隔离级别  |
| ---- | -------------------- | -------- | :------- | -------------- | -------- | ------------------- |
| 1    | **read uncommitted** | 允许读取尚未提交的数据变更 | 是       | 是             | 是       |                     |
| 2    | **read committed**   |  允许读取并发事务已经提交的数据 | 否       | 是             | 是       | Oracle / SQL Server |
| 3    | **repeatable read**  | 一个事务执行过程中看到的数据, 总是跟这个事务在启动时看到的数据是一致的,除非数据是被本身事务自己所修改， | 否       | 否             | 是       | MySQL               |
| 4    | **serializable**    | 串行化   | 否       | 否             | 否       |                     |

![在这里插入图片描述](https://img-blog.csdnimg.cn/abe2034a58a34c0799267bfcb2bbe766.png)

# 引擎InnoDB

- 如果使用InnoDB，默认的隔离级别是Repeatable Read。
- MySQL InnoDB 引擎通过 锁机制或者MVCC 等手段来保证事务的隔离性（ 默认支持的隔离级别是 REPEATABLE-READ ）。

## 引擎区别

- InnoDB支持事务, MyISAM 不支持
- MVCC  MyISAM 不支持，而 InnoDB 支持。

- InnoDB 支持行级锁(row-level locking)和表级锁,默认为行级锁, MyISAM 采用表级锁(table-level locking)。
- 是否支持数据库异常崩溃后的安全恢复:
	- MyISAM 不支持，而 InnoDB 支持。

# MVCC 靠的是隐藏字段和UNDO_LOG版本链和ReadView

- 隐藏字段: 事务ID, 回滚指针(指向上一个事务id)
- ReadView: 保存了四个事务ID属性
  - *trx_ids*，系统当前正在活跃的事务 ID 集合
  - *max_limit_id*，活跃的事务中最大的事务 ID
  - *min_limit_id*，活跃的事务中最小的事务 ID
  - *creator_trx_id*，创建这个 Read View 的事务 ID



在不需要锁的情况下, 靠MVCC实现 **可重复读和读已提交**, 
总结: Multi-Version Concurrency Control）多版本并发控制，是数据库控制并发访问的一种手段。



> - 是 数据库引擎（InnoDB） 层面实现的，不用加锁来处理读写冲突的手段, 提高访问性能

![在这里插入图片描述](https://img-blog.csdnimg.cn/68e55ce547f4412fba27a7ab2d2cc8f3.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/3238bd7f32aa44cda88343fecc8fe8a4.png)

- RC: 每一次执行快照读(普通的SELECT 语句) 时生成ReadView

  ![image](https://img2022.cnblogs.com/blog/1568946/202203/1568946-20220310101013879-1272313458.png)
  ![image](https://img2022.cnblogs.com/blog/1568946/202203/1568946-20220310100512137-305195547.png)

## 可重复读(RR) :仅在第一次执行快照读时生成ReadView,后续快照读复用, 从而实现可重复读

(有例外: 后面会说)
<img src="https://img-blog.csdnimg.cn/f7a6be06d4e4436ab5f2bd65e879e359.png" alt="在这里插入图片描述" style="zoom:200%;" />

## 产生幻读问题
![在这里插入图片描述](https://img-blog.csdnimg.cn/609fedc1c60948ff81339429a43e4f2f.png)

- 为何有幻读?

  在可重复读隔离级别的MVCC机制下,当两次快照读之间存在着修改和新增这种当前读,导致ReadView重新生成, 导致产生幻读, 也就是读到了本不应该存在的行记录

![在这里插入图片描述](https://img-blog.csdnimg.cn/83919e5f9b0a4806bb56d13c91e8f730.png)

## 解决幻读

- 如果连续多次快照读, 依靠MVCC机制的readView复用来解决幻读
- 如果是当两次快照读之间存在当前读, 那么使用间隙锁加行锁组成的next-key锁来解决, 锁住主键的区间



[通过间隙锁加行锁解决幻读](https://blog.csdn.net/qq_42799615/article/details/110942949#t4)

[参考](https://www.bilibili.com/video/BV1hL411479T?from=search&seid=16891136366669976326&spm_id_from=333.337.0.0)



# 索引 B+树

## 不同引擎区别

- MyISAM叶子节点只存储数据记录的地址 [参考](https://blog.51cto.com/u_15239532/2835757#_40)

![MySQL -  MySQL不同存储引擎下索引的实现_Mysql学习_03](https://s8.51cto.com/images/blog/202105/31/1ed39819917d8a4012e94374a7226de1.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)



[参考](https://www.cnblogs.com/luyucheng/p/6289714.html)

```sql
-- 标准语法
CREATE UNIQUE INDEX 索引名称 ON 表名(列名);


-- 为student表中姓名列创建一个普通索引
CREATE INDEX idx_name ON student(NAME);

-- 为student表中年龄列创建一个唯一索引
CREATE UNIQUE INDEX idx_age ON student(age);
```

索引的作用就相当于目录的作用。打个比方: 我们在查字典的时候，如果没有目录，那我们就只能一页一页的去找我们需要查的那个字，速度很慢。如果有目录了，我们只需要先去目录里查找字的位置，然后直接翻到那一页就行了。
![image](https://img-blog.csdnimg.cn/img_convert/e9f7c813866b52f0c0eb3cd8f5a4aa8e.png)

## 索引分类
- 普通索引： 最基本的索引，它没有任何限制。

- 唯一索引(UNIQUE)：索引列(这个表的这一列的值不能重复)的值必须唯一，但允许有空值。如果是组合索引，则列值的`组合`必须唯一. 为了保证数据记录的唯一性

  ![image](https://img-blog.csdnimg.cn/img_convert/436c29e4c0d09bcd649d35dca1294144.png)

- 主键索引：一种特殊的唯一索引，不允许有空值。一般在建表时同时创建主键索引。

- 组合索引：**而最左前缀原则指的是，如果查询的时候查询条件的列精确匹配组合索引的左边连续一列或几列，就会使用到索引**
	
	- 底层原理: B+树的叶子节点组成的链表递增排列 [参考](https://www.cnblogs.com/xuwc/p/14007766.html)![](https://img2020.cnblogs.com/blog/1804577/202005/1804577-20200521182659976-48843100.png)
	
	- (好处: 减少空间开销: 建一个联合索引 (col1,col2,col3)，实际相当于建了 (col1)，(col1,col2)，(col1,col2,col3) 三个索引。
	- 实现覆盖索引
	
- 外键索引：只有InnoDB引擎支持外键索引，用来保证数据的一致性、完整性和实现级联操作。

- 全文索引：快速匹配全部文档的方式。InnoDB引擎5.6版本后才支持全文索引。MEMORY引擎不支持。







## B+ 树的索引
不用B树的原因, 
- 非叶子节点不存储具体行记录, 只存储行记录的主键key, 从而同样的大小的磁盘块能存储更多的非叶子节点, 使得B+树的层数高度变低, 对于数据库来说，每进入树的一层，就要从磁盘读取一次数据, 磁盘的读取时间非常耗时, B+树减少了树的层数, 使得磁盘IO次数变少 
- 对于范围查询, B+树所有叶子节点形成有序链表，便于范围查询。B-树只能依靠繁琐的中序遍历[参考](https://zhuanlan.zhihu.com/p/167938523)

B+Tree相对于BTree区别：
  - 非叶子节点只存储键值信息。
  - 所有叶子节点之间都有一个连接指针。
  - 数据记录都存放在叶子节点中。
- 将上一节中的BTree优化，由于B+Tree的非叶子节点只存储键值信息，假设每个磁盘块能存储4个键值及指针信息，则变成B+Tree后其结构如下图所示：
![image](https://img-blog.csdnimg.cn/img_convert/c2fd4a4518b6c3549252e20d156007eb.png)


B+树只要遍历叶子节点就可以实现整棵树的遍历。
- B+有利于`范围查询`, 比如查询所有的比键值5大的数据, 那么直接去叶子节点组成的有序链表里面查询, 而且在数据库中基于范围的查询是非常频繁的，而B树不支持这样的操作或者说效率太低。

## 索引设计原则
- 对查询频次较高，且数据量比较大的表建立索引。
- 被频繁更新的字段应该慎重建立索引。
虽然索引能带来查询上的效率，但是维护索引的成本也是不小的。 如果一个字段不被经常查询，反而被经常修改，那么就更不应该在这种字段上建立索引了.
原因: 对表进行insert、update和delete。因为更新表时，不仅要保存数据，还要保存一下索引文件。

## 索引失效 使用覆盖索引解决之
- 以%开头的like查询
like “%aaa%” 不会使用索引而like “aaa%”可以使用索引。
- 使用不等于运算符（!=、<>）的时候无法使用索引会导致全表扫描（除覆盖索引外）
```sql
explain select * from user where age != 20;
explain select * from user where age <> 20;
```
- 在索引列上做加减乘除操作 : SELECT * FROM  user WHERE `age - 1 = 20`;





## 回表

聚簇索引: **聚簇索引就是按照每张表的主键构造一颗B+树，同时所有叶子节点中存放的就是整张表的所有行记录数据**

左边的是主键索引(聚集索引)，右边的是普通索引(非聚集索引)
![在这里插入图片描述](https://img-blog.csdnimg.cn/f05a8a333b44458c816371cb693d04d2.png)

那回表是什么：如果是通过非主键列索引进行查询，select所要获取的字段不能通过非主键索引获取到，需要先通过非主键索引获取到主键，然后再从主键索引树再次查询一遍，获取到所要查询的记录，这个查询的过程就是回表, 总共搜索两次索引树
聚集索引(聚簇索引)和非聚集索引(非聚簇索引)

## 索引还是慢时
[索引还是慢时,使用虚拟列, 创建了一个更紧凑的索引，来加速了查询的过程](https://www.cnblogs.com/jackyfei/p/12122767.html)
## 优化: 使用短索引
对字符串的列进行索引，如果可能应该指定一个前缀长度。例如，如果有一个char(255)的列，如果在前10个或20个字符内，多数值是惟一的，那么就不要对整个列进行索引。短索引不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。
## 索引面试题
- 每次update/insert/delete操作都会导致索引被重新更改

- 联合索引(a,b,c)，
where a=x and c=x and b=x会走索引, where里面的条件顺序在查询之前会被mysql自动优化 成 abc的顺序
![在这里插入图片描述](https://img-blog.csdnimg.cn/1d6e0ce5dbd245afb3c068cc98770b16.png)
(6)    select * from myTest  where a>4 and b=7 and c=9;
a用到了  b没有使用，c没有使用

# 联结（以列为单位对表进行联结）JOIN
![](https://img-blog.csdnimg.cn/img_convert/1c679ad8c75e98f1dda702bc76a219a0.png)

## 图解SQL

MySQL全连接 : UNION 配合 LEFT和 RIGHT JOIN使用

```mysql
SELECT * FROM t1
LEFT JOIN t2 ON t1.id = t2.id
UNION
SELECT * FROM t1
RIGHT JOIN t2 ON t1.id = t2.id
```

uninon: 将不同表中相同列中查询的数据展示出来；（不包括重复数据）

![image](https://img2022.cnblogs.com/blog/1568946/202203/1568946-20220314163050300-611113078.png)

union all: 包含重复数据

![image](https://img2022.cnblogs.com/blog/1568946/202203/1568946-20220314163327933-1281524395.png)

![image](https://img-blog.csdnimg.cn/img_convert/fa77dca8fbb554eb019643fccfa9ac15.png)

- 图解 GROUP BY
![在这里插入图片描述](https://img-blog.csdnimg.cn/5bef656eb5bc473db940ea70a87b9ee6.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/bd2a2e2e37d247aca916f65584dae6ca.png)





# 锁

- 乐观锁: 假设即使在并发环境中, 对数据的的操作不会造成冲突, 所以不会去加锁, 但是在更新完数据提交事务的时候, 回去检测是否有冲突, 一般使用版本号检测, 乐观锁用来在不加锁的情况下解决多个事务之间更新数据的冲突, 适合于读操作多, 写操作少的情况. 

  乐观锁的代表是MVCC多版本并发控制

- 按操作分类：
  - 共享锁：也叫读锁。针对同一份数据，多个事务读取操作可以同时加锁而不互相影响 ，但是不能修改数据记录。
```sql
  -- 标准语法
SELECT语句 LOCK IN SHARE MODE;
```

  - 排他锁：也叫写锁。当前的操作没有完成前,会阻断其他操作比如读取和写入 

```sql
  SELECT语句 FOR UPDATE;
```


- 按粒度分类：
  - 表级锁：操作时，会锁定整个表。开销小，加锁快；不会出现死锁；锁定力度大，发生锁冲突概率高，并发度最低。偏向于MyISAM存储引擎！
  - 行级锁：给带有索引的行记录加锁
    如果更新一个字段,然后在筛选的时候where后面的字段没有加索引,就会把整个表的所有记录都锁定.  行记录是针对索引加的锁，不是针对行记录加的锁。并且该索引不能失效，否则都会从行锁升级为表锁
    在where后面的筛选字段加上索引后就不会锁表只会锁行,即使这个筛选范围字段只是组合索引里的其中一个![在这里插入图片描述](https://img-blog.csdnimg.cn/197d0dedf85f4a2f9014ba61f82efd69.png)
  - 记录锁: 锁住索引的行记录
  - 间隙锁(GAP) 为了阻止多个事务将记录插入到同一开区间内(比如ID的1到5这个开区间)注意是开区间，避免幻读问题的产生. ![在这里插入图片描述](https://img-blog.csdnimg.cn/d29c7d446fd242d391883060cdadd1af.png)
  - 临键锁（Next-key Lock）== 记录锁 + 间隙锁, InnoDB对于行的查询采用的就是这种锁定算法
  	![在这里插入图片描述](https://img-blog.csdnimg.cn/3adafa7178414856b67e0bf595c274e7.png)
  	[参考](https://blog.csdn.net/zhangchaoyang/article/details/108809249)

# sql常见优化

-  尽量使用 TIMESTAMP 而非  DATETIME
- 避免使用NULL字段
- 不用 SELECT *
- 对于连续数值，使用 BETWEEN 不用  IN ： 
`SELECT id FROM t WHERE num BETWEEN 1 AND 5`
- 单表不要有太多的字段(列)，建议在20以内 
- 给经常被访问的字段创建索引，也可以创建联合索引，但是遵从最左原则。

            例：create index '索引名'  on '表名'( 字段名 (长度 ) )
	    
                   create index index_name on table(column(length))
	
- 避免使用子查询, 使用多表连接查询JOIN等
子查询是将一个查询语句嵌套在另一个查询语句中: 比如
通常情况下子查询都与 SELECT 语句一起使用，其基本语法如下所示：
![image](https://img-blog.csdnimg.cn/img_convert/8c9370decbc937537f7015ff19b20ec3.png)
- 批量插入代替循环单条插入
- 把一个大表拆成两个一对一的表，目的是把经常读取和不经常读取的字段分开，以获得更高的性能。
例如，把一个大的用户表分拆为用户基本信息表user_info和用户详细信息表user_profiles，大部分时候，只需要查询user_info表，并不需要查询user_profiles表，这样就提高了查询速度。
- 查询出来的数据量很大时, 使用分页查询 LIMIT
## 查询优化

![在这里插入图片描述](https://img-blog.csdnimg.cn/5416f84ac5554582bd436118ba51d57e.png)
- 不在索引列进行数学运算和函数运算(会导致无法使用索引从而导致全表扫描)，如WHERE id+1 = 100 和WHERE  id = 100 - 1，效率差很远
- 小表驱动大表: 比如 IN和EXIST, EXIST是先查外层SELECT的表  [参考](https://www.cnblogs.com/xuwc/p/14059032.html)
- ![在这里插入图片描述](https://img-blog.csdnimg.cn/da16faacf23a4466a4fc2b42b64d087a.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/64f20374dcb9480ba96044796e987360.png)

### 查询尽可能使用 limit 减少返回的行数，减少数据传输时间和带宽浪费。

### 任何地方都不要使用select * from t ，用具体的字段列表代替 * ，不要返回用不到的任何字段。
**EXIST语法介绍**  [参考](https://malaoshi.top/show_1IX1hsc00ErT.html)

![image](https://img2022.cnblogs.com/blog/1568946/202205/1568946-20220516083309719-503457460.png)

## 大字段去进行压缩

使用mysql的 COMPRESS函数来压缩字段 [参考](https://blog.csdn.net/fenglllle/article/details/103865600)

![image](https://img2022.cnblogs.com/blog/1568946/202203/1568946-20220311101324831-533390556.png)

# 数据库设计三范式

[参考](https://greenhathg.github.io/2020/02/08/%E9%80%9A%E4%BF%97%E7%90%86%E8%A7%A3%E6%95%B0%E6%8D%AE%E5%BA%93%E4%B8%89%E5%A4%A7%E8%8C%83%E5%BC%8F/)

- 第一范式是指数据库表的每一列都是不可分割的基本数据项

- 第二范式是指在第一范式的基础上, 并且要求每个不是主键的字段列需要完全依赖于主键，而不是仅依赖于主键字段的一部分。

	然后开始举例子, 当主键是两个字段组合而成的比如(id, course)
	
	作用: 让一张表只做一件事情, 不要把所有的字段列放到一张表里面, 应该根据业务需求分成多个表 [看里面的参考例子](https://zh.wikipedia.org/wiki/%E7%AC%AC%E4%BA%8C%E6%AD%A3%E8%A6%8F%E5%8C%96)
	
- ![在这里插入图片描述](https://img-blog.csdnimg.cn/0194e43d6f6d42d697e44a15253a938a.png)

  - 不满足第二范式后的问题: 
  	- 插入异常: 
  	如计划开新课，由于没人选修，没有课程号信息，那么因为（学号，课程号）是主键，所以连学生基本信息也不能插入，只有学生选课之后，才能插入信息。
  	- 删除异常
  	若学生申请病假休学一学期，从当前数据库删除`选修记录(也就是包括了课程号)`，那么学生的基本信息也就丢了。

  # 第三范式要求非主键字段之间不能有依赖关系, 非主键字段相互独立

  [看里面的参考例子](https://zh.wikipedia.org/wiki/%E7%AC%AC%E4%B8%89%E6%AD%A3%E8%A6%8F%E5%8C%96)

# sql常用语句

![在这里插入图片描述](https://img-blog.csdnimg.cn/b12f2e0a97b94f94a2401e5d5a1c6241.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/92a2e5a829da4b47b0a43321f1ac2454.png)
## 一个普通的表，如何把相同的两行数据查询出来？
- 子查询先执行 IN 后面的语句, 再执行外层的SELECT
![在这里插入图片描述](https://img-blog.csdnimg.cn/c58f755e3e7e4b1eb02359476379530a.png)
- 下面只可以查出哪一行是重复的
![在这里插入图片描述](https://img-blog.csdnimg.cn/9615594c7cb54384b3fc93349d80e829.png)
- 删除重复行![在这里插入图片描述](https://img-blog.csdnimg.cn/cb423c5a4aaa41258f07c6d8954dfc21.png)
- 不等于运算符  <>
- **CHAR**数据类型是MySQL中固定长度的字符类型。 
- 我们经常声明CHAR类型，其长度指定要存储的最大字符数。 例如，CHAR(20)最多可以容纳20个字符。
如果要存储的数据是固定大小，则应使用CHAR数据类型。在这种情况下，与VARCHAR相比，您将获得更好的性能。
CHAR数据类型的长度可以是从0到255的任何值。
- 字符串类型大小排序：字典顺序![在这里插入图片描述](https://img-blog.csdnimg.cn/7c739a4d75c544a49a317a6186761040.png)
## HAVING 和 WHERE 区别

- 相同点

![在这里插入图片描述](https://img-blog.csdnimg.cn/9a94b156d3604b7cb862f7c38866de4e.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/8e8c02d627b147d6b4fe1781e6de0006.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/aa2243124db649a3b16bb448f044c0ff.png)
- 不同点：HAVING 执行顺序在GROUP BY 之后,作用于分组后的数据, WHERE 在GROUP BY之前

![在这里插入图片描述](https://img-blog.csdnimg.cn/2e676d358fdc4cce8d202f6b3d6c0a68.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/5ad9079e153a4c66aecfb9f05557d606.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/cab20e4b041147299bee178986b12568.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/5e8f4eeb6e6b4adf990e86adccb91aef.png)

## 视图就是 SELECT 语句
应该将经常使用的SELECT语句做成视图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/60bffd3a8969488bb0394282d68f6f27.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2e35136f633a4a3d84de176016c450ee.png)

## 子查询使用例子

- 例子1:

```sql
/* 题目: 查询出工资比 Abel 高的行记录
    分析1： 求出Abel的工资
    分析2： 筛选employees表，判断 salary > Abel的工资
*/
SELECT * 
FROM employees 
WHERE salary>(
    SELECT salary 
    FROM employees 
    WHERE last_name = 'Abel'
);

```
- 例子2: 

```sql
题目: 查询job_id与141号员工相同，salary比143号员工多 的员工行记录
    分析：先查询 141 号员工的job_id
           再查询 143号员工的salary
SELECT job_id 
FROM employees 
WHERE employee_id = 141

SELECT salary 
FROM employees 
WHERE employee_id = 143
2》结合以上两步将其作为被比较的条件对象筛选指定的员工

# 最终方法如下:
SELECT *
FROM employees
WHERE job_id =(
    SELECT job_id 
    FROM employees 
    WHERE employee_id = 141

) AND salary > (
    SELECT salary 
    FROM employees 
    WHERE employee_id = 143
)

```
### 重点: 返回工资最少的员工的last_name,job_id和salary

```sql

SELECT last_name,job_id,salary 
FROM employees
WHERE salary=(

    //把如下语句获取的值作为salary比较的条件对象
    SELECT MIN(salary) FROM employees
);


```
## UPDATE入门

```sql
UPDATE table_name SET field1=new-value1, field2=new-value2
[WHERE Clause]
```

# 杂项知识点

- TEXT类型, 文本字符串, 最大4GB, 用于存储产品详细描述

 - 慢查询日志: 用来记录在 MySQL 中执行时间超过指定时间的 SQL 语句。通过慢查询日志，可以查找出哪些语句的执行效率低，以便进行优化。通过explain命令分析sql的执行计划并进行相应的优化

- CASE表达式

  ```sql
  CASE value
  WHEN compare_value_1 THEN result_1
  WHEN compare_value_2 THEN result_2
  …
  ELSE result_3 
  END
  ```

  如果value等于compare_value_1，则CASE表达式返回result_1 

  如果值不与任何compare_value匹配，则CASE表达式将返回ELSE后面的 result_3
  CASE表达式的第二种形式如下：

```sql
CASE
WHEN condition_1 THEN result_1
WHEN condition_2 THEN result_2
ELSE result 
END
```

在第二种形式中，如果condition_1结果为True，则CASE表达式返回结果result_1，如果所有条件都为false，则返回ELSE部分中的结果。如果省略ELSE部分，CASE表达式将返回NULL。
例子:

```sql
假设您要按状态对客户进行排序，如果state为NULL，则要使用国家country作为排序标准。要实现这一点，您可以使用第一种形式的CASE表达式如下：
SELECT 
    customerName, state, country
FROM
    customers
ORDER BY
(CASE
    WHEN state IS NULL THEN country
    ELSE state
END);

```
## delete，drop，truncate 都有删除表的作用
- delete和truncate 只删除表的数据, drop表数据和表结构都删除, 连表都不存在了
- delete可以回滚, 其他不能
- 执行速度drop>truncate>delete
## 分页查询
![在这里插入图片描述](https://img-blog.csdnimg.cn/b5ba58397bb14c949e0b24034e879103.png)
SELECT * FROM table LIMIT 5,10表示从第6行开始返回、一共返回10行。
![在这里插入图片描述](https://img-blog.csdnimg.cn/089fb07d24204d10b0bfb3e1e0596100.png)

- 实现分页查询的例子:
每页显示pageSize条记录, 然后实现查询第n页的前三行数据
LIMIT  (n-1) * pageSize , 3
解释为啥要(n-1), 因为当查询第一页时, 肯定(n-1)
## 唯一性约束 和 主键 和 唯一索引(三者作用相同)
- UNIQUE KEY出现的意义:
每个表只有一个主键。 如果要让多个列具有唯一值, 或者多个列联合起来作为唯一值, 则不能使用主键约束。
请注意，每个表可以有多个 UNIQUE 约束，但是每个表只能有一个 PRIMARY KEY 约束。
UNIQUE 约束唯一标识数据库表中的每条记录。

UNIQUE 和 PRIMARY KEY 约束均为列或列集合提供了唯一性的保证。

PRIMARY KEY 约束拥有自动定义的 UNIQUE 约束。
![在这里插入图片描述](https://img-blog.csdnimg.cn/28d574482feb4168880ffd71f771077e.png)
[参考](https://www.yiibai.com/mysql/unique.html)

### 多列联合唯一约束
如果业务中要求两个字段联合起来才是唯一的, 
![在这里插入图片描述](https://img-blog.csdnimg.cn/83cf86c7a1294e4aa13d907725255d9f.png)
## 分库分表
- 分库
  将每个服务相关的表拆出来单独建立一个数据库，这其实就是“分库”了。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/354f13851e0f46b190b4967d20dcbb83.png)
  [参考](https://mp.weixin.qq.com/s/RlOezSf9bLiMAaevhwT5Zg)

- 分表: 水平拆分和垂直拆分

## 死锁

- 死锁是指两个或两个以上的事务在执行过程中，因争夺资源而造成的一种互相等待的现象。
	> 死锁案例: 开启事务A, 更新id=1的行记录, 开启事务B, 更新id=2的行记录, 然后在事务A里面再去更新id=2的行记录, 会发现卡住了, 去事务B里面更新id=1的记录, 然后mysql报错信息提示出现死锁

- 解决死锁: 一种是超时回滚，一种是采用死锁检测机制（wait-for graph等待图）

## 碰到错误

对于Mysql，数据是持久化在磁盘上的。

如果误删数据，可以使用binlog进行恢复;

突然宕机时，其本身可以借助redo log进行崩溃恢复。

## 精确存储金额

```sql
amount DECIMAL(6,2); # 总长为6位, 小数点就占了2位
```

在此示例中，`amount`列最多可以存储`6`位数字，这个6位也包括了小数点的位数, 小数位数为`2`位; 因此，`amount`列的范围是从`-9999.99`到`9999.99`。

##  外键

如果一个字段X在一张表（表一）中是主关键字，而在另外一张表（表二）中X不是主关键字，则字段X称为表二的外键；

外键用来和其他表建立联系用, 

 以学生和成绩的关系为例，学生表中的 student_id 是主键，那么成绩表中的 student_id 则为外键。如果更新学生表中的 student_id，同时触发成绩表中的 student_id 更新，即为级联更新。[参考](http://www.mytecdb.com/blogDetail.php?id=111)



外键与级联更新适用于单机低并发，不适合分布式、高并发集群;





# 复杂查询

需要用的表的信息如下：
1.学生表：student（编号sid，姓名sname，年龄sage，性别ssex)
2.课程表：course（课程编号cid，课程名称cname，教师编号tid）
3.成绩表：sc（学生编号sid，课程编号cid，成绩score）
4.教师表：teacher（教师编号tid，姓名tname）

## 查询课程名称为“数据库”，且分数低于60的学生姓名和分数



```sql
select sname,score
    from student,sc,course
    where  sc.sid=student.sid 
    and  sc.cid=course.cid 
    and  course.cname='数据库'
    and  score<60; 

```

[参考](https://blog.csdn.net/qq_43278189/article/details/120110256)

## 查询平均成绩大于等于 85 的所有学生的学号、姓名和平均成绩

![在这里插入图片描述](https://img-blog.csdnimg.cn/ac61ea2ec9e04f868963b3a23cbecd40.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/a9d0aab5e79248f7a8b6ff8091dcf105.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/5ab62bcce27e450ea195469ce21e6994.png)

GROUP BY 学生的id 这个最重要, 

[参考](https://zhuanlan.zhihu.com/p/67645448)



## 可以不看_查询平均成绩大于等于 60 分的同学的学生编号和学生姓名和平均成绩

```sql
select a.SId,b.Sname,avgs 
from(select SId,
AVG(score) as avgs from sc 
group by SId having AVG(score)>=60) a
left join student b
on a.SId=b.SId
```

![](https://pic1.zhimg.com/80/v2-0e2499f01248614fdf60e820d6ed2380_1440w.jpg)

## 查询各科前三名

[不使用窗口函数, 原生SQL](https://blog.csdn.net/weixin_44497013/article/details/107317719)

# 踩坑

要设置时区(time zone) , 要不然会有奇怪的错误

```sql
url: jdbc:mysql://localhost:3306/lagou?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
```
