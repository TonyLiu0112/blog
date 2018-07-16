---
title: Consul集群安装
tags: 
    - Consul
categories: 
    - Consul
---

# Consul集群

## 架构简介

![](./consul-arch.png)

上图是官网提供的一个consul集群架构，其中`Server`是consul集群的服务节点，它保证consul集群的高可用，各个`Server`之间使用选举算法产生Leader统一管理，leader负责处理所有的查询和事务， 当一个非leader得server收到一个RPC请求时，它将请求转发给集群leader。集群间使用`GOSSIP`协议通信，`Server`之间使用raft算法保证强一致性(实现了CAP中的CP, Eureka实现的是AP). `Client`代表一个consul客户端，客户端是无状态的，可以有任意个，`Server`和`Client`都属于一个consul agent。

### 术语描述

* Agent

    agent是一直运行在Consul集群中每个成员上的守护进程。通过运行consul agent来启动。agent可以运行在client或者server模式。指定节点作为client或者server是非常简单的，除非有其他agent实例。所有的agent都能运行DNS或者HTTP接口，并负责运行时检查和保持服务同步.

* Client

    Client是一个转发所有RPC到server的代理。这个client是相对无状态的。client唯一执行的后台活动是加入LAN gossip池。这有一个最低的资源开销并且仅消耗少量的网络带宽。

* Server

    server是一个有一组扩展功能的代理，这些功能包括参与Raft选举，维护集群状态，响应RPC查询，与其他数据中心交互WAN gossip和转发查询给leader或者远程数据中心。

* DataCenter

    数据中心，存储写入consul的数据。

* Consensus

    一致性，使用Consensus来表明就leader选举和事务的顺序达成一致。为了以容错方式达成一致，一般有超过半数一致则可以认为整体一致。Consul使用Raft实现一致性，进行leader选举，在consul中的使用bootstrap时，可以进行自选，其他server加入进来后bootstrap就可以取消。

* Gossip

    Consul建立在Serf的基础之上，它提供了一个用于多播目的的完整的gossip协议。Serf提供成员关系，故障检测和事件广播。Serf是去中心化的服务发现和编制的解决方案，节点失败侦测与发现，具有容错、轻量、高可用的特点。

* LAN Gossip

    它包含所有位于同一个局域网或者数据中心的所有节点。

* WAN Gossip

    它只包含Server。这些server主要分布在不同的数据中心并且通常通过因特网或者广域网通信。

## 集群构建

### 环境说明

| 主机名 | 主机IP | Consul类型 |
| ----- | ----- | ----- |
| consul-server01 | 10.211.55.13 | server |
| consul-server02 | 10.211.55.14 | server |
| consul-server03 | 10.211.55.15 | server |
| consul-client01 | 10.211.55.16 | client |

### 创建目录

每一台机器上执行

```bash
mkdir -p /usr/local/tools/consul/data
```

```bash
mkdir -p /usr/local/tools/consul/config
```

### 启动Server主节点

`consul-server01`机器

```bash
./consul agent -server -bootstrap \
    -data-dir=/usr/local/tools/consul/data \
    -config-dir=/usr/local/tools/consul/config \
    -client=10.211.55.13 \
    -bind=10.211.55.13 \
    -node=consul-server01 \
    -ui &
```

### 启动Server子节点

`consul-server02`机器

```bash
./consul agent -server -syslog \
    -ui \
    -data-dir=/usr/local/tools/consul/data \
    -config-dir=/usr/local/tools/consul/config \
    -client=10.211.55.14 \
    -bind=10.211.55.14 \
    -node=consul-server02 \
    -join=10.211.55.13 &
```
`consul-server03`机器

```bash
./consul agent -server -syslog \
    -ui \
    -data-dir=/usr/local/tools/consul/data \
    -config-dir=/usr/local/tools/consul/config \
    -client=10.211.55.15 \
    -bind=10.211.55.15 \
    -node=consul-server03 \
    -join=10.211.55.13 &
```

### 启动Client节点

`consul-client01`机器

```bash
./consul agent -syslog \
    -data-dir=/usr/local/tools/consul/data \
    -client=10.211.55.16 \
    -bind=10.211.55.16 \
    -node=consul-client01 \
    -join=10.211.55.13 &
```

### 检查集群状态

```bash
./consul members --http-addr 10.211.55.13:8500
```

返回如下集群状态

![](./consul-members.jpeg)

consul提供可访问页面查看信息: `http://10.211.55.13:8500`

## 注册应用

consul集群的使用方式不同于普通集群的方式，当使用zk集群的时候，往往会将所有的zk机器信息配置到客户端中进行连接，当其中一台宕机后，自动切换到存活的leader节点，而consul是使用client的方式连接集群，client是无状态、可无限扩展的一个轻量级的客户端，当有5个应用需要使用consul，可以让每个应用启动自己的client来连接到consul集群。

以spring boot项目为例，按照上述配置的结构，我们使用`consul-client01`机器作为测试项目的连接入口。

```yml
server:
  port: 8080
  servlet:
    context-path: /demo
spring:
  application:
    name: demo
  cloud:
    consul:
      host: 10.211.55.16
      port: 8500
      discovery:
        health-check-path: /${spring.application.name}/actuator/health
        instance-id: ${spring.application.name}
        serviceName: ${spring.application.name}
        port: 8080
logging:
  level:
    root: info
```

访问ui页面查看注册信息

![](./consul-ui.jpeg)