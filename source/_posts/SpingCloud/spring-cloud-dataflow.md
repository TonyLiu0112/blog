---
title: Spring Cloud Data Flow -- Stream实战
tags: 
    - Spring Dataflow
    - Stream
categories:
    - Spring Cloud Data Flow
---

## 简介
Spring Cloud Data Flow Stream是Spring Cloud Data Flow的一种编程模型, 用于处理无限循环的实时数据流处理(类似Storm流式计算), Stream对组件对象的抽象类似Apche Flume, 提供了3种基本的组件

* **Source**: 
  消费各种事件的应用程序
* **Processor**: 
  从Source消费数据, 对数据做处理, 发射处理好的数据到下一个应用
* **Sink**: 
  从Source或者Processor消费数据, 将数据写入期望的存储层

## 实战

使用之前, 需要安装官方提供的Data Flow Server, 这个组件的作用是用来构建整个Stream计算的Topology图; 同时, 还需要使用到消息队列, RabbitMQ或者Kafka, 官方提供了一个Docker-compose版本的文件来构建Data Flow Server

### Docker构建Data Flow Server

`docker-compose.yml`

```yml
version: '3'

services:
  kafka:
    image: wurstmeister/kafka:0.10.1.0
    expose:
      - "9092"
    environment:
      - KAFKA_ADVERTISED_PORT=9092
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
    depends_on:
      - zookeeper
  zookeeper:
    image: wurstmeister/zookeeper
    expose:
      - "2181"
    environment:
      - KAFKA_ADVERTISED_HOST_NAME=zookeeper
  dataflow-server:
    image: springcloud/spring-cloud-dataflow-server-local
    container_name: dataflow-server
    ports:
      - "9393:9393"
    environment:
      - spring.cloud.dataflow.applicationProperties.stream.spring.cloud.stream.kafka.binder.brokers=kafka:9092
      - spring.cloud.dataflow.applicationProperties.stream.spring.cloud.stream.kafka.binder.zkNodes=zookeeper:2181
    depends_on:
      - kafka
  app-import:
    image: alpine:3.7
    depends_on:
      - dataflow-server
    command: >
      /bin/sh -c "
        while ! nc -z dataflow-server 9393;
        do
          sleep 1;
        done;
        # wget -qO- 'http://dataflow-server:9393/apps' --post-data='uri=https://bit.ly/Celsius-SR1-stream-applications-kafka-10-maven&force=true';
        wget -qO- 'http://dataflow-server:9393/apps' --post-data='uri=http://repo.spring.io/libs-release/org/springframework/cloud/stream/app/spring-cloud-stream-app-descriptor/Celsius.SR1/spring-cloud-stream-app-descriptor-Celsius.SR1.stream-apps-kafka-10-maven';
        echo 'Stream apps imported'
        # wget -qO- 'http://dataflow-server:9393/apps'  --post-data='uri=https://bit.ly/Clark-GA-task-applications-maven&force=true';
        wget -qO- 'http://dataflow-server:9393/apps'  --post-data='uri=http://repo.spring.io/libs-snapshot/org/springframework/cloud/task/app/spring-cloud-task-app-descriptor/Clark.RELEASE/spring-cloud-task-app-descriptor-Clark.RELEASE.task-apps-maven';
        echo 'Task apps imported'"
```

在当前目录使用`docker-compose up` 启动, 访问`http://localhost:9393`可以看到Server的界面.

### 构建Source组件
新建spring boot项目, 并集成Spring Cloud

```java
@EnableBinding(Source.class)
@SpringBootApplication
public class SourceApplication {

    private final Logger logger = LoggerFactory.getLogger(SourceApplication.class);
    private int count = 0;

    public static void main(String[] args) {
        SpringApplication.run(SourceApplication.class, args);
    }

    @InboundChannelAdapter(
            value = Source.OUTPUT,
            poller = @Poller(maxMessagesPerPoll = "1", fixedDelay = "10000")
    )
    public Long timeSource() {
        logger.info("Send event: {}", ++count);
        return new Date().getTime();
    }
}
```
`@InboundChannelAdapter`用来定义发送事件的目的地和频率, 这里代表将事件输出到Source组件的输出, 每10秒钟发送一次.

### 构建Processor组件

Processor组件用户消费Source发出的事件

```java
@EnableBinding(Processor.class)
@SpringBootApplication
public class ProcessorApplication {

    private final Logger logger = LoggerFactory.getLogger(ProcessorApplication.class);

    private int count = 0;

    public static void main(String[] args) {
        SpringApplication.run(ProcessorApplication.class, args);
    }

    @Transformer(inputChannel = Processor.INPUT, outputChannel = Processor.OUTPUT)
    public Object transform(Long timestamp) {
        logger.info("processor input: {}", ++count);
        DateFormat dateFormat = new SimpleDateFormat("yyyy/MM/dd hh:mm:ss");
        return dateFormat.format(timestamp);
    }
}
```

使用`@Transformer`注解, 将Source发送的时间进行格式化处理后, 发送到Processor的输出

### 构建Sink组件

Sink组件用户消费Processor发出的数据

```java
@SpringBootApplication
@EnableBinding(Sink.class)
public class SinkApplication {

    private final Logger logger = LoggerFactory.getLogger(SinkApplication.class);

    public static void main(String[] args) {
        SpringApplication.run(SinkApplication.class, args);
    }

    @StreamListener(Sink.INPUT)
    public void loggerSink(String date) {
        logger.info("Received: " + date);
    }

}
```

至此, 基本用的组件构建完毕, 整体逻辑是: `Source`每10秒钟发送一个时间毫秒数, `Processor`格式化毫秒数成字符串, `Sink`输出.

### 注册自定义组件到Data Flow Server

在UI界面中的Apps菜单中, 将这三个组件的jar包注册到server中, 服务的注册可以是一个maven的uri说明, 也可以是一个本地的文件, 这里使用本地文件方式注册, 在注册页面的URI中分别输入:

```
file:///usr/local/tony/stream-source-1.0-SNAPSHOT.jar

file:///usr/local/tony/stream-processor-1.0-SNAPSHOT.jar

file:///usr/local/tony/stream-sink-1.0-SNAPSHOT.jar
```

完成后, 可以在Apps中搜索到刚刚上传的App

### 构建Stream

在UI界面的Streams菜单中, 选择 `Create Stream`按钮, 使用Linux的管道符语言来构建topology图

```
mySource | myProcessor | mySink
```

这里的名称需要和你注册的app的名称一致

点击CREATE STREAM按钮, 并勾选住`deploying`按钮, 完成Stream构建; 当这个流的状态变为deployed之后, 可以到Runtime菜单中查看每个节点的运行的日志路径, 比如这里使用docker查看:

```log
docker exec -it dataflow-server tail -f /tmp/spring-cloud-deployer-2848555245698758788/ticktock-1522639285733/ticktock.log/stdout_0.log
```

可以看到日志在持续输出时间.
	