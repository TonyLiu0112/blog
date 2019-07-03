---
title: MySQL性能优化 -- 数据库锁
tags: 
    - mysql
categories: 
    - mysql
---

### 引言

mysql对于锁的使用，和大多数锁的定义一样（S锁和X锁），一方面在于保证数据的一致和完整性，另一方面用户保证数据在并发读写的场景下的有序性，一个高效的锁模型直接影响访问的性能

### 锁

锁在不用语言、组件的实现中，主要遵循两种标准：**读锁**（或者称为共享锁、S锁）、**写锁**（或者称为排他锁、X锁），读锁特点是**可以对同一个资源加多个读锁，且有读锁存在的时候，写锁是阻塞的**，而写锁的主要特点是，**当某个线程访问资源开启了一个写锁，这个资源不允许添加任何其他的锁**

### MySQL的锁的原理

mysql中，不同的存储引擎对锁的实现是有差异的，这里以InnoDB和MyISAM为例，mysql实现了3种类型的锁定机制：**行级锁**、**表级锁**、**页锁**，每个级别的锁主要在于粒度上的区别，下面来分析一下各个锁的区别

#### 表级锁

表级锁，顾名思义，就是锁的粒度是以整张表为基础的，粒度在整个mysql的锁中最大，因为粒度大，所以整个表锁的实现相当简单，资源的消耗最小，获取和释放锁的速度很快，但相对的带来的弊端就是，在高并发的场景下，非常容易出现竞争，所以性能表现的很差。

表锁的实现在内部分为读锁定和写锁定，主要通过4个队列来实现这两种的锁定

```shell
current read-lock queue   // 当前读锁队列
pending read-lock queue   // 等待读锁队列
current wirte-lock queue  // 当前写锁队列
pending write-lock queue  // 等待写锁队列
```

4个队列按照时间顺序存放锁的信息

##### 读锁定

当申请获取资源的读锁定时需要满足两个条件：

1. 当前申请的资源没有一个写锁定
2. 当前申请的资源对应的`pending write-lock queue`没有一个等待写锁定的请求满足条件后，会立即在`current read-lock queue`中存入一个读锁定的信息，反之则进入`pending read-lock queue`

##### 写锁定

当申请获取资源的读锁定时，会先判断`current wirte-lock queue`是否存在值，如果不存在，则直接写入`current wirte-lock queue`，否在再判断`pending write-lock queue`，如果则写入此队列，否则继续判断`current read-lock queue`与`pending read-lock queue`，原理逻辑同上。*这里与S锁和X锁的规范实现原理一致*

#### 行级锁

行级锁最大的特点就是锁定的颗粒度很小，由于锁定的资源最小，所以发生竞争的概率也最小，能过给与程序最大的能力处理并发的场景，提供应用系统的整体性能。MyISAM引擎不支持该锁。

行级锁主要由存储引擎自己实现，InnoDB的实现也是基于共享锁和排他锁的原理，为了让共享锁和排他锁共存，InnoDB实现了另外两种类型的锁：意向共享锁、意向排他锁。意向锁的主要作用是，当事物访问一个资源的时候，如果该资源（行记录）存在一个排他锁，这时候可以对该表加一个意向锁。所以，InnoDB对锁的类型总共可以分为4种类型的锁：

* 共享锁（S）
* 排他锁（X）
* 意向共享锁（IS）
* 意向排他锁（IX）

MySQL在**Repeatable Read**隔离级别下，通过使用间隙锁+行锁的方式（Next-key lock）来防止幻读，对于间隙锁，是指对一段范围内的数据加锁，即使这个数据不存在，间隙锁的实现是基于当前记录对应的第一个索引键之前和最后一个索引键之后的区域，例如：

| id   | name | age  |
| ---- | ---- | ---- |
| 1    | T2   | 1    |
| 2    | T3   | 2    |
| 5    | T6   | 5    |
| 9    | T9   | 9    |
| 10   | T10  | 10   |

```mysql
select * from t_user where age = 5 for update
```

如果事务A对上述表执行如下sql，此时会存在一个`(2, 9]`的间隙锁，事务B如果想在范围内做新增，修改，删除，加锁等操作，都将阻塞，例如

```mysql
insert into t_user (name, age) values ('T3', 3);
insert into t_user (name, age) values ('T6', 6);
```

如果where后面是一个主键、或者是一个唯一索引，那么将不会开启这个间隙锁，间隙锁唯一的作用就是防止其他事务的插入引起的幻读问题

#### 页锁

页锁是mysql中一种特殊的锁，介于行锁和表锁之间，暂不讨论。

### 优化锁的建议

不管是MySQL还是Java，对锁的优化都有一个大体的思路，优化策略都是围绕这个思路来进行的，锁优化的步骤一般有，减小锁的粒度

，缩短锁的时间，分离锁，锁的消除。

#### 表锁优化

1. 缩短锁的时间

   缩短锁的时间并不是一个简单的操作，缩短锁的时间唯一的办法就是减少持锁后的处理时间，能做的就是减少查询的时间

   * 尽量减少大的复杂的query，将复杂Query拆分成几个简单的sql执行
   * 尽可能的让执行走高效的索引，提高查询速度

2. 分离锁

   将一个大锁拆分成几个小的锁并行操作

#### 行锁优化

1. 尽可能的让所有的检索都通过索引来完成，从而避免InnoDB因无法通过索引键加锁而升级为表锁定
2. 合理设计索引，保证锁的准确性，尽可能的缩小锁定的范围
3. 减少基于范围的数据检索过滤条件，避免范围索引产生的间隙锁带来的负面影响



### MySQL锁的情况查询

MySQL中提供了查看当前的锁的情况的语句

```mysql
-- 查看表锁
mysql> show status like 'table%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Table_locks_immediate      | 9981  |
| Table_locks_waited         | 0     |
| Table_open_cache_hits      | 0     |
| Table_open_cache_misses    | 0     |
| Table_open_cache_overflows | 0     |
+----------------------------+-------+
5 rows in set (0.00 sec)
```

* Table_locks_immediate：产生表级锁的次数
* Table_locks_waited：出现表级锁竞争，等待的次数

```mysql
mysql> show status like 'innodb_row_lock%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| Innodb_row_lock_current_waits | 0     |
| Innodb_row_lock_time          | 47190 |
| Innodb_row_lock_time_avg      | 1474  |
| Innodb_row_lock_time_max      | 13754 |
| Innodb_row_lock_waits         | 32    |
+-------------------------------+-------+
5 rows in set (0.00 sec)
```

* Innodb_row_lock_current_waits：当前等待锁的数量
* Innodb_row_lock_time：mysql启动到当前，锁总共耗费的时长
* Innodb_row_lock_time_avg：每次锁平均耗时
* Innodb_row_lock_time_max：从MySQL启动到现在，最长一次等待的时长
* Innodb_row_lock_waits：从MySQL启动到现在，总共产生锁的次数

当总等待等待次数（Innodb_row_lock_waits）很高，且总等待时长（Innodb_row_lock_time）也不低时，需要分析排查导致这个的原因了。

