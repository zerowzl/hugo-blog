---
title: "基础架构：一条SQL查询语句是如何执行的？"
date: 2022-06-15T21:10:48+08:00
draft: false
tag: MySQL
---


### MySQL 包含哪些组件？分别有什么作用？
![MySQl架构图](http://localhost:1313/images/jiagou.png)

大体来说，MySQL可以分为server层和存储引擎层。

server层包含连接器（管理连接，权限检查）、查询缓存、分析器（词法分析、语法分析）、优化器（生成执行计划，索引选择）、执行器（操作引擎，返回结果）等，涵盖MySQL基本核心功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有的跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。

MySQL的存储引擎层负责数据的存储和提取。其架构模式采用插件式的，支持InnoDB、MyISAM、Memory等多个存储引擎。从5.5.5版本开始，默认的存储引擎为innoDB。可以在创建表时使用engine=memory来指定使用内存引擎创建表。不同的存储引擎的表数据存取方式不同，支持的功能也不同。

### 一条查询语句的执行流程？


##### 连接器
第一步，你需要先连接上数据库，这就是连接器的工作。连接器负责和客户端建立连接、获取权限、维持和管理连接。命令如下
```sql
mysql -h$ip -P$port -u$user -p
```
输完命令后需要输入密码，连接命令中的mysql是客户端工具，用来和服务端建立连接。在完成经典的TCP三次握手后，连接器就开始认证你的身份，这个时候用的就是你的用户名和密码。
- 如果密码正确，连接器会到权限表查询你的权限信息。之后这个连接里面的权限判断逻辑，都将依赖此时读到的权限。也就说如果你在这次连接中修改了当前用户的权限，也需要在下次连接才会生效。
- 如果用户名或者密码错误，或收到一个"ERROR 1045 (28000): Access denied for user 'zero'@'localhost' (using password: YES)"错误，然后客户端程序结束执行。

连接成功后如果后续的动作，这个连接就处于空闲状态。你可以通过show processlist 命令查看。其中Command显示为Sleep表示当前有个空闲连接。

![processList](http://localhost:1313/images/processlist.jpg)




如果空闲时间太长，连接器就会自动将它断开。这个参数由wait_timeout控制，默认值是8小时。

如果在断开连接后，客户端再次发送请求的话，就会收到一个错误提醒：Lost connection to MySQL server during query。

数据库里面，长连接就是建立连接之后如果一直有请求就会一直使用这个连接。短连接只会执行很少的几次操作就会断开连接，由于建立连接是一个比较复杂的过程，所以一般建议使用长连接。

长连接会有一个问题，就是如果你当前系统有大量的长连接，有时候会发现MySQL占用的内存涨的特别快。这是因为MySQL在执行过程中临时使用的内存是管理在连接对象里面的。这些资源只有在连接断开会才会释放掉，所以如果长连接累积下来，可能导致内存占用过大，被系统强行抹杀掉（OOM），现象就是MySQL重启了。可以使用下面两个方案

- 定期断开长连接。使用一段时间，或者在程序中判断执行过一个占用内存的大查询后，断开连接，之后查询再重新连接。
- MySQL5.7版本后可以在每次大操作后通过执行mysql_reset_connection来重新初始化连接资源。这个过程不需要重连和重新做权限验证，但会将连接恢复到刚刚创建完时的状态。

##### 查询缓存

MySQL的缓存是以查询语句为key，结果为value存储的。如果你开启了缓存，那么在每次查询后mysql会将结果缓存起来。当再次查询时会先查询缓存，命中就返回。

但是，缓存弊大于利，建议不开启。MySQL8.0开始移除了整个缓存模块。为什么这么说呢？是因为每次对表的操作都会导致缓存失效。比如inset、update、delete、truncate、alter table、drop table、drop database等。所以除非你有一个静态表，很久才会更新一次，那么使用缓存并不能提高你的性能，反而会降低性能。

MySQL也提供了按需使用的功能呢。你可以将query_cache_type设置为DEMAND，这样对于默认的查询语句是不会走缓存的，如果你想使用缓存可以这样：

```sql
select SQL_CACHE * from T where id = 10;
```



##### 分析器

如果没有命中缓存就要开始执行语句了。第一步是词法分析，因为你输入的是几个字符串加一些空格，mySQL需要知道你想干什么，以select * from T where id =1为例，从关键字select知道这是一条查询语句，将T识别为表T，将id识别为字段id。如果当前表或者字段不存在，分析器这时会抛出ERROR 1146 (42S02): Table 'test.t2' doesn't exist/ERROR 1054 (42S22): Unknown column 'k'错误。

词法分析后是语法分析，MySQL会检查当前的语句是否符合MySQL语法规则。比如如果你select少写前面的s会得到以下的错误提示：ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'elect * from T' at line 1; 需要特点关注use near后面紧跟的地方。

##### 优化器

优化的工作是当查询的表中存在多个索引时，决定使用哪个索引。或者当你的语句是多表关联时决定表的连接顺序。

```sql
select * from t1 join t2 where t1.id = t2.id;
```

- 这句sql可以表示为从t1中取出id然后和t2表中的id进行比较；
- 也可以表示为从t2中取出id然后和t1表中的id进行比较；

这两句话的逻辑是一样的，但是执行的效率可能不同，这时候使用哪个方案就是优化器的工作。

##### 执行器

通过分析器MySQL知道了你要做什么，优化器决定了怎么做之后，就进行执行器阶段开始执行语句。

开始执行之前需要先判断你对这个表T有没有执行查询的权限，如果没有，就会返回没有权限的错误。如下所示（在工程上如果命中缓存，会在查询缓存放回结果的时候，做权限验证。查询也会在优化器之前调用precheck验证权限）

```sql
mysql> select * from T where ID=10;

ERROR 1142 (42000): SELECT command denied to user 'b'@'localhost' for table 'T'

```

如果有权限，就打开表继续。打开表的时候，会根据表的引擎的定义，去使用这个引擎提供的接口。

比如以我们select * from T where id = 1在没有索引的情况下，流程为：

1. 调用InnoDB引擎接口取这个表的第一行，判断id是不是等于1，如果是就将数据放入结果集中。
2. 取“下一行”，重复以上的逻辑。直到最后一行。
3. 执行器将上述遍历过程中所有满足条件的行组成记录集作为结果集返回给客户端。

使用索引的情况也差不多，第一次调用的是“取满足条件的第一行”这个接口，之后循环取“满足条件的下一行”这个接口，这些接口都是引擎定义好的。

当我们开启慢查询日志时，显示的rows_examined代表了这次查询扫描了多少行。这个值就是在执行器每次调用引擎接口获取数据行的时候累加的。在有些场景下，执行器调用一次，在引起内部则扫描了多行，**因此引擎扫描行数跟rows_examined并不是完全相同的。**

```sql
macdeMacBook-Pro-3:data mac$ more macdeMacBook-Pro-3-slow.log
/usr/local/mysql/bin/mysqld, Version: 5.7.24 (MySQL Community Server (GPL)). started with:
Tcp port: 3306  Unix socket: /tmp/mysql.sock
Time                 Id Command    Argument
# Time: 2019-01-07T07:46:54.491815Z
# User@Host: root[root] @ localhost []  Id:     8
# Query_time: 10.003401  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 0
SET timestamp=1546847214;
select sleep(10);
```