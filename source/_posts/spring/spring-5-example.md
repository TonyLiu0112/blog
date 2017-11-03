---
title: Spring 5.0 Example
tags: 
    - spring
    - spring5
categories:
    - Spring
---

Spring官网已经出了5.0版本的blog，预计不久将来就要普及了. 参考:[Spring 5 blog](https://spring.io/)

5.0官方定义是：`Functional Web Framework`，功能新框架不仅支持新的路由等处理方式，还支持传统的注解方式（`@RequestMapping` `@Controller`）

## 新特性

### 路由机制

使用`RouterFunction`定义自己的访问路由.

``` java
RouterFunction<?> route = route(GET("/person/{id}"),
  request -> {
    Mono<Person> person = Mono.justOrEmpty(request.pathVariable("id"))
      .map(Integer::valueOf)
      .then(repository::getPerson);
    return Response.ok().body(fromPublisher(person, Person.class));
  })
  .and(route(GET("/person"),
    request -> {
      Flux<Person> people = repository.allPeople();
      return Response.ok().body(fromPublisher(people, Person.class));
    }))
  .and(route(POST("/person"),
    request -> {
      Mono<Person> person = request.body(toMono(Person.class));
      return Response.ok().build(repository.savePerson(person));
    }));
```

### Flux和Mono

规范了方法的返回，当你希望返回一个`List<Persion>`对象的时候，使用`Flux`对象作为方法返回值，当你希望返回一个普通`Person`对象，使用`Mono<Persion>`作为返回值.

之所以有这个，目测是整个架构上对lambada的全面支持.

``` java
public iinterface DemoService {
  Mono<Person> getPerson(int id);
  Flux<Person> allPeople();
  Mono<Void> savePerson(Mono<Person> person);
}
```

### lambda全面支持

功能性框架变化最大的就是对Java8的lambda全面支持，所有业务逻辑基本是都是按照函数式编程风格开发. 基本符合了现代语言的特性

## 实战

### 基本信息
目前spring5还未正式上线，官方发布的是一个预生产版本`2.0.0.M5`,参考: [Documents](https://docs.spring.io/spring-boot/docs/2.0.0.M5/reference/htmlsingle/)
#### java8
#### mave
#### spring-boot 2.0.0.M5

### 项目基本结构

``` bash
src
    ├── config
    ├── handlers
    ├── model
    ├── routers
    └── services
```

### 添加依赖

``` maven
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.0.M5</version>
</parent>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/libs-milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

### 编写service类

``` java
@Service
public class MessageService {

    public Mono<MessageResponse> getMessage(Mono<String> nameMono) {
        return nameMono.flatMap(name -> Mono.just(new MessageResponse(name, "Hi, " + name + "! Welcome to Spring 5!")));
    }

    public Mono<MessageResponse> saveMessage(Mono<MessageRequest> messageRequestMono) {
        return messageRequestMono.flatMap(messageRequest -> Mono.just(new MessageResponse(messageRequest.getName(), "保存消息成功")));
    }

}
```

### 编写handler类

``` java
@Component
public class MessageHandler {

    private final MessageService messageService;
    private final ErrorHandler errorHandler;

    public MessageHandler(MessageService messageService, ErrorHandler errorHandler) {
        this.messageService = messageService;
        this.errorHandler = errorHandler;
    }

    /**
     * 获取消息
     *
     * @param request
     * @return
     */
    public Mono<ServerResponse> getMessage(final ServerRequest request) {
        return Mono.just(request.queryParam("name").orElse("Default")) // 获取查询参数
                .transform(this::buildResponse) // 转换成响应
                .onErrorResume(errorHandler::buildError); // 错误处理
    }

    private Mono<ServerResponse> buildResponse(final Mono<String> nameMono) {
        return nameMono
                .transform(messageService::getMessage)
                .transform(this::serverResponse);
    }

    /**
     * 提交消息
     *
     * @param request
     * @return
     */
    public Mono<ServerResponse> postMessage(final ServerRequest request) {
        return request.bodyToMono(MessageRequest.class) // 获取http body参数，并映射到MessageRequest对象上
                .transform(messageService::saveMessage) // 转换请求对象，也就是调用业务逻辑
                .transform(this::serverResponse) // 构建响应对象
                .onErrorResume(errorHandler::buildError); // 错误处理
    }

    private Mono<ServerResponse> serverResponse(final Mono<MessageResponse> messageResponseMono) {
        return messageResponseMono.flatMap(messageResponse ->
                ServerResponse.ok().body(Mono.just(messageResponse), MessageResponse.class));
    }

}
```

### 编写路由器

`RouterFunction`是抽象路由的对象，这个对象通过`RouterFunctions`工具类来构建，通过`andOther()`方法将各个路由器组合在一起，这个方法每次都返回一个新的实例，源码可以推测，路由对象不断构建的应该是一个类似树结构的对象.

按理来说，全局应该只有一个`RouterFunction`对象，如果只使用单独构建的话，也就意味着这个对象无法托管给spring，而且构建的时候，所有的依赖spring对象都需要通过构造函数传入，业务多了非常繁琐:

``` java
public class MainRouter {
    public static RouterFunction<?> doRoute(final Biz1Hander handler, final ErrorHandler errorHandler) {
        return Biz1Router
                .doRoute(handler, errorHandler)
                .andOther(Biz2Router.doRoute());
    }
}

@Configuration
public class Configurations {

    // 如果MainRouter依赖的过多的处理器，那么就要注入非常多的处理对象。
    @Bean
    RouterFunction<?> mainRouterFunction(final Biz1Hander apiHandler, final ErrorHandler errorHandler) {
        return MainRouter.doRoute(apiHandler, errorHandler);
    }
}
```

针对上面问题，为了解耦。先构建一个默认路由器, 该路由器仅仅为了初始化一个`RouterFunction`对象给spring管理
``` java
@Component
public class DefaultRouter {
    @Bean
    public RouterFunction<?> routerFunction() {
        return nest(path("/"),
                nest(accept(MediaType.ALL),
                        route(
                                GET("/default"),
                                request -> ServerResponse.ok().body(Mono.just("Worked!"), String.class)
                        )
                )
        );
    }
}
```

业务路由器1
``` java
@Component
@ConditionalOnBean(DefaultRouter.class) // 当spring上下文有这个对象，才初始化
public class MessageRouter {

    private final MessageHandler messageHandler;
    private final ErrorHandler errorHandler;

    public MessageRouter(MessageHandler messageHandler, ErrorHandler errorHandler) {
        this.messageHandler = messageHandler;
        this.errorHandler = errorHandler;
    }

    /**
     * 注入主路由器, 加入当前业务的路由逻辑，并返回到spring
     */
    @Bean
    public RouterFunction<?> setMessageRouter(RouterFunction<?> routerFunction) {
        return routerFunction.andOther(
                nest(path("/msg"), // 业务根路径
                        nest(accept(APPLICATION_JSON), // 接收的类型
                                route(GET("/message"), messageHandler::getMessage) // 处理/msg/message get请求
                                        .andRoute(POST("/message"), messageHandler::postMessage) // 处理 /msg/message post请求
                        ).andOther(route(all(), errorHandler::notFund)) // 匹配失败统一处理
                )
        );
    }

}
```

这样可以实现多个路由的解耦，但存在一个问题，业务路由对象多了之后，spring 上下文中会存在多个RouterFunction对象，暂时还没找到更好的解决办法.

### 测试

请求务必添加accept-type: `application/json`
``` bash
GET／POST: http://localhost:8080/msg/message
```

源码地址: (https://github.com/TonyLiu0112/spring5-example)[https://github.com/TonyLiu0112/spring5-example]






