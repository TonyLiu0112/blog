---
title: Redis数据持久化
tags: 
    - redis
    - NoSQL
    - 缓存
categories:
    - redis
---
Redis对数据持久化一般使用三种方式

snapshot、AOF、Replication

## Snapshot 
使用内存快照的方式将数据在某个时间点写入到磁盘.
快照存储的数据为：<font color="red">上一次快照结束时间点</font> ～ <font>本次快照开始时间点</font>

### 触发时间

* 任何redis客户端都可以使用`BGSAVE`命令来触发一个snapshot.(注意：当执行`BGSAVE`命令的时候，redis会做一个fork进程动作，fork出来的子进程将处理snapshot到磁盘, fork会消耗和原来同等大小的内存, 而原始进程将继续接受请求的命令)
* 一个客户端可以通过发起一个`save`命令来开始做snapshot,当触发这个命令之后，在snapshot完成之前，会拒绝处理任何命令（和`BGSAVE`的区别）, 这个不常用，通常只是在没有更多的内存执行bgsave的时候(因为fork会消耗和原来同等大小的内存, 进程隔离)会谨慎使用
* 通过在配置文件中配置`save 60 10000`的配置项后，如果在上一次snapshot成功之后60秒内有超过10000次的写操作，redis会自动触发`BGSAVE`模式的snapshot操作.当配置文件配置了多个`save`配置项的时候，任何时候只要匹配到1个，就会触发一个`BGSAVE`操作(并存的关系，满足任何一个条件就触发)
* 当redis接收到一个`shutdown`命令或者接收到一个标准TERM信号,将会触发一个`save`的snapshot操作
* 当redis服务器连接到其他服务器并确保一个`SYNC`命令开始执行复制数据，那么这个master服务器将开始执行一个`BGSAVE`的操作

### 场景

假设, 2:35分完成了第一次内存快照，在3:06分又开始第二次内存快照了，直到3:08分还未结束快照，3:06~3:08期间又写入了30个key到数据库, 3:08分某一刻数据库宕机.

* 如果在3:08分发生了死机或者系统崩溃导致redis重启，那么，在2:35分~3:08分之间的数据将全部丢失。
* 如果3:08分数据快照完成了，那么3:06~3:08之间的30个key将丢失。

<font color="red">快照期间, 请求量级越大, 丢失的数据越多</font>

## AOF

append only file, 仅仅追加文件，他的工作原理是，使用复制来将发生的命令写入到磁盘.

### 原理

先将数据写到缓存，当数据都在缓存里面后，操作系统会在将来的时间点上，把这些数据写到磁盘上（异步写）
>在写的过程中如果访问磁盘上这些数据，将会访问不到（因为操作系统还没写完），如果我们让操作系统执行`sync`指令，那么操作会在数据全部写完前一直阻塞.

><font color="red">**如果使用SSD硬盘来处理aways类型的配置，会大大降低SSD磁盘的使用寿命**</font>

### 配置项

`everysec`

### BGREWRITEAOF

AOF已经能很好的帮我们解决数据持久化的问题，能最大程度的减少数据丢失，保证最多丢失1秒的数据，对于AOF来说，因为是文件追加的方式，所以随着数据量的变大，Redis每次启动初始化数据所花费的时间越来越多，当处理大量AOF，Redis的可能需要很长的时间来启动。为了解决这个问题，通常使用`BGREWRITEAOF`，`BGREWRITEAOF`的工作方式类似于`bgsave` `fork`进程。

## 自我复制

在集群环境下，Redis采用Master-Slave模式保证数据持久化，对于集群环境，redis推荐最少配置为3M-3S模式，3个Master用来接收数据，并将数据异步同步到Slave上存储.
>Redis集群中，Redis将65000多个slot按照算法分配到3个Master上，每次请求的Key通过算法可以得到一个固定的slot编号，根据slot编号来决定使用那个master，原理类似一致性Hash环.

当一个slave和一个master建立连接后，会发生如下事情

| 步骤        |   Master操作    | slave操作  |
| ---------  |:---------------:| ---------:|
| 1 | 等待命令 | 连接到Master, 发起`SYNC`命令 |
| 2 | 开始BGSAVE操作; 保留BGSAVE之后发送的所有写命令的备份log | 提供旧数据(如果有)或者返回错误到命令(取决于配置) |
| 3 | 完成BGSAVE; 开始向slave发送snapshot; 继续持有写命令的备份log | 舍弃所有旧数据（如果有）; 在收到snapshot时开始加载它 |
| 4 | 完成向slave发送snapshot操作，开始发送持有的写命令的备份log | 完成解析snaptshot，开始正常响应命令 |
| 5 | 完成发送备份log, 开始直接将写入命令发送到slave | 完成备份log的写入,继续执行其他命令 |







