---
title: 'mysql两万字精华总结'
date: 2021-08-09 16:11:00
categories: 
  - 数据库
tags: 
  - little_eight
---
## 基础架构


&ensp;&ensp;&ensp;&ensp;这是官网的一个架构图，总体是可以分为四层的。
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/1.png)

- **连接层**：最上层是一些客户端和连接服务。**主要完成一些类似于连接处理、授权认证、及相关的安全方案**。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于SSL的安全链接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。
- **服务层**：第二层服务层，主要完成大部分的核心服务功能， 包括查询解析、分析、优化、缓存、以及所有的内置函数，所有跨存储引擎的功能也都在这一层实现，包括触发器、存储过程、视图等
- **引擎层**：第三层存储引擎层，存储引擎真正的负责了MySQL中数据的存储和提取，服务器通过API与存储引擎进行通信。不同的存储引擎具有的功能不同，这样我们可以根据自己的实际需要进行选取
- **存储层**：第四层为数据存储层，主要是将数据存储在运行于该设备的文件系统之上，并完成与存储引擎的交互
<!--more--> 
​

&ensp;&ensp;&ensp;&ensp;可以从sql的执行过程分析总体的架构，比如下图：
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/2.png)



### 连接器
&ensp;&ensp;&ensp;&ensp;客户端请求连接数据库时，连接器就会负责跟客户端建立连接、获取权限、维持和管理连接。MySQL服务器端也会有一个连接池，因为一般都会有多个系统与MySQL建立很多个连接，MySQL通过这个连接池去维护与客户端的数据库连接。
&ensp;&ensp;&ensp;&ensp;除此之外，连接器还会根据请求的账号和密码，进行安全认证，库表权限认证。如果用户名或密码不对，就会收到一个"Access denied for user"的错误。如果用户名密码认证通过，连接器会到权限表里面查出你拥有的权限。之后，这个连接里面的权限判断逻辑，都将依赖于此时读到的权限。这也意味着，一个用户成功建立连接后，即使你用管理员账号对这个用户的权限做了修改，也不会影响已经存在连接的权限。修改完成后，只有再新建的连接才会使用新的权限设置。
&ensp;&ensp;&ensp;&ensp;连接完成后，如果没有后续的动作，这个连接就处于空闲状态。客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数 wait_timeout 控制的，默认值是 8 小时。
&ensp;&ensp;&ensp;&ensp;对于一个 MySQL连接，任何时刻都有一个状态，该状态表示了MySQL当前正在做什么。可以使用 **SHOW FULL PROCESSLIST **命令查看当前连接的状态（Command列）。在一个查询的生命周期中，状态会变化很多次。
&ensp;&ensp;&ensp;&ensp;MySQL官方手册中给出了如下状态：

- **Sleep**：处于空闲状态，线程正在等待客户端发送新的请求。
- **Query**：线程正在执行查询或者正在将结果发送给客户端。
- **Locked**：在MySQL服务器层，该线程正在等待表锁。在存储引擎级别实现的锁，例如InnoDB的行锁，并不会体现在线程状态中。对于MyISAM来说这是一个比较典型的状态。
- **Analyzing and statistics**：线程正在收集存储引擎的统计信息，并生成查询的执行计划。
- **Copying to tmp table [on disk]**：线程正在执行查询，并且将其结果集都复制到一个临时表中，这种状态一般要么是在做 GROUP BY 操作，要么是文件排序操作，或者是 UNION 操作。如果这个状态后面还有“on disk”标记，那表示MySQL正在将一个内存临时表放到磁盘上。
- **Sorting result**：线程正在对结果集进行排序。
- **Sending data**：线程可能在多个状态之间传送数据，或者在生成结果集，或者在向客户端返回数据。

​

### 缓存
&ensp;&ensp;&ensp;&ensp;在解析一个查询语句之前，如果查询缓存是打开的，那么MySQL会优先检查这个查询是否命中查询缓存中的数据。之前执行过的语句及其结果可能会以 key-value 对的形式，被直接缓存在内存中。key 是查询的语句，value 是查询的结果。查询和缓存中的查询即使只有一个字节不同，那也不会匹配缓存结果，这种情况下查询就会进入下一阶段的处理。
&ensp;&ensp;&ensp;&ensp;如果当前的查询恰好命中了查询缓存，那么在返回查询结果之前MySQL会检查一次用户权限。这仍然是无须解析查询SQL语句的，因为在查询缓存中已经存放了当前查询需要访问的表信息。如果权限没有问题，MySQL会跳过所有其他阶段，直接从缓存中拿到结果并返回给客户端。这种情况下，查询不会被解析，不用生成执行计划，不会被执行，这个效率会很高。
&ensp;&ensp;&ensp;&ensp;但大多数情况下建议不要使用查询缓存，因为查询缓存往往弊大于利。查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。因此很可能缓存起来的结果还没使用，就被一个更新全清空了。对于更新压力大的数据库来说，查询缓存的命中率会非常低。
&ensp;&ensp;&ensp;&ensp;需要注意的是，**MySQL 8.0 版本直接将查询缓存的整块功能删掉了**，也就是说 8.0 开始彻底没有这个功能了。
### 查询解析器
&ensp;&ensp;&ensp;&ensp;解析器来拆解这个SQL，生成一棵对应的**解析树**，将其变成MySQL能理解的东西。
&ensp;&ensp;&ensp;&ensp;首先会进行词法分析，SQL语句是由多个字符串和空格组成的，MySQL 需要识别出里面的字符串分别是什么，代表什么。接着进行语法分析，根据词法分析的结果，语法分析器会根据语法规则，判断你输入的这SQL语句是否满足 MySQL 语法。
&ensp;&ensp;&ensp;&ensp;例如，它将验证是否使用错误的关键字，使用关键字的顺序是否正确，引号是否能前后正确匹配等。如果你的语句不对，就会收到“You have an error in your SQL syntax”的错误提醒。
&ensp;&ensp;&ensp;&ensp;预处理器则根据一些MySQL规则进一步检查解析树是否合法，这里将检查数据表和数据列是否存在，解析名字和别名，看看它们是否有歧义。下一步预处理器会验证是否有表权限、字段权限。
​

### 查询优化器
&ensp;&ensp;&ensp;&ensp;一条查询可以有很多种执行方式，最后都返回相同的结果。优化器的作用就是找到这其中最好的执行计划。MySQL使用基于成本的优化器，它将尝试预测一个查询使用某种执行计划时的成本，并选择其中成本最小的一个。
#### 查询成本
&ensp;&ensp;&ensp;&ensp;无论是执行单表查询，还是多表关联查询，都有多种执行计划可以选择，比如有的表可以全表扫描，也可以用索引A，也可以用索引B，那么到底是选择哪个执行计划呢?
&ensp;&ensp;&ensp;&ensp;MySQL使用基于成本的优化器，它将尝试预测一个查询使用某种执行计划时的成本，并选择其中成本最小的一个。
&ensp;&ensp;&ensp;&ensp;一条SQL查询语句的成本主要有两块：
**I/O成本**
&ensp;&ensp;&ensp;&ensp;首先数据都是存储在磁盘文件中的，需要先把数据从磁盘读到内存中才能操作。这个从磁盘读数据到内存所损耗的时间就是I/O成本。对于InnoDB来说，页是磁盘和内存之间交互的最小单位，MySQL 约定读一页的IO成本是1.0。
**CPU成本**
&ensp;&ensp;&ensp;&ensp;拿到数据之后，接着就会对数据做一些运算，比如验证是否满足条件，做一些分组排序的事情，这些都是耗费CPU资源，属于CPU成本。MySQL 约定读取和检测一条数据是否符合条件的CPU成本是0.2。
&ensp;&ensp;&ensp;&ensp;这个所谓 1.0 和 0.2 就是MySQL自定义的一个成本值，也称为成本常数，代表的意思就是一个数据页IO成本就是 1.0，一条数据检测的CPU成本就是 0.2。
&ensp;&ensp;&ensp;&ensp;举个例子，一条sql查出的表的记录数为1000，它的索引的字节数大小为64436B，则可算出数据页为64436/1024/16=4个（数据页默认为16k），所以全表扫描io的成本为：4*1=4，cpu成本为1000*2=2000，总成本为2004。
&ensp;&ensp;&ensp;&ensp;在计算出全表扫描、使用各个索引查询的成本之后，就会对比各个执行计划的成本，然后找出成本最低的一个执行计划。经过计算，全表扫描的成本显示是最大的，使用索引的成本最低。
​

## 基础数据类型


- 整数类型：BIT、BOOL、**TINYINT**、SMALLINT、MEDIUMINT、 **INT**、 **BIGINT**
- 浮点数类型：FLOAT、DOUBLE、DECIMAL
- 字符串类型：CHAR、**VARCHAR**、TINY TEXT、TEXT、MEDIUM TEXT、LONGTEXT、TINY BLOB、BLOB、MEDIUM BLOB、LONG BLOB
- 日期类型：**Date**、**DateTime**、TimeStamp、Time、Year
- 其他数据类型：BINARY、VARBINARY、ENUM、SET、Geometry、Point、MultiPoint、LineString、MultiLineString、Polygon、GeometryCollection等

​

![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/3.png)


![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/add1.png)


![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/4.png)


**字段类型后面括号的数字作用**：

- Char跟varchar，表示字符的最长长度
- 数据类型，表示显示数据的最小长度，但需要指定某个字符才能填充，比如zerofill（拿0来填充）

​

​

## 存储引擎
&ensp;&ensp;&ensp;&ensp;存储引擎是MySQL的组件，用于处理不同表类型的SQL操作。不同的存储引擎提供不同的存储机制、索引技巧、锁定水平等功能，使用不同的存储引擎，还可以获得特定的功能。
&ensp;&ensp;&ensp;&ensp;使用哪一种引擎可以灵活选择，**一个数据库中多个表可以使用不同引擎以满足各种性能和实际需求**，使用合适的存储引擎，将会提高整个数据库的性能 。
&ensp;&ensp;&ensp;&ensp;MySQL服务器使用**可插拔**的存储引擎体系结构，可以从运行中的 MySQL 服务器加载或卸载存储引擎 。
&ensp;&ensp;&ensp;&ensp;常见的存储引擎就 MyISAM、InnoDB。
​

### MyISAM
&ensp;&ensp;&ensp;&ensp;在mysql5.5版之前MyISAM是MySQL的默认数据库引擎，由早期的ISAM所改良。虽然性能极佳，但却有一个缺点：不支持事务处理。
&ensp;&ensp;&ensp;&ensp;每个MyISAM表，皆由存储在硬盘上的3个文件所组成，每个文件都以表名称为文件主名，并搭配不同扩展名区分文件类型：
.frm－－存储资料表定义，此文件非MyISAM引擎的一部分。.
.MYD－－存放真正的数据。.
.MYI－－存储索引信息。
### InnoDB
&ensp;&ensp;&ensp;&ensp;InnoDB 现在是 MySQL 默认的存储引擎，支持**事务、行级锁定和外键。**其物理文件结构为**：**

- .frm ：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等。
- .ibd 或 .ibdata ： 这两种文件都是存放 InnoDB 数据的文件，之所以有两种文件形式存放 InnoDB 的数据，是因为 InnoDB 的数据存储方式能够通过配置来决定是使用**共享表空间**存放存储数据，还是用**独享表空间**存放存储数据。

​

&ensp;&ensp;&ensp;&ensp;独享表空间存储方式使用.ibd文件，并且每个表一个.ibd文件 共享表空间存储方式使用.ibdata文件，所有表共同使用一个.ibdata文件（或多个，可自己配置。
​

#### 事务
&ensp;&ensp;&ensp;&ensp;事务的acid：

- **atomicity** 原子性，一个事务要么全部提交成功要么全部失败回滚。
- **consistency**一致性，数据库总是从一个一致性的状态转换到另一个一致性的状态（成功全部提交，失败全部回滚）。
- **isolation**隔离性，一个事务在提交时对其他事务是不可见的。
- **durability**持久性，一旦事务提交，其修改就会保存到数据库。



&ensp;&ensp;&ensp;&ensp;每一种特性，innodb都有自己的方式去实现，具体如下图：
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/5.png)


#### 隔离级别

- 未提交读read uncommitted，事务可读取未提交的数据，可出现脏读、不可重复读、幻读
- 提交读read committed，事务可读取已提交的数据，可出现不可重复读、幻读
- 可重复读repeatable read，同个事务多次读取同样的记录结果是一致的，可能出现幻读
- 可串行化serializable，会在读取每一行数据时加上锁

​

**为什么默认RR级别？**
1、越高的隔离级别，能解决的数据一致性问题越多，理论上性能损耗更大，可并发性越低。
2、在mysql5.0版本之前，binlog在读已提交这个隔离级别的主从复制是有bug的，因此把可重复读作为默认3、原因其实很简单，就是主机的执行顺序是先删除和插入！此时，binlog 是 STATEMENT 格式的，其记录的顺序首先插入和删除！同步来自(从属)是 binlog，所以从属的执行顺序与主机不一致！主人和奴隶之间会有矛盾！如何解决？有两个解决方案！
(1)将隔离级别设置为可重复读，并在此隔离级别引入间隙锁。当会话1执行 delete 语句时，间隙被锁定。然后，Ssession 2将阻止 insert 语句的执行！
(2)将 binglog 格式改为行格式。在这个时候，它是一个基于行的副本。当然，sql 的执行顺序不会有问题！这种格式是在 mysql5.1版本开始时引入的。因此，由于历史原因，mysql 将默认隔离级别设置为 Repeatable Read (Repeatable Read) ，以确保主从复制不会导致问题！


#### 锁的分类
&ensp;&ensp;&ensp;&ensp;**innodb基于索引来实现行级锁的，条件里无索引的话则为表锁。**
&ensp;&ensp;&ensp;&ensp;innodb实现了如下两种标准的行级锁：

- **共享锁**(读锁 S Lock)，又称读锁，若事务T对数据对象A加上S锁，则事务T可以读A但是不能修改，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S锁。这保证了其他事务可以读A，不可以修改A。
- **排它锁**(写锁 X Lock)，又称写锁，若事务T对数据对象A加上X锁，则事务T可以读A也可以修改，其他事务不能对A进行加锁，直到T释放A上的X锁。

&ensp;&ensp;&ensp;&ensp;举几个例子说明：

- SELECT ... 语句正常情况下为快照读，不加锁；（快照读的意思是，数据有多个版本，当事务并发执行时，某一事务读取的数据来	自其中一个版本（快照））
- SELECT ... LOCK IN SHARE MODE：加共享(S)锁
- SELECT ... FOR UPDATE：加排他(X)锁
- INSERT / UPDATE / DELETE：加排他(X)锁

​

&ensp;&ensp;&ensp;&ensp;InnoDB 支持两种**意向锁**(即为表级别的锁)：

- **意向共享锁**(读锁 IS Lock)，事务想要获取一张表的几行数据的共享锁，事务在给一个数据行加共享锁前必须先取得该表的 IS 锁。
- **意向排他锁**(写锁 IX Lock)，事务想要获取一张表中几行数据的排它锁，事务在给一个数据行加排它锁前必须先取得该表的 IX 锁。

​

&ensp;&ensp;&ensp;&ensp;加意向锁为了表明某个事务正在锁定一行或者将要锁定一行数据。首先申请意向锁的动作是InnoDB 完成的，怎么理解意向锁呢?例如：事务 A 要对一行记录 R 进行上 X 锁，那么 InnoDB 会先申请表的 IX 锁，再锁定记录 R 的 X 锁。在事务 A 完成之前，事务 B 想要来个全表操作，此时直接在表级别的 IX 就告诉事务 B 需要等待而不需要在表上判断每一行是否有锁。意向排它锁存在的价值在于节约 InnoDB 对于锁的定位和处理性能。另外注意了，除了全表扫描以外意向锁都不会阻塞。
[​

](https://blog.csdn.net/yue_2018/article/details/89047758)
InnoDB 有 3 种行锁的算法：

- **Record Lock**：记录锁，存在于包括主键的唯一索引，单个行记录上的锁。
- **Gap Lock**：间隙锁，存在于非唯一索引，对索引项之间的“间隙”加锁，锁定记录的范围（对第一条记录前的间隙或最后一条将记录后的间隙加锁），不包含索引项本身。这种锁只会存在于**范围查询，而对于单个条件的查询则会用临键锁。**
- **Next-Key Lock**：临键锁，存在于非唯一索引，锁定一个范围，并且锁定记录本身。主要解决的问题是 RR 隔离级别下的幻读。



#### 死锁
&ensp;&ensp;&ensp;&ensp;锁之间的互相等待，就会造成死锁，比如以下例子：
事务a           									事务b
Begin;														begin;
Select * from user where id = 1 for update;						Select * from user where id = 2 for update;
Insert into user(“name”) value(“哈哈”) 	 where id = 1;		Insert into user(“name”) value(“哈哈”)  where id =2;	
Commit;									      					Commit;
&ensp;&ensp;&ensp;&ensp;此时a、b就会互相等待死锁。
​

**检测死锁**：数据库系统实现了各种死锁检测和死锁超时的机制。InnoDB存储引擎能检测到死锁的循环依赖并立即返回一个错误。
**死锁恢复**：死锁发生以后，只有部分或完全回滚其中一个事务，才能打破死锁，InnoDB目前处理死锁的方法是，**将持有最少行级排他锁的事务进行回滚**。所以事务型应用程序在设计时必须考虑如何处理死锁，多数情况下只需要重新执行因死锁回滚的事务即可。
**外部锁的死锁检测**：发生死锁后，InnoDB 一般都能自动检测到，并使一个事务释放锁并回退，另一个事务获得锁，继续完成事务。但在涉及外部锁，或涉及表锁的情况下，InnoDB 并不能完全自动检测到死锁， 这需要通过设置锁等待超时参数 innodb_lock_wait_timeout 来解决
**死锁影响性能**：死锁会影响性能而不是会产生严重错误，因为InnoDB会自动检测死锁状况并回滚其中一个受影响的事务。在高并发系统上，当许多线程等待同一个锁时，死锁检测可能导致速度变慢。 有时当发生死锁时，禁用死锁检测（使用innodb_deadlock_detect配置选项）可能会更有效，这时可以依赖innodb_lock_wait_timeout设置进行事务回滚。
​

**如何避免死锁**

- 为了在单个InnoDB表上执行多个并发写入操作时避免死锁，可以在事务开始时通过为预期要修改的每个元祖（行）使用**SELECT ... FOR UPDATE**语句来获取必要的锁，即使这些行的更改语句是在之后才执行的。
- 在事务中，如果要更新记录，应该直接申请足够级别的锁，即排他锁，而不应先申请共享锁、更新时再申请排他锁，因为这时候当用户再申请排他锁时，其他事务可能又已经获得了相同记录的共享锁，从而造成锁冲突，甚至死锁
- 如果事务需要修改或锁定多个表，则应在每个事务中以相同的顺序使用加锁语句。 在应用中，如果不同的程序会并发存取多个表，应尽量约定以相同的顺序来访问表，这样可以大大降低产生死锁的机会
- 改变事务隔离级别，改成串行化

​

&ensp;&ensp;&ensp;&ensp;如果出现死锁，可以用** show engine innodb status**命令来确定最后一个死锁产生的原因。返回结果中包括死锁相关事务的详细信息，如引发死锁的 SQL 语句，事务已经获得的锁，正在等待什么锁，以及被回滚的事务等。据此可以分析死锁产生的原因和改进措施。
​

​

#### mvcc多版本并发控制
&ensp;&ensp;&ensp;&ensp;在内部，innodb会为每一行数据添加三个额外的字段

- **db_trx _ id**：一个6字节的 db_trx _ id 字段表示插入或更新行的最后一个事务的事务标识符。此外，删除在内部被视为更新，其中行中的一个特殊位被设置为将其标记为 deleted。
- **db_roll _ ptr**：一个7字节的 db_roll _ ptr 字段称为滚动指针。滚动指针指向写入回滚段的 undo log记录。如果更新了行，则 undo log记录包含在更新行之前重新生成其内容所必需的信息。
- **db_row _ ID** ：一个6字节的 db_row _ ID 字段包含一个行 ID，该行 ID 随着插入新行而单调增加。如果 InnoDB 自动生成聚集索引，则索引包含行 ID 值。否则，db_row _ id 列不会出现在任何索引中。

​

&ensp;&ensp;&ensp;&ensp;innodb在select的时候，创建新的**read view**时，会把全局读写事务id（活跃的事务id）拷贝到本地descriptors，设置up_limit_id为这个descriptors里最小的值作为低水位，设置low_limit_id为创建readview时应该分给下一个事务的id作为高水位。
1、当db_trx_id（记录的事务id）小于低水位时，说明这条记录的修改是在创建rv之前，可见状态。
2、当该id大于高水位，说明这条记录的修改是在创建rv之后，不可见状态。
3、当该id处于两者之间时，当不在descripors里时，说明是已提交的事务而可见，而在descripors里时说明被其他事务修改中不可见。
4、当不可见状态时，会根据db_roll_ptr来获取undo log的有关上一个版本的数据并重新进行比较，直到找到一个能够被当前事务能够看到的版本返回具体数据。
&ensp;&ensp;&ensp;&ensp;read commited与repeatable read的区别在于，前者在每次select都会新建一个readview，后者会在每次事务新建并共用一个readview，所以能够解决幻读问题。
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/6.png)




### MYISAM与InnoDB的区别
#### 存储结构
**MyISAM：**每个MyISAM在磁盘上存储成三个文件。分别为：**表定义文件、数据文件、索引文件。**第一个文件的名字以表的名字开始，扩展名指出文件类型。.frm文件存储表定义。数据文件的扩展名为.MYD (MYData)。索引文件的扩展名是.MYI (MYIndex)。
**InnoDB：**所有的表都保存在同一个数据文件中（也可能是多个文件，或者是独立的表空间文件），InnoDB表的大小只受限于操作系统文件的大小，一般为2GB。
#### 存储空间
**MyISAM：** MyISAM支持支持三种不同的存储格式：静态表(默认，但是注意数据末尾不能有空格，会被去掉)、动态表、压缩表。当表在创建之后并导入数据之后，不会再进行修改操作，可以使用压缩表，极大的减少磁盘的空间占用。
**InnoDB：** 需要更多的内存和存储，它会在主内存中建立其专用的缓冲池用于高速缓冲数据和索引。
#### 可移植性、备份及恢复
**MyISAM：**数据是以文件的形式存储，所以在跨平台的数据转移中会很方便。在备份和恢复时可单独针对某个表进行操作。
**InnoDB：**免费的方案可以是拷贝数据文件、备份 binlog，或者用 mysqldump，在数据量达到几十G的时候就相对痛苦了。
#### 事务支持
**MyISAM：**强调的是性能，每次查询具有原子性,其执行数度比InnoDB类型更快，但是不提供事务支持。
**InnoDB：**提供事务支持事务，外部键等高级数据库功能。 具有事务(commit)、回滚(rollback)和崩溃修复能力(crash recovery capabilities)的事务安全(transaction-safe (ACID compliant))型表。
#### 表锁差异
**MyISAM：** 只支持表级锁，用户在操作myisam表时，select，update，delete，insert语句都会给表自动加锁，如果加锁以后的表满足insert并发的情况下，可以在表的尾部插入新的数据。
**InnoDB：** 支持事务和行级锁，是innodb的最大特色。行锁大幅度提高了多用户并发操作的新能。但是InnoDB的行锁，只是在WHERE的主键是有效的，非主键的WHERE都会锁全表的。
#### 全文索引
**MyISAM：**支持 FULLTEXT类型的全文索引
**InnoDB：**不支持FULLTEXT类型的全文索引，但是innodb可以使用sphinx插件支持全文索引，并且效果更好。
#### 表主键
**MyISAM：**允许没有任何索引和主键的表存在，索引都是保存行的地址。
**InnoDB：**如果没有设定主键或者非空唯一索引，就会自动生成一个6字节的主键(用户不可见)，数据是主索引的一部分，附加索引保存的是主索引的值。
#### 表的具体行数
**MyISAM：** 保存有表的总行数，如果select count(_) from table;会直接取出出该值。_
**InnoDB：** 没有保存表的总行数，如果使用select count(*) from table；就会遍历整个表，消耗相当大，但是在**加了wehre条件**后，myisam和innodb处理的方式都一样。
#### CRUD操作
**MyISAM：**如果执行大量的SELECT，MyISAM是更好的选择。
**InnoDB：**如果你的数据执行大量的INSERT或UPDATE，出于性能方面的考虑，应该使用InnoDB表。
#### 外键
**MyISAM：**不支持
**InnoDB：**支持
​

​

## 索引


### 索引的分类
&ensp;&ensp;&ensp;&ensp;下图为hash跟b+树的数据结构中被支持的存储引擎：
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/7.png)

- 数据结构类型
   - B+树索引
   - Hash索引
   - Full-Text全文索引，
   - R-Tree索引，空间索引是MyISAM的一种特殊索引类型，主要用于地理空间数据类型。
- 物理存储类型
   - 聚集索引（clustered index）
   - 非聚集索引（non-clustered index），也叫辅助索引（secondary index）聚集索引和非聚集索引都是B+树结构
- 逻辑类型
   - 主键索引：主键索引是一种特殊的唯一索引，不允许有空值
   - 普通索引或者单列索引：每个索引只包含单个列，一个表可以有多个单列索引
   - 多列索引（复合索引、联合索引）：复合索引指多个字段上创建的索引，只有在查询条件中使用了创建索引时的第一个字段，索引才会被使用。使用复合索引时遵循最左前缀集合
   - 唯一索引或者非唯一索引
   - 空间索引：空间索引是对空间数据类型的字段建立的索引，MYSQL中的空间数据类型有4种，分别是GEOMETRY、POINT、LINESTRING、POLYGON。 MYSQL使用SPATIAL关键字进行扩展，使得能够用于创建正规索引类型的语法创建空间索引。创建空间索引的列，必须将其声明为NOT NULL，空间索引只能在存储引擎为MYISAM的表中创建



### b+树索引
&ensp;&ensp;&ensp;&ensp;首先来看图了解它的构造：
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/8.png)


&ensp;&ensp;&ensp;&ensp;对于b树来讲，它不需要在每个节点中都存储相关的数据，反而把数据都放到了叶子节点（**myisam叶子节点存储的是数据记录的地址**，如下图）。所以b树有明显的2个缺点：

- 插入删除新的数据记录会破坏b树的性质，因此在插入删除时，需要对树进行一个分裂、合并、转移等操作以保持b树性质。造成IO操作频繁。
- 区间查找可能需要返回上层节点重复遍历，IO操作繁琐。

​

![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/9.png)


#### 如何从磁盘读取数据？
&ensp;&ensp;&ensp;&ensp;磁盘IO时间 = 寻道 + 磁盘旋转 + 数据传输时间，从磁盘读取数据时，系统会将逻辑地址发给磁盘，磁盘将逻辑地址转换为物理地址(哪个磁道，哪个扇区)。 磁头进行机械运动，先找到相应磁道，再找该磁道的对应扇区，扇区是磁盘的最小存储单元。
&ensp;&ensp;&ensp;&ensp;myisam每次都会先一次性加载索引到内存中，然后根据查到的物理地址去一次获取对应地址所在的扇区的数据到操作系统的缓存中存储起来，用来减少磁盘读取。
&ensp;&ensp;&ensp;&ensp;而innodb的话定义了一个树的节点为一个数据页，每访问一个节点就需要去磁盘读取加载到内存中，一次取一个数据页缓存起来，如果没有符合条件的再进行下次磁盘读取，如下图所示：
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/10.png)


&ensp;&ensp;&ensp;&ensp;数据页由7大部分组成：

- file header，文件头部，38字节
- page header，页面头部，56字节
- Infimum + Supremum， Infimum记录是比该页中任何主键值都要小的记录，Supremum记录 是比改页中何主键值都要大的记录。
- user records, 行记录
- free space，空闲空间
- page directory，页面目录
- file trailer，文件尾部，8字节

​

&ensp;&ensp;&ensp;&ensp;在myisam的索引文件中（MYI），连续的单元组成一个Block，索引块index block的大小等于该索引节点的大小，为了最小化磁盘I/O，myisam将最频繁访问的索引块都放在内存中，这样的内存缓冲区我们称之为Key Cache，它的大小可以通过参数**key_buffer_size**来控制，Key Cache就是以Block为单位的。至于数据的话由操作系统的缓存来存储的。
&ensp;&ensp;&ensp;&ensp;而innodb在mysql启动一段时间后,将经常访问的innodb引擎表的数据放入innodb_buffer_pool.**即innodb_buffer_pool**保存的是热数据.然后根据一定算法淘汰不常访问的数据。在5.6版本之后，mysql支持关闭mysql服务时将内存中的热数据保存到硬盘，MySQL重启后首先将硬盘中的如数据加载到Innodb缓冲池中，以便缩短warmup进程的时间，提高业务繁忙高并发时的效率。




#### 千万级的数据对于innodb需要几次查询？
&ensp;&ensp;&ensp;&ensp;在b+树里一个节点存储的内容是：

- 非叶子节点：主键+指针
- 叶子节点：数据

​

&ensp;&ensp;&ensp;&ensp;一个数据页大小默认是16k，假设我们一行数据大小为1K，那么**一页就能存16条数据**，也就是一个叶子节点能存16条数据；再看非叶子节点，假设主键ID为bigint类型，那么长度为8B，指针大小在Innodb源码中为6B，一共就是14B，那么一页里就可以存储16K/14=1170个(主键+指针)，那么一颗高度为2的b+树能存储的数据为：1170*16=18720条，一颗高度为3的B+树能存储的数据为：11701170*16=21902400（千万级条）。所以在innodb中B+树高度一般为1-3层，它就能满足千万级的数据存储。在查找数据时一次页的查找代表一次IO，所以通过主键索引查询通常只需要1-3次IO操作即可查找到数据。
#### 索引的优化方案

- 尽量使用主键索引查询。
- 多建立联合索引，根据最左匹配原则，比如(a,b,c)，那么其实我们可利用的索引就有(a), (a,b), (a,b,c)，这样就减少了冗余索引。
- 不在索引列上做任何操作（计算、函数、(自动or手动)类型转换），会导致索引失效而转向全表扫描。
- 存储引擎不能使用索引中范围条件右边的列。
- 尽量使用覆盖索引(只访问索引的查询(索引列和查询列一致))，减少select。
- is null ,is not null 也无法使用索引。
- 还是最左匹配原则，like "xxxx%" 是可以用到索引的，like "%xxxx" 则不行(like "%xxx%" 同理)。like以通配符开头('%abc...')索引失效会变成全表扫描的操作，
- 字符串不加单引号索引失效。
- 少用or，用它来连接时会索引失效。
- <，<=，=，>，>=，BETWEEN，IN 可用到索引，<>，not in ，!= 则不行，会导致全表扫描。

## innodb的缓存
### 缓冲池
&ensp;&ensp;&ensp;&ensp;我们知道对于内存与磁盘，肯定是内存的速度更快，所以在业务上使用缓存来加快读取速度，而mysql也有这种东西，称之为缓冲池。
&ensp;&ensp;&ensp;&ensp;InnoDB 的缓冲池叫buffer pool，当需要访问某个页时，就会把这一页的数据全部加载到缓冲池中，这样就可以在内存中进行读写访问了。对于数据库中页的修改操作，也是先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。有了缓冲池，就可以省去很多磁盘IO的开销了，从而提升数据库性能。
&ensp;&ensp;&ensp;&ensp;注意即使只访问页中的一条记录，也需要把整个页的数据加载到内存中。通过索引可以定位到磁盘中的页，将页加载到内存后，就可以通过页目录（Page Directory）去定位到某条具体的记录。
&ensp;&ensp;&ensp;&ensp;buffer pool是mysql在启动的时候就会向操作系统申请的一块内存，其大小可由**innodb_buffer_pool_size**控制**，**默认为128M，官方建议配置为物理内存的50%-60%**。**
&ensp;&ensp;&ensp;&ensp;buffer pool也是按页来划分的，默认和磁盘上的页一样大小。buffer pool 中不只缓存了数据页，还包括索引页、undo页、插入缓冲、自适应哈希索引、InnoDB存储的锁信息、数据字典信息等。
&ensp;&ensp;&ensp;&ensp;为了管理buffer pool中的缓存页，InnoDB 为每一个缓存页都创建了一些描述信息（元数据），用来描述这个缓存页。描述信息主要包括该页所属的**表空间编号、页号、缓存页的地址、链表节点信息、锁信息、LSN信息**等等。
&ensp;&ensp;&ensp;&ensp;另外需要注意下，每个描述数据大约相当于缓存页大小的 5%，也就是800字节左右的样子。而我们设置的 innodb_buffer_pool_size 并不包含描述数据的大小，实际上 Buffer Pool 的大小会超出这个值。比如默认配置 128MB，那么InnoDB在为 buffer pool 申请连续的内存空间时，会申请差不多 128 + 128*5% ≈ 134MB 大小的空间。
&ensp;&ensp;&ensp;&ensp;mysql启动时就会为buffer pool申请一片连续的内存空间，然后按照默认的数据页的大小划分出一个个的页来，还会按照800字节左右的大小划分出页对应的描述数据来。划分完成后， buffer pool  中的缓存页都是空的，等执行增删改查等操作时，才会把数据对应的页从磁盘加载到  buffer pool  中的页来。
### 缓存页hash表
&ensp;&ensp;&ensp;&ensp;innodb设计了一个hash表(表空间号+数据页号为key值)，当执行增删改查等操作时会先去这个hash表里查该页是否被缓存了，来判断是从buffer pool获取还是从磁盘获取。当需要从磁盘获取获取并加载到buffer pool时，就会用到了free链表。
### free链表
&ensp;&ensp;&ensp;&ensp;innodb设计了一个**free链表，**来展示buffer pool里对于空闲页的信息。它是一个双向链表数据结构，这个链表的每个节点就是一个空闲缓存页的描述信息。
实际上，每个描述信息中有 **free_pre、free_next **两个指针，free链表 就是由这两个指针连接起来形成的一个双向链表。然后 free链表 有一个基础节点，这个基础节点存放了链表的头节点地址、尾节点地址，以及当前链表中节点数的信息。free链表 看起来就像下图这样：
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/11.png)


&ensp;&ensp;&ensp;&ensp;需要注意的是，链表的基础节点占用的内存空间并不包含在 buffer pool  之内，而是单独申请的一块内存空间，每个基节点只占用40字节大小。后边介绍的其它链表的基础节点也是一样的，都是单独申请的一块40字节大小的内存空间。
&ensp;&ensp;&ensp;&ensp;有了这个 free链表 之后，当需要从磁盘加载一个页到 buffer pool  时，就从 free链表 中取出一个描述数据块，然后将页写入这个描述数据块对应的空闲缓存页中。并把一些描述数据写入描述数据块中，比如页的表空间号、页号之类的。最后，把缓存页对应的描述数据块从 free链表 中移除，表示该缓存页已被使用了。
​

### flush链表
&ensp;&ensp;&ensp;&ensp;当我们执行增删改的时候，肯定是去更新了 buffer pool 中的某些缓存页，那这些被更新了的缓存页就和磁盘上的数据页不一致了，就变成了脏页。这些脏页最终肯定会被刷回到磁盘中，但并不是所有的缓存页都需要刷回到磁盘，因为有些页只是被查询了，但并没有被增删改过。于是flush链表就是这样被设计出来的，专门存储那些被修改的缓存页，在合适的时机将数据页刷回磁盘。
&ensp;&ensp;&ensp;&ensp;其描述信息有两个指针** flush_pre、flush_next。**
​

### LRU链表
&ensp;&ensp;&ensp;&ensp;为了最大化利用内存，一些不常用的数据当然希望是尽早清除掉，于是就有了LRU链表**Least Recently Used**，意思就是最近最少使用的链表，用来清除掉一些不常用的缓存数据页。LRU链表与 free链表的结构是类似的，都会有一个基础节点来指向链表的首、尾描述信息块，加入LRU链表中的描述信息块就通过 free_pre 和 free_next 两个指针连接起来行程一个双向链表。
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/12.png)

&ensp;&ensp;&ensp;&ensp;mysql有个预读机制，如果顺序访问了一个区里的多个数据页，就会触发预读机制，把下一个区中所有的数据页都加载到缓存页里，由**innodb_read_ahead_threshold**控制，默认是56，而是否开启通过参数**innodb_random_read_ahead**控制，默认是off。为了最大化利用内存，LRU链表采用了冷热数据分离的思想，LRU链表会被拆成两部分，一部分是**热数据**（又称new列表），一部分是**冷数据**（又称old列表）。
&ensp;&ensp;&ensp;&ensp;这个冷热数据的位置并不固定，是一个比例，由参数 **innodb_old_blocks_pct **来控制，默认比例是 37，也就是冷数据占 37%，大约占 LRU链表 3/8 的样子。
&ensp;&ensp;&ensp;&ensp;基于冷热分离的LRU链表，这时新加载一个缓存页时，就不是直接放到LRU的头部了，而是放到冷数据区域的头部。 InnoDB 设置了一个规则，在第一次访问冷数据区域的缓存页的时候，就在它对应的描述信息块中记录第一次访问的时间，默认要间隔1秒后再访问这个页，才会被移到热数据区域的头部。也就是从第一次加载到冷数据区域后，1秒内多次访问都不会移动到热数据区域，基本上全表扫描查询缓存页的操作1秒内就结束了。
&ensp;&ensp;&ensp;&ensp;这个间隔时间是由参数** innodb_old_blocks_time** 控制的，默认是 1000毫秒。如果我们把这个参数值设置为0，那么每次访问一个页面时就会把该页面放到热数据区域的头部。
&ensp;&ensp;&ensp;&ensp;之后缓存页不够用的时候，就会优先从冷数据区域的尾部进行刷盘清空，频繁访问的数据页还是会留在热数据区域，不会受到影响。而冷数据区域停留超过1秒的页，被再次访问时就会移到热数据区域的头部。
&ensp;&ensp;&ensp;&ensp;热数据区域中的页是每访问一次就移到头部吗？也不是的，热数据区域是最频繁访问的数据，如果频繁的对LRU链表进行节点移动操作也是不合理的。所以 InnoDB 就规定只有在访问了热数据区域的 后3/4 的缓存页才会被移动到链表头部，访问 前1/4 中的缓存页是不会移动的。
​

### 脏数据刷盘
#### 内存不足
&ensp;&ensp;&ensp;&ensp;当buffer pool无法为新的缓存页添加数据时，就会去清除LRU链表里的数据页，如果清理过程中发现该数据页在flush链表存在的话，就进行刷盘。
#### page cleaner
&ensp;&ensp;&ensp;&ensp;在mysql中会有**page cleaner**的线程，每秒一次专门把flush链表和LRU链表里的数据刷盘。有2个关联主要参数：

- **innodb_io_capacity**，计算机存储设备每秒的读写次数，默认200
- **innodb_io_capacity_max**，当刷新活动比较慢时，innodb就把读写次数提升，默认取innodb_io_capacity的2倍跟2000中较大的值。

&ensp;&ensp;&ensp;&ensp;LRU链表有个参数是**innodb_lru_scan_depth，默认1024k，**该参数表示从LRU链表扫描的深度，调大该值有助于多释放些空闲页，避免用户线程去做**single page flush**。
&ensp;&ensp;&ensp;&ensp;在5.7.4版本里引入了多个page cleaner线程，从而达到并行刷脏的效果。
​

### 缓冲池的优化
&ensp;&ensp;&ensp;&ensp;多线程访问 buffer pool 的时候，会涉及到对同一个 free、lru、flush 等链表的操作，例如节点的移动、缓存页的刷新等，那必然是会涉及到加锁的。
&ensp;&ensp;&ensp;&ensp;首先要知道，就算只有一个 buffer pool ，多线程访问要加锁、释放锁，由于基本都是内存操作，所以性能也是很高的。但在一些高并发的生产环境中，配置多个 buffer Pool，还是能极大地提高数据库并发性能的。
&ensp;&ensp;&ensp;&ensp;可以通过参数 **innodb_buffer_pool_instances** 来配置 buffer pool 实例数，通过参数** innodb_buffer_pool_size** 设置所有 buffer pool 的总大小（单位字节），每个 buffer pool 的大小就是 innodb_buffer_pool_size / innodb_buffer_pool_instances。注意的是，InnoDB 规定，当 innodb_buffer_pool_size** 小于1GB**的时候，设置多个实例是无效的，会默认把innodb_buffer_pool_instances 的值修改为1。
&ensp;&ensp;&ensp;&ensp;我们可以在运行时动态调整 innodb_buffer_pool_size 这个参数，但 InnoDB 并不是一次性申请 pool_size 大小的内存空间，而是以 chunk 为单位申请。**一个 chunk 默认就是 128M**，代表一片连续的空间，申请到这片内存空间后，就会被分为若干缓存页与其对应的描述信息块。每个chunk 的大小由参数 **innodb_buffer_pool_chunk_size** 控制，这个参数只能在服务器启动时指定，不能在运行时动态修改。
&ensp;&ensp;&ensp;&ensp;合理设置buffer pool的大小，一般建议是物理机内存的**50%-60%**，**innodb_buffer_pool_size 必须是 innodb_buffer_pool_chunk_size × innodb_buffer_pool_instances 的倍数**，主要是保证每一个Buffer Pool实例中包含的chunk数量相同。
​

## 重要的三个日志
### redo日志
&ensp;&ensp;&ensp;&ensp;redo日志主要是为了恢复数据用，因为当数据还在buffer pool的时候就宕机了会导致丢失，于是设计了redo log，在提交事务的时候，**先把对缓存页的修改以日志的形式，写到 redo log 文件里去**，而且保证写入文件成功才算事务提交成功。而且redo log是顺序写入磁盘文件，每次都是追加到磁盘文件末尾去，速度是非常快的。之后再在某个时机将修改的缓存页刷入磁盘，这时就算数据库宕机，也可以利用redo log来恢复数据。这就是mysql里提到的**write-ahead-logging**，**先写日志再写磁盘**。
&ensp;&ensp;&ensp;&ensp;redo log 本质上记录的就是对某个表空间的某个数据页的某个偏移量的地方修改了几个字节的值，它需要记录的其实就是 **日志类型 + 表空间号+数据页号+偏移量+修改的长度+具体的值**，所以 redo log 占用的空间非常小，一条 redo log 也就几个字节到几十个字节的样子。
​

&ensp;&ensp;&ensp;&ensp;一个事务中可能有多个增删改的SQL语句，而一个SQL语句在执行过程中可能修改若干个页面，会有多个操作。以InnoDB将执行语句的过程中产生的redo log划分成了若干个不可分割的组，一组redo log就是对底层页面的一次原子访问，这个原子访问也称为 **Mini-Transaction**，简称 mtr。一个 mtr 就包含一组redo log，在崩溃恢复时这一组redo log就是一个不可分割的整体。
&ensp;&ensp;&ensp;&ensp;一个事务可以包含若干条SQL语句，每一条SQL语句其实是由若干个mtr组成，每一个mtr又可以包含若干条redo log。
&ensp;&ensp;&ensp;&ensp;redo log 并不是一条一条写入磁盘的日志文件中的，而且一个原子操作的 mtr 包含一组 redo log，一条一条的写就无法保证写磁盘的原子性了。
&ensp;&ensp;&ensp;&ensp;InnoDB设计了一个 **redo log block** 的数据结构，称为**重做日志块（block）**，重做日志块跟缓存页有点类似，只不过日志块记录的是一条条 redo log。一个 mtr 中的 redo log 实际上是先写到一个地方，然后再将一个 mtr 的日志记录复制到block中，最后在一些时机将block**刷新到磁盘日志文件**中。
&ensp;&ensp;&ensp;&ensp;一个 redo log block 固定 512字节 大小，由三个部分组成：**header块头（12字节） + body块体（496字节） + trailer块尾（4字节）**。redo log 就是存放在 body 块体中，也就是一个块实际只有 496字节 用来存储 redo log。
**header **块头记录了四个信息：

- LOG_BLOCK_HDR_NO：表示块的唯一编号。
- LOG_BLOCK_HDR_DATA_LEN：表示 block 中已经使用了多少字节，初始值为12，因为body从第12个字节处开始。如果block body已经被全部写满，那么本属性的值就被设置为512。
- LOG_BLOCK_FIRST_REC_GROUP：表示block中第一个mtr日志组中的第一条 redo log 的偏移量。
- LOG_BLOCK_CHECKPOINT_NO：表示 checkpoint 的序号，后面会介绍。

**trailer** 只记录了一个信息：

- LOG_BLOCK_CHECKSUM：表示block的校验值。



&ensp;&ensp;&ensp;&ensp;跟 buffer pool 类似的，服务器启动时，就会申请一块连续的内存空间，作为 redo log block 的缓冲区也就是 **redo log buffer**。然后这片内存空间会被划分成若干个连续的 redo log block，redo log 就是先写到 redo log buffer 中的 redo log block 中的。
&ensp;&ensp;&ensp;&ensp;可以通过启动参数**innodb_log_buffer_size**来指定log buffer的大小，该参数的默认值为16MB。
​

&ensp;&ensp;&ensp;&ensp;redo log buffer跟buffer pool一样，都会在一些时机里**刷盘，**主要有以下几个时机：

- **log buffer 空间不足时**，如果写入 log buffer 的日志占据了 log buffer 总容量的一半了，默认情况下也就是超过8MB的时候，此时就会把他们刷入到磁盘文件里去。这种情况一般在高并发的场景下可能会出现，每秒执行了很多增删改SQL语句，产生的redo log 瞬间超过了8M，然后就立马触发刷新 log block 到磁盘。不过这种情况一般比较少。
- **事务提交时**，一个事务提交的时候，必须把它的redo log都刷入到磁盘文件里去，只有这样，才能保证事务的持久性，才算事务提交成功了（这就是force log at commit机制，即在事务提交的时候，必须先将该事务的所有事务日志写入到磁盘上的日志文件中进行持久化）。如果在写入的过程中MySQL宕机了，那事务也就失败了。
- **后台线程刷盘**，后台有一个线程会每隔1秒，把redo log block刷到磁盘文件里去。
- **MySQL关闭的时候**，redo log block都会刷入到磁盘里去。
- **做 checkpoint 的时候**。这个后面会说。

&ensp;&ensp;&ensp;&ensp;需要注意的是，不管什么时机刷盘，redo log block 始终是**顺序刷盘**的，比如事务提交的时候，会把这个事务mtr之前的block都刷入磁盘。
​

&ensp;&ensp;&ensp;&ensp;比较重要的是，在提交事务的时候，InnoDB会根据配置的策略来将 redo log 刷盘，这个参数可以通过**innodb_flush_log_at_trx_commit** （默认值为1）来配置。
&ensp;&ensp;&ensp;&ensp;可以配置如下几个值：

- 0：事务提交时不会立即向磁盘中同步 redo log，而是由后台线程来刷。这种策略可以提升数据库的性能，但事务的持久性无法保证。
- 1：事务提交时会将 redo log 刷到磁盘，这可以保证事务的持久性，这也是默认值。其实数据会先写到操作系统的缓冲区（os cache），这种策略会调用 fsync 强制将 os cache 中的数据刷到磁盘。
- 2：事务提交时会将 redo log 写到操作系统的缓冲区中，可能隔一小段时间后才会从系统缓冲区同步到磁盘文件。这种情况下，如果机器宕机了，而系统缓冲区中的数据还没同步到磁盘的话，就会丢失数据。

&ensp;&ensp;&ensp;&ensp;为了保证事务的持久性，一般使用默认值，将 innodb_flush_log_at_trx_commit 设置为1即可。
​

&ensp;&ensp;&ensp;&ensp;mysql设定了一个文件组的形式来持久化redo log，文件名的格式为 **ib_logfile[x]**（x 为从0开始的数字），由以下参数控制：

- **innodb_log_group_home_dir**：指定redo log文件所在的目录，默认值就是当前的数据目录。
- **innodb_log_file_size**：指定每个redo log文件的大小，默认值为48MB。
- **innodb_log_files_in_group**：指定redo log文件的个数，默认值为2，最大值为100。

​

&ensp;&ensp;&ensp;&ensp;在将 redo log 写入日志文件组时，是从 ib_logfile0 开始写，如果 ib_logfile0 写满了，就接着ib_logfile1 写，ib_logfile1 写满了就去写 ib_logfile2，依此类推。如果写到最后一个文件也满了，就会重新转到ib_logfile0覆盖写入。
​

&ensp;&ensp;&ensp;&ensp;log block 固定为512字节大小，redo log 文件也是一样按512字节来划分的，每个 redo log 文件的格式也是一样的，都由若干个512字节的块组成。
&ensp;&ensp;&ensp;&ensp;每个 redo log 文件由两部分组成：

- 前2048字节，也就是前4个block是用来存储一些管理信息。其中第1个 block 存储header，第2个和第4个存储checkpoint，第3个block保留未没用。
- 从第2048字节往后是用来存储 redo log block 的。

&ensp;&ensp;&ensp;&ensp;所以在循环写日志文件的时候，其实是从每个日志文件的第2048字节 开始的。但需要注意的是，一组日志文件中，**只有第1个日志文件的前4个block才会存储管理信息**，其余的日志文件只是保留这些空间，不存储信息。
​

**header **中的各个属性：

- LOG_HEADER_FORMAT：redo日志的版本
- LOG_HEADER_PAD1：做字节填充用的，没什么实际意义
- LOG_HEADER_START_LSN：标记本日志文件开始的LSN值，初始值就2048，指向文件偏移量2048字节处。
- LOG_HEADER_CREATOR：标记本日志文件的创建者。
- LOG_BLOCK_CHECKSUM：本block的校验值

**checkpoint **中的各个属性：

- LOG_CHECKPOINT_NO：服务器做checkpoint的编号，每做一次checkpoint，该值就加1。
- LOG_CHECKPOINT_LSN：服务器做checkpoint结束时对应的LSN值，系统崩溃恢复时将从该值开始。
- LOG_CHECKPOINT_OFFSET：上个属性中的LSN值在redo日志文件组中的偏移量。
- LOG_CHECKPOINT_LOG_BUF_SIZE：服务器在做checkpoint操作时对应的log buffer的大小。
- LOG_BLOCK_CHECKSUM：本block的校验值。

​

#### LSN
&ensp;&ensp;&ensp;&ensp;上面可以看到出现了2次**LSN，**mysql设计了一个全局变量：日志序列号**log sequence number，代表了写入的日志总量，初始值是8704，占用8个字节，且是单调递增的。**
&ensp;&ensp;&ensp;&ensp;在每次写入**redo log buffer**时**，**LSN都会叠加记录下已写入的数据量，如下图所示：
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/13.png)


&ensp;&ensp;&ensp;&ensp;事务产生的mtr写入log block后，会将修改的脏页加入到Flush链表头部，Flush链表对应的描述信息块中会有两个属性来记录LSN信息：

- **oldest_modification**：记录mtr开始的LSN值。
- **newest_modification**：记录mtr结束时的LSN值。

&ensp;&ensp;&ensp;&ensp;接着另一个mtr写入后，可能Flush链表中已经存在了对应的脏页，此时会将mtr结束时的LSN值写入newest_modification，原本的oldest_modification则保持不变。
实际上Flush链表中的脏页就是按照修改发生的时间顺序进行排序，也就是按照oldest_modification代表的LSN值进行排序的。链表靠近尾部的是最早修改的，链表头部则是最新修改的。
​

#### checkpoint 
&ensp;&ensp;&ensp;&ensp;redo log 只是为了系统崩溃后恢复脏页用的，如果对应的脏页已经刷新到了磁盘，那么就算崩溃后也用不着这部分 redo log 了，那么它占用的磁盘空间就可以被覆盖重用。如果脏页没有刷入磁盘，那么对应的 redo log 就必须保留着。
&ensp;&ensp;&ensp;&ensp;InnoDB 设计了一个全局变量 **checkpoint_lsn **来代表当前系统中可以被覆盖的redo log总量是多少，这个变量初始值也是8704。当脏页被刷入磁盘时，就会做一次 checkpoint 来计算 checkpoint_lsn 的值，并写入 redo log 文件中。
&ensp;&ensp;&ensp;&ensp;脏页只要已经刷入磁盘，那他们对应的redo log就可以被覆盖，那如何判断哪些脏页已经刷入磁盘呢？
&ensp;&ensp;&ensp;&ensp;前面说过 Flush链表 中的脏页是按修改时间，也就是oldest_modification代表的LSN值排序的，链表尾部的脏页就是最早修改的，它所对应的oldest_modification就是最小的一个LSN值，那这个LSN之前的脏页就是已经刷入磁盘的。
&ensp;&ensp;&ensp;&ensp;在做 checkpoint 时，其实就是将Flush链表尾部的脏页的**oldest_modification赋值给checkpoint_lsn**。
&ensp;&ensp;&ensp;&ensp;接着根据checkpoint_lsn计算对应的redo log文件日志偏移量checkpoint_offset。
&ensp;&ensp;&ensp;&ensp;InnoDB还设计了一个全局变量checkpoint_no，代表checkpoint的次数，每做一次checkpoint，这个值就会加1。
&ensp;&ensp;&ensp;&ensp;然后就会将这些信息写入日志文件组中的第一个日志文件的checkpoint中。至于存到 checkpoint1 还是 checkpoint2，则根据checkpoint_no来计算，如果是偶数，就写到checkpoint1，如果是奇数，就写入checkpoint2。
&ensp;&ensp;&ensp;&ensp;可以看到checkpoint中就有三个属性来存储这些信息：

- checkpoint_no 写入 LOG_CHECKPOINT_NO
- checkpoint_lsn 写入 LOG_CHECKPOINT_LSN
- checkpoint_offset 写入 LOG_CHECKPOINT_OFFSET

![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/14.png)


#### 恢复
&ensp;&ensp;&ensp;&ensp;InnoDB在启动时不管上次数据库是否正常关闭，都会尝试进行恢复操作。如果数据库是正常关闭，redo log 其实没什么用，但如果数据库宕机，redo log 就可以用来恢复数据了。
**恢复的起点**
&ensp;&ensp;&ensp;&ensp;首先要读取日志组中的第一个 redo log 文件头部的两个 checkpoint，先比较其中的 checkpoint_no，哪个大就使用哪个 checkpoint。
&ensp;&ensp;&ensp;&ensp;然后读取 checkpoint_lsn，这个值之前的都是已经刷盘了的，但之后的可能刷盘了，也可能没有刷盘。所以恢复的起点就是 checkpoint_lsn 对应的文件偏移量，从这个偏移量开始读取 redo log 来恢复页面。
**恢复的终点**
&ensp;&ensp;&ensp;&ensp;redo log block 的头部header中有一个属性 LOG_BLOCK_HDR_DATA_LEN 记录了当前block里使用了多少字节的空间，对于被写满的block来说，该属性就是512。如果该属性的值不为512，说明这个block还没写满，那终点就是这个block了。
**使用哈希表**
&ensp;&ensp;&ensp;&ensp;读取到内存中的 redo log，并不是直接就按顺序去重做页的。而是使用了一个哈希表来加快恢复的速度。
&ensp;&ensp;&ensp;&ensp;它会根据 redo log 的表空间ID和页号计算出散列值，以此作为哈希表的 Key，哈希表的 Value 则是一个链表，相同表空间ID和页号的 redo log 就会挨个按顺序加入这个链表中。
&ensp;&ensp;&ensp;&ensp;之后就遍历哈希表来恢复页，因为对同一个页面修改的 redo log 都在一个链表中，所以可以一次性将一个页面修复好（避免了很多读取页面的随机IO），这样可以加快恢复速度。
**跳过已经刷新到磁盘的页面**
&ensp;&ensp;&ensp;&ensp;checkpoint_lsn 之前的可以保证 redo log 对应的脏页已经刷盘了，但是之后的就不能确定了。因为在做 checkpoint 之后，可能一些脏页会不断的被刷到磁盘中，那这部分 redo log 就不能在页中重做一遍。
&ensp;&ensp;&ensp;&ensp;这个时候就会用到前面说过的页中的FIL_PAGE_LSN属性，这个属性记录了最近一次修改页面对应的LSN值。
&ensp;&ensp;&ensp;&ensp;如果在做了某次checkpoint之后有脏页被刷新到磁盘中，那么该页对应的FIL_PAGE_LSN代表的LSN值肯定大于checkpoint_lsn的值，对于这种页面就不需要在应用 redo log 了。
​

### undo日志
&ensp;&ensp;&ensp;&ensp;事务的第一个特性就是原子性，原子性就是要保证一个事务中的增删改操作要么都成功，要么都不做。这时就需要 undo log，在对数据库进行修改前，会先记录对应的 undo log，然后在事务失败或回滚的时候，就可以用这些 undo log 来将数据回滚到修改之前的样子。
&ensp;&ensp;&ensp;&ensp;事务执行过程中在对某个表执行增、删、改操作时，InnoDB就会给这个事务分配一个唯一的事务ID。如果一个事务中没有执行增删改操作，就不会分配事务ID。行记录中会有三个隐藏列：

- DB_ROW_ID：如果没有为表显式的定义主键，并且表中也没有定义唯一索引，那么InnoDB会自动为表添加一个row_id的隐藏列作为主键。
- DB_TRX_ID：事务中对某条记录做增删改时，就会将这个事务的事务ID写入trx_id中。
- DB_ROLL_PTR：回滚指针，本质上就是指向 undo log 的指针。

​

&ensp;&ensp;&ensp;&ensp;每对一条记录做一次改动，就会产生1条或者2条 undo log。一个事务中可能会有多个增删改SQL语句，一个SQL语句可能会产生多条 undo log，一个事务中的这些 undo log 会被从 0 开始递增编号，这个编号称为 undo no。
​

&ensp;&ensp;&ensp;&ensp;事务提交前需要将 undo log 写磁盘，这会造成多次磁盘 IO。
#### 分类
&ensp;&ensp;&ensp;&ensp;分为两个大类来存储：

- TRX_UNDO_INSERT

类型为 TRX_UNDO_INSERT_REC 的 undo log 属于此大类，一般由 INSERT 语句产生，或者在 UPDATE 更新主键的时候也会产生。

- TRX_UNDO_UPDATE

除了类型为 TRX_UNDO_INSERT_REC 的 undo log，其他类型的 undo log 都属于这个大类，比如 TRX_UNDO_DEL_MARK_REC 、 TRX_UNDO_UPD_EXIST_REC ，一般由 DELETE、UPDATE 语句产生。
之所以要分成两个大类，是因为不同大类的 undo log 不能混着存储，因为类型为TRX_UNDO_INSERT_REC的 undo log 在事务提交后可以直接删除掉，而其他类型的 undo log 还需要提供MVCC功能，不能直接删除。
​

#### insert undo
&ensp;&ensp;&ensp;&ensp;插入一条数据对应的undo操作其实就是根据主键删除这条数据就行了。所以 insert 对应的 undo log 主要是把这条记录的主键记录上。
&ensp;&ensp;&ensp;&ensp;INSERT 产生的 undo log 类型为 **TRX_UNDO_INSERT_REC**，大致结构如下图所示：

- start、end：指向记录开始和结束的位置。
- undo type：undo log 的类型，也就是 TRX_UNDO_INSERT_REC。
- undo no：在当前事务中 undo log 的编号。
- table id：表空间ID。
- 主键列信息：这一块就需要记录INSERT这行数据的主键ID信息，或者唯一列信息。

![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/15.png)

#### delete undo
&ensp;&ensp;&ensp;&ensp;删除一条数据大致可以分为两个阶段：

- 阶段一

首先是用户线程执行删除时，会先将记录头信息中的 delete_mask 标记为 1，而不是直接从页中删除，因为可能其它并发的事务还需要读取这条数据。（后面讲MVCC的时候就知道为什么了）

- 阶段二

提交事务后，后台有一个 purge 线程会将数据真正删除。
首先要知道，页中的数据是通过记录头信息中的 netx_record 连接起来的单向链表（假设这个链表称为数据链表）。页中还有另一个链表，称为垃圾链表，记录真正删除后，会从数据链表中移除，然后加入到垃圾链表的头部，以便重用空间。
所以阶段二就是将记录从数据链表移除，加入到垃圾链表的头部。
也就是说，删除操作在事务提交前，只会经历阶段一，就是将记录的 delete_mask 标记为 1。
DELETE 对应的 undo log 类型为 **TRX_UNDO_DEL_MARK_REC**，它的结构大致如下图所示，与 TRX_UNDO_INSERT_REC 类型相比，主要多了三个部分：

- old trx_id：这个属性会保存记录中的隐藏列trx_id，这个属性在MVCC并发读的时候就会起作用了。
- old roll_pointer：这个属性保存记录中的隐藏列roll_pointer，这样就可以通过这个属性找到之前的 undo log。
- 索引列信息：这部分主要是在第二阶段事务提交后用来真正删除记录的。

![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/16.png)

#### update undo
&ensp;&ensp;&ensp;&ensp;在执行UPDATE语句时，InnoDB对更新主键和不更新主键这两种情况有截然不同的处理方案，对应中两种不同的 undo log 类型。
##### 不更新主键
&ensp;&ensp;&ensp;&ensp;在不更新主键的情况下，又可以细分为被更新的列占用的存储空间不发生变化和发生变化的情况。

- 存储空间未发生变化

更新记录时，对于被更新的每个列来说，如果更新后的列和更新前的列占用的字节数都一样大，那么就可以进行**就地更新**，也就是直接在原记录的基础上修改对应列的值。

- 存储空间发生变化

&ensp;&ensp;&ensp;&ensp;如果有任何一个被更新的列更新前和更新后占用的字节数大小不一致，那么就会先把这条旧的记录从聚簇索引页面中删除掉，然后再根据更新后列的值创建一条新的记录插入到页面中。注意这里的删除并不是将 delete_mask 标记为 1，而是真正的删除，从数据链表中移除加入到垃圾链表的头部。
&ensp;&ensp;&ensp;&ensp;如果新的记录占用的存储空间大小不超过旧记录占用的空间，就可以直接重用刚加入垃圾链表头部的那条旧记录所占用的空间，否就会在页面中新申请一段空间来使用。
&ensp;&ensp;&ensp;&ensp;不更新主键的这两种情况生成的 undo log 类型为 **TRX_UNDO_UPD_EXIST_REC**，大致结构如下图所示，与 TRX_UNDO_DEL_MARK_REC 相比主要是多了更新列的信息。
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/17.png)


##### 更新主键
&ensp;&ensp;&ensp;&ensp;要知道记录是按主键大小连成一个单向链表的，如果更新了某条记录的主键值，这条记录的位置也将发生改变，也许就被更新到其它页中了。
&ensp;&ensp;&ensp;&ensp;这种情况下的更新分为两步：

- 首先将原记录做标记删除，就是将 delete_mask 改为 1，还没有真正删除。
- 然后再根据更新后各列的值创建一条新记录，并将其插入到聚簇索引中。

&ensp;&ensp;&ensp;&ensp;所以这种情况下，会产生两条 undo log：

- 第一步标记删除时会创建一条 TRX_UNDO_DEL_MARK_REC 类型的 undo log。
- 第二步插入记录时会创建一条 TRX_UNDO_INSERT_REC 类型的 undo log。
#### undo 回滚
&ensp;&ensp;&ensp;&ensp;前面在一个事务中增删改产生的一系列 undo log，都有 undo no 编号的。在回滚的时候，就可以应用这个事务中的 undo log，根据 undo no 从大到小开始进行撤销操作。
&ensp;&ensp;&ensp;&ensp;但需要注意的是，undo log 是逻辑日志，只是将数据库逻辑地恢复到原来的样子。所有修改都被逻辑地取消了，但是数据结构和页本身在回滚之后可能大不相同。因为同时可能很多并发事务在对数据库进行修改，因此不能将一个页回滚到事务开始的样子，因为这样会影响其他事务正在进行的工作。
​

#### undo 恢复
&ensp;&ensp;&ensp;&ensp;undo log 写入 undo 页后，这个页就变成脏页了，也会加入 Flush 链表中，然后在某个时机刷到磁盘中。
&ensp;&ensp;&ensp;&ensp;事务提交时会将 undo log 放入一个链表中，是否可以最终删除 undo log 及 undo log 所在页，是由后台的一个 purge 线程来完成的。
&ensp;&ensp;&ensp;&ensp;最后也是最为重要的一点是，undo log 写入 undo 页的时候也会产生 redo log，因为 undo log 也需要持久性的保护。


### binlog
&ensp;&ensp;&ensp;&ensp;binlog是一种二进制日志，其主要是用来记录对mysql数据更新或潜在发生更新的SQL语句，并以"事务"的形式保存在磁盘中，作用主要有：

- 复制：在master端开启binlog，master把它的二进制日志传递给slaves并回放来达到master-slave数据一致的目的
- 数据恢复：通过mysqlbinlog工具恢复数据
- 增量备份

​

​

## 分区、分表、分库
&ensp;&ensp;&ensp;&ensp;一般情况下我们创建的表对应一组存储文件，使用MyISAM存储引擎时是一个.MYI和.MYD文件，使用Innodb存储引擎时是一个.ibd和.frm（表结构）文件。当数据量较大时（一般千万条记录级别以上），MySQL的性能就会开始下降，这时我们就需要将数据进行分片，保证mysql的执行性能，有三种方式可实现。


### mysql分区
​

分区类型由以下几种：

- **RANGE分区**：基于属于一个给定连续区间的列值，把多行分配给分区。mysql将会根据指定的拆分策略，,把数据放在不同的表文件上。相当于在文件上,被拆成了小块.但是,对外给客户的感觉还是一张表，透明的。按照 range 来分，就是每个库一段连续的数据，这个一般是按比如**时间范围**来的，比如交易表啊，销售表啊等，可以根据年月来存放数据。可能会产生热点问题，大量的流量都打在最新的数据上了。range 来分，好处在于说，扩容的时候很简单。
- **LIST分区**：类似于按RANGE分区，每个分区必须明确定义。它们的主要区别在于，LIST分区中每个分区的定义和选择是基于某列的值从属于一个值列表集中的一个值，而RANGE分区是从属于一个连续区间值的集合。
- **HASH分区**：基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL 中有效的、产生非负整数值的任何表达式。hash 分发，好处在于说，可以平均分配每个库的数据量和请求压力；坏处在于说扩容起来比较麻烦，会有一个数据迁移的过程，之前的数据需要重新计算 hash 值重新分配到不同的库或表
- **KEY分区**：类似于按HASH分区，区别在于KEY分区只支持计算一列或多列，且MySQL服务器提供其自身的哈希函数。必须有一列或多列包含整数值。

​

但是一般情况下都不会采用分区来水平扩展，而是自己实现分表分库来做，有几个原因：

- 分区表，分区键设计不太灵活，如果不走分区键，很容易出现全表锁
- 一旦数据并发量上来，如果在分区表实施关联，就是一个灾难
- 自己分库分表，自己掌控业务场景与访问模式，可控。分区表，研发写了一个sql，都不确定mysql是怎么玩的，不太可控。

​

### mysql分表
&ensp;&ensp;&ensp;&ensp;分表有两种分割方式，一种垂直拆分，另一种水平拆分。

- **垂直拆分**垂直分表，通常是按照业务功能的使用频次，把主要的、热门的字段放在一起做为主要表。然后把不常用的，按照各自的业务属性进行聚集，拆分到不同的次要表中；主要表和次要表的关系一般都是一对一的。
- **水平拆分(数据分片)**单表的容量不超过500W，否则建议水平拆分。是把一个表复制成同样表结构的不同表，然后把数据按照一定的规则划分，分别存储到这些表中，从而保证单表的容量不会太大，提升性能；当然这些结构一样的表，可以放在一个或多个数据库中。水平分割的几种方法：
   - 使用MD5哈希，做法是对UID进行md5加密，然后取前几位（我们这里取前两位），然后就可以将不同的UID哈希到不同的用户表（user_xx）中了。
   - 还可根据时间放入不同的表，比如：article_201601，article_201602。
   - 按热度拆分，高点击率的词条生成各自的一张表，低热度的词条都放在一张大表里，待低热度的词条达到一定的贴数后，再把低热度的表单独拆分成一张表。
   - 根据ID的值放入对应的表，第一个表user_0000，第二个100万的用户数据放在第二 个表user_0001中，随用户增加，直接添加用户表就行了。

​

### mysql分库
&ensp;&ensp;&ensp;&ensp;一个库里表太多了，导致了海量数据，系统性能下降，把原本存储于一个库的表拆分存储到多个库上， 通常是将表按照功能模块、关系密切程度划分出来，部署到不同库上。
&ensp;&ensp;&ensp;&ensp;优点：

- 减少增量数据写入时的锁对查询的影响
- 由于单表数量下降，常见的查询操作由于减少了需要扫描的记录，使得单表单次查询所需的检索行数变少，减少了磁盘IO，时延变短。
## mysql优化方案
### 性能瓶颈定位
&ensp;&ensp;&ensp;&ensp;我们可以通过 show 命令查看 MySQL 状态及变量，找到系统的瓶颈：
​


- show status ——显示状态信息（扩展show status like ‘XXX’）
- show variables ——显示系统变量（扩展show variables like ‘XXX’）
- show innodb status ——显示InnoDB存储引擎的状态
- show processlist ——查看当前SQL执行，包括执行状态、是否锁表等 
- mysqladmin variables -u username -p password——显示系统变量 
- mysqladmin extended-status -u username -p password——显示状态信息

​

### explain执行计划
&ensp;&ensp;&ensp;&ensp;使用 **Explain** 关键字可以模拟优化器执行sql 查询语句，从而知道 mysql 是如何处理你的 sql 语句的，分析你的查询语句或是表结构的性能瓶颈，它可以为我们带来：

- 表的读取顺序
- 数据读取操作的操作类型
- 哪些索引可以使用
- 哪些索引被实际使用
- 表之间的引用
- 每张表有多少行被优化器查询

![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/18.png)

&ensp;&ensp;&ensp;&ensp;各字段解释

- **id**（select 查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序）
   - id相同，执行顺序从上往下
   - id全不同，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行
   - id部分相同，执行顺序是先按照数字大的先执行，然后数字相同的按照从上往下的顺序执行
- **select_type**（查询类型，用于区别普通查询、联合查询、子查询等复杂查询）
   - **SIMPLE** ：简单的select查询，查询中不包含子查询或UNION
   - **PRIMARY**：查询中若包含任何复杂的子部分，最外层查询被标记为PRIMARY
   - **SUBQUERY**：在select或where列表中包含了子查询
   - **DERIVED**：在from列表中包含的子查询被标记为DERIVED，MySQL会递归执行这些子查询，把结果放在临时表里
   - **UNION**：若第二个select出现在UNION之后，则被标记为UNION，若UNION包含在from子句的子查询中，外层select将被标记为DERIVED
   - **UNION RESULT**：从UNION表获取结果的select
- **table**（显示这一行的数据是关于哪张表的）
- **type**（显示查询使用了那种类型，从最好到最差依次排列	**system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL** ）tip: 一般来说，得保证查询至少达到range级别，最好到达ref
   - system：表只有一行记录（等于系统表），是 const 类型的特例，平时不会出现
   - const：表示通过索引一次就找到了，const 用于比较 primary key 或 unique 索引，因为只要匹配一行数据，所以很快，如将主键置于 where 列表中，mysql 就能将该查询转换为一个常量
   - eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描
   - ref：非唯一性索引扫描，范围匹配某个单独值得所有行。本质上也是一种索引访问，他返回所有匹配某个单独值的行，然而，它可能也会找到多个符合条件的行，多以他应该属于查找和扫描的混合体
   - range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是在你的where语句中出现了between、<、>、in等的查询，这种范围扫描索引比全表扫描要好，因为它只需开始于索引的某一点，而结束于另一点，不用扫描全部索引
   - index：Full Index Scan，index于ALL区别为index类型只遍历索引树。通常比ALL快，因为索引文件通常比数据文件小。（**也就是说虽然all和index都是读全表，但index是从索引中读取的，而all是从硬盘中读的**）
   - ALL：Full Table Scan，将遍历全表找到匹配的行
- **possible_keys**（显示可能应用在这张表中的索引，一个或多个，查询涉及到的字段若存在索引，则该索引将被列出，但不一定被查询实际使用）
- **key**
   - 实际使用的索引，如果为NULL，则没有使用索引
   - **查询中若使用了覆盖索引，则该索引和查询的 select 字段重叠，仅出现在key列表中**
- **key_len**
   - 表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。在不损失精确性的情况下，长度越短越好
   - key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的
- **ref** （显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值）
- **rows** （根据表统计信息及索引选用情况，大致估算找到所需的记录所需要读取的行数）
- **Extra**（包含不适合在其他列中显示但十分重要的额外信息）
   1. using filesort: 说明mysql会对数据使用一个外部的索引排序，不是按照表内的索引顺序进行读取。mysql中无法利用索引完成的排序操作称为“文件排序”。常见于order by和group by语句中
   1. Using temporary：使用了临时表保存中间结果，mysql在对查询结果排序时使用临时表。常见于排序order by和分组查询group by。
   1. using index：表示相应的select操作中使用了覆盖索引，避免访问了表的数据行，效率不错，如果同时出现using where，表明索引被用来执行索引键值的查找；否则索引被用来读取数据而非执行查找操作
   1. using where：使用了where过滤
   1. using join buffer：使用了连接缓存
   1. impossible where：where子句的值总是false，不能用来获取任何元祖
   1. select tables optimized away：在没有group by子句的情况下，基于索引优化操作或对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化
   1. distinct：优化distinct操作，在找到第一匹配的元祖后即停止找同样值的动作

​

​

### 慢查询日志
&ensp;&ensp;&ensp;&ensp;mysql 的慢查询日志是 mysql 提供的一种日志记录，它用来记录在 mysql 中响应时间超过阈值的语句，具体指运行时间超过 long_query_time 值的 sql，则会被记录到慢查询日志中。
&ensp;&ensp;&ensp;&ensp;默认情况下，mysql 数据库没有开启慢查询日志，需要手动设置参数开启：

- set global slow_query_log='ON'  是否开启慢查询日志
- set global slow_query_log_file='/var/lib/mysql/hostname-slow.log' 日志存储地方
- set global long_query_time=10 运行10秒以上的sql才被记录

在生产环境中，如果手工分析日志，查找、分析SQL，还是比较费劲的，所以mysql 提供了日志分析工具**mysqldumpslow**，比如：

- mysqldumpslow -s r -t 10 /var/lib/mysql/slow.log  得到返回记录集最多的10个sql 
- mysqldumpslow -s c -t 10  /var/lib/mysql/slow  得到访问次数最多的10个sql .log
- 也可以和管道配合使用   mysqldumpslow -s r -t 10  /var/lib/mysql/slow.log | more

​

### Show Profile 分析查询
&ensp;&ensp;&ensp;&ensp;通过慢日志查询可以知道哪些 sql 语句执行效率低下，通过 explain 我们可以得知 sql  语句的具体执行情况，索引使用等，还可以结合Show Profile命令查看执行状态。
&ensp;&ensp;&ensp;&ensp;Show Profile 是 mysql提供可以用来分析当前会话中语句执行的资源消耗情况，可以用于sql 的调优的测量。默认情况下，参数处于关闭状态，并保存最近15次的运行结果。
&ensp;&ensp;&ensp;&ensp;确认是否开启：**Show variables like 'profiling'**
&ensp;&ensp;&ensp;&ensp;开启 ：** set profiling=1**
&ensp;&ensp;&ensp;&ensp;诊断**：**show profile cpu,block io for "query_id", 诊断结果一般需要注意的如下：

   - converting HEAP to MyISAM 查询结果太大，内存都不够用了往磁盘上搬了。
   - create tmp table 创建临时表，这个要注意
   - Copying to tmp table on disk 把内存临时表复制到磁盘
   - locked



## mysql高可用
### MHA
&ensp;&ensp;&ensp;&ensp;对主节点进行监控，可实现自动故障转移至其它从节点；通过提升某一从节点为新的主节点，基于主从复制实现，还需要客户端配合实现，目前MHA主要支持一主多从的架构，要搭建MHA,要求一个复制集群中必须最少有三台数据库服务器，一主二从，即一台充当master，一台充当备用master，另外一台充当从库。工作原理：MHA Manager会定时探测集群中的master节点，当master出现故障时，它可以自动将最新数据的slave提升为新的master，然后将所有其他的slave重新指向新的master，整个故障转移过程对应用程序完全透明。
**优点：**

1. 可以进行故障的自动检测和转移。
1. 可扩展性较好，可以根据需要扩展MySQL的节点数量和结构;
1. 相比于双节点的MySQL复制，三节点/多节点的MySQL发生不可用的概率更低​

**缺点：**

1. 至少需要三节点，相对于双节点需要更多的资源;
1. 逻辑较为复杂，发生故障后排查问题，定位问题更加困难;
1. 数据一致性仍然靠原生半同步复制保证，仍然存在数据不一致的风险;
1. 可能因为网络分区发生脑裂现象;
###  Galera
&ensp;&ensp;&ensp;&ensp;基于Galera的MySQL高可用集群， 是多主数据同步的MySQL集群解决方案，使用简单，没有单点故障，可用性高。常见架构如下：
![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/19.png)


**优点：**

1. 多主写入，无延迟复制，能保证数据强一致性；
1. 有成熟的社区，有互联网公司在大规模的使用；
1. 自动故障转移，自动添加、剔除节点；

**缺点：**

1. 需要为原生MySQL节点打wsrep补丁
1. 只支持innodb储存引擎
1. 至少三节点；

​

### 主从复制原理


![](https://gitee.com/littleeight/blog-images/raw/master/mysql%E4%B8%A4%E4%B8%87%E5%AD%97%E7%B2%BE%E5%8D%8E%E6%80%BB%E7%BB%93/20.png)

1. 主库对所有DDL和DML产生的日志写进binlog；
1. 主库生成一个 log dump 线程，用来给从库I/O线程读取binlog；
1. 从库的I/O Thread去请求主库的binlog，并将得到的binlog日志写到relay log文件中；
1. 从库的SQL Thread会读取relay log文件中的日志解析成具体操作，将主库的DDL和DML操作事件重放。

​

&ensp;&ensp;&ensp;&ensp;在5.7版本以后，可使用**并行复制**来提升效率。
