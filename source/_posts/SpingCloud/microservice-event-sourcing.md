---
title: Event Sourcing分布式事务处理
tags: 
    - 分布式事务
    - Event Sourcing
categories:
    - Spring Cloud
---

## 微服务框架的问题

### 分布式数据管理

开发微服务架构应用的时候, 服务通常是松耦合的, 每一个服务都有自己私有的数据库, 有的服务使用的是NoSQL, 有的服务使用的是RDB, 这样就没有办法保证ACID。因此, 保证数据的一致性是一个巨大的挑战。

### 异步的微服务架构

#### 使用sagas保证数据一致性

sagas是一个微服务架构的事务模型, 每个微服务保持自己的事务ACID, 完成后触发下一个事务的消息驱动。
![](./eventdrivencreditcheck.png)

#### 使用CQRS(命令查询职责分离模式)

CQRS视图是实现跨微服务架构中的服务的查询的一种方式。 CQRS视图是来自一个或多个服务的数据副本，它针对特定的一组查询进行了优化。 维护视图的服务通过订阅域事件来完成。 每当一个服务，更新它的数据，它就会发布一个事件。
![](./maintainingviews.png)

### 举个例子

假设有如下两个服务: 
1. `Order-Service`
2. `Customer-Service`

`Order-Service`用于订单的创建、审核、取消等行为, `Customer-Service`记录用户可用的信用卡额度，每产生一笔订单, 用户的信用卡额度减少相应的金额。

在微服务架构中, 当`Order-Service`发起了一个创建订单的动作, 插入订单表数据后，同时发送创建的event到message broker。问题在于, 如何保证插入订单表数据和发送event的原子性？
![](./atomicity.png)

解决办法之一是使用基于事件驱动架构(Event-Driven architecture)的Event Sourcing(事件源溯)

## Event Sourcing

### 是什么

Event Sourcing是一个保证更新状态和发布事件的原子性的很好的方法, 它以事件为中心的方法来持久化业务对象。业务对象被持久化成一系列按顺序的状态改变的事件集合（数据库事件表）。每当业务的状态发生改变后，一个新的事件将追加到这个集合。由于每次只是追加一个事件, 所以这是一个原子性的操作。

### 如何工作

以`Order-Service`为例, 传统上，每一个订单都映射到订单表的一条订单记录，但是，当使用Event Sourcing时，订单服务通过持久化其状态变更事件来存储订单：创建，批准，发货，取消。 每个事件将包含足够的数据来重建订单的状态。

![](./storingevents.png)

所有的事件都存储在Event Store中, 它提供了一个API, 让各个微服务订阅事件。Event Store是整个微服务框架的中坚架构。

## Eventuate框架

Eventuate是实现Event Sourcing的一个框架: 

![](eventuateparts.png)

后续描述Eventuate框架的使用。


















