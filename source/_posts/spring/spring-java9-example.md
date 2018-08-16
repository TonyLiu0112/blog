# 使用Java9模块化构建spring boot项目



## Java9 模块化



### 简介

***模块化（modularization）***是指将系统分解成独立且相互连接的模块的行为。***模块（module）***是包含代码的可识别工件，使用了元数据来描述模块及其他模块的关系。在理想情况下，这些工件从编译到运行时都是可识别的。一个应用程序由多个模块协作组成。

模块化需遵循3个核心原则:

* 强封装性
* 定义良好的接口
* 显示依赖

Java9中，模块化信息的定义在java包下定义名为`module-info.java`的文件来描述一个模块。

### 基础语法

### module

声明一个模块，`module-info.java`文件唯一

```java
module mymodule {
}
```

### requires

声明一个依赖关系，说明当前模块需要依赖声明的模块

```java
module mymodule {
    requires java.sql;
    // transitive 代表可见性的传递
    requires transitive java.sql;
}
```

### exports

声明当前模块暴露的可用接口，外部模块仅可调用此声明的接口

```java
module mymodule {
    exports com.tony666.examples.spring.java9.getaway.client;
}
```

### open

开放一个模块下所有的包

```java
open module mymodule {
    
}
```

### opens 

开放模块下指定的包

```java
module mymodule {
    opens com.tony666.examples.spring.java9.getaway.client;
}
```

> Node：open 和 opens比较特殊，在jdk9之前，任意包中的类都能通过反射获得类的实例，在jdk9中，由于模块具有强封装性，即无法对模块内部未导出的类进行反射操作，在很多框架（spring）中往往不能满足需求，使用open和opens关键字能将需要反射的包放开，这是一种***深度反射***

### provides with

声明一个接口的实现，允许消费模块使用这个声明的服务

```java
module mymodule {
    provides com.tony666.demo.MyService with com.tony666.demo.MyServiceImpl;
}
```

### uses

`uses`有两种使用方式：

* 消费模块显示声明使用某个服务

```java
module mymodule {
    uses com.tony666.demo.MyService;
}
```

* 服务模块隐式的定义某个服务的加载

```java
public interface CarService {

    Car getCar();

    static ServiceLoader<CarService> getServices() {
        return ServiceLoader.load(CarService.class);
    }

}
```

```java
module mymodule {
    uses CarService;
}
```

此时客户端无需声明`uses`，可直接通过`CarService.getServices()`方法获取所有实现的服务



Java9还有很多深层的特性，参考: **《Java9模块化开发 核心原则与实践》**



## Spring boot 2.x 简介

spring boot 2.x使用spring5的新特性，支持反应式（Reactor Flux）编程，提供了异步非阻塞 IO 的响应式 Stream，且明确支持java9的自动模块特性。这里实例基于2.x版本实现。

参考: [spring2.x手册](https://docs.spring.io/spring-boot/docs/2.1.0.M1/reference/htmlsingle/)

## 实战

使用java9 + spring boot 2.x实现一个简单的支付模块，支付模块包含如下功能点:

> 模块化和微服务概念类型，但并不冲突，每个微服务的内部通常使用模块化来实现，它们是一种互补的关系

- 保证金支付
- 积分支付
- 积分退款
- 支付前置校验
- 退款前置校验
- 外部支付服务调用
- 交易状态一致性回调处理

### 技术栈

* spring boot: 2.1.0.M1
* jdk: 9
* maven: 3.x

### 项目结构

```bash
spring-java9-examples		# ROOT目录
├── gateway-client/ 		# 外部调用服务模块
├── gateway-thirdparty/		# 第三方服务模块
├── trade-basic/		    # 交易基础模块
├── trade-callback/			# 交易状态一致性回调模块
├── trade-channels/			# 交易渠道模块(多个渠道)
├── trade-specification/	# 交易规则模块
└── trade-state/			# 交易状态模块
```

#### gateway-client

客户端网关模块，聚合内部模块，暴露http接口到外部模块

`module-info.java`

```java
module gateway.client {
    // 声明需要的支付退款模块
    requires payment.integration;
    requires refund.integration;
	// 声明需要的spring模块
    requires java.sql;
    requires spring.core;
    requires spring.beans;
    requires spring.context;
    requires spring.aop;
    requires spring.web;
    requires spring.webmvc;
    requires spring.expression;
    requires spring.boot;
    requires spring.boot.autoconfigure;
    requires spring.boot.starter;
    requires spring.boot.starter.actuator;
    requires spring.boot.actuator;
    requires spring.boot.actuator.autoconfigure;
    requires spring.boot.starter.web;
	// 放开包路径，便于spring反射使用
    opens com.tony666.examples.spring.java9.getaway.client;
    opens com.tony666.examples.spring.java9.getaway.client.controller;
}
```

`Controller.java`

```java
@RestController
@RequestMapping("trade")
public class IntegrationController {
	// 注入支付、退款接口
    private final Payment payment;
    private final Refund refund;

    public IntegrationController(Payment payment, Refund refund) {
        this.payment = payment;
        this.refund = refund;
    }

    @GetMapping("pay")
    public ResponseEntity<Object> pay() {
        Optional<PaymentRes> paymentRes = payment.payment(new PaymentReq());
        return new ResponseEntity<>(paymentRes, HttpStatus.OK);
    }

    @GetMapping("refund")
    public ResponseEntity<Object> refund() {
        Optional<RefundRes> refundRes = refund.refund(new RefundReq());
        return new ResponseEntity<>(refundRes, HttpStatus.OK);
    }

}
```

> spring5.0中，重构了controller路径映射的加载方式，在spring4中，启动后能明显在控制台看到`RequestMappingHandlerMapping`输出自定义的Mapped映射路径，例如:
>
> ```bash
> INFO  RequestMappingHandlerMapping - Mapped "{[/liveroom/second/comments],methods[GET]}" onto public methodInfo
> ```
>
> 而spring5中，默认info日志级别下不输出此信息，需要将日志级别调整为`trace`，因为：
>
> ```java
> public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
> 	/**
> 	 * Look for handler methods in a handler.
> 	 * @param handler the bean name of a handler or a handler instance
> 	 */
> 	protected void detectHandlerMethods(final Object handler) {
>         // ...
> 		if (logger.isTraceEnabled()) {
> 				logger.trace("Mapped " + methods.size() + " handler method(s) for " + userType + ": " + methods);
> 			}
>         // ...
> 	}
> }
> ```

#### gateway-thirdparty

第三方系统集成闸道，集合所有外部系统的出口网关

#### trade-basic

交易基础模块，抽象部分交易接口和交易逻辑

```java
public interface Payment {

    Optional<PaymentRes> payment(PaymentReq paymentReq);

}
```

```java
public abstract class AbstractPayment implements Payment {
	// 支付规则器集合
    private List<Specification<PaymentState>> paymentSpecification = new ArrayList<>();

    public AbstractPayment() {
    }

    @SuppressWarnings("ClassEscapesDefinedScope")
    public AbstractPayment(List<Specification<PaymentState>> paymentSpecification) {
        this.paymentSpecification = paymentSpecification;
    }

    @Override
    public Optional<PaymentRes> payment(PaymentReq paymentReq) {
        Optional<PaymentRes> paymentRes = doCheck(paymentReq);
        if (paymentRes.isPresent())
            return paymentRes;
        return doPayment(paymentReq);
    }

    private Optional<PaymentRes> doCheck(PaymentReq paymentReq) {
        PaymentState checkRes = paymentSpecification.stream().collect(new SpecificationCollector<>(paymentReq));
        return checkRes == null ? Optional.empty() : Optional.of(new PaymentRes(checkRes));
    }

    public abstract Optional<PaymentRes> doPayment(PaymentReq paymentReq);

}
```

```java
module trade.basic {
    // pay
    exports com.tony666.examples.spring.java9.refund.facade.payment;
    exports com.tony666.examples.spring.java9.refund.facade.payment.vo;
    // refund
    exports com.tony666.examples.spring.java9.refund.facade.refund;
    exports com.tony666.examples.spring.java9.refund.facade.refund.vo;
	// 依赖规则模块和交易状态模块
    requires trade.specification;
    requires transitive trade.state;
}
```

#### trade-callback

交易回调模块，交易中可能存在各种意外，比如网络抖动，响应超时等，回调模块负责定时回调业务系统方法，保证数据一致性

#### trade-channels

交易渠道模块，聚合各种渠道的交易，每个渠道实现各自逻辑的支付、退款逻辑



依赖于基础交易模块

```java
module payment.integration {
    // self.
    requires transitive trade.basic;
    requires trade.specification;
    requires jf.service;

    // about spring.
    requires spring.beans;
    requires spring.context;

    opens com.tony666.examples.spring.java9.payment.integration.facade;
    opens com.tony666.examples.spring.java9.payment.integration.config;
}
```

实现当前渠道的支付接口：

```java
@Service
public class IntegrationPayment extends AbstractPayment {

    private final JfPaymentService jfPaymentService;

    public IntegrationPayment(JfPaymentService jfPaymentService, List<Specification<PaymentState>> paymentSpecifications) {
        super(paymentSpecifications);
        this.jfPaymentService = jfPaymentService;
    }

    @Override
    public Optional<PaymentRes> doPayment(PaymentReq paymentReq) {
        JfPaymentReq jfPaymentReq = new JfPaymentReq();
        BeanUtils.copyProperties(paymentReq, jfPaymentReq);
        Optional<JfPaymentRes> payment = jfPaymentService.payment(jfPaymentReq);
        if (payment.isPresent()) {
            JfPaymentRes jfPaymentRes = payment.get();
            PaymentRes res = new PaymentRes();
            BeanUtils.copyProperties(jfPaymentRes, res);
            res.setPaymentState(PaymentState.SUCCESS);
            return Optional.of(res);
        }
        return Optional.empty();
    }
}
```

#### trade-specification

交易规则模块，定义基础交易规则，比如价格不匹配、余额不足、重复支付等，每个支付渠道可能有各自特有的约束，可在各自渠道中单独实现

#### trade-state

交易状态模块，记录所有交易状态



### 测试

```http
curl http://localhost:8080/trade/pay
curl http://localhost:8080/trade/refund
```



## 源码

[Github Repository](https://github.com/TonyLiu0112/spring-java9-examples)



## 结语

使用模块化编程在风格上有很大的不适应，模块的难点是合理的切分各个模块，定义清晰的接口，除了最小的应用程序之外，建议针对新代码采用模块。如果一开始就应用强封装并管理显示依赖，那么就为创建可维护系统奠定了坚实的基础