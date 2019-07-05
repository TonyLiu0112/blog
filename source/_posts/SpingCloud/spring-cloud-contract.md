---
title: Spring Cloud Contract服务契约
tags: 
    - 契约
    - Spring Cloud Contract
    - STUB
categories:
    - Spring Cloud

---



## 简介

在开发分布式服务时，经常需要面对多个服务之间的依赖，譬如开发一个功能，A服务依赖于B服务的接口，开发过程中，可能B服务能先定义好接口让A服务进行对接开发，但是A服务的测试往往是困难的，A服务测试必然要依赖B服务部署的接口，B服务需要保证服务的可用性，如果B服务是一个功能复杂的服务，对A服务的测试，就是一个漫长的等待…...

![](Deps.png)

`spring-cloud-contract`就是解决这个问题的一个组件，`contract`就像一个契约，服务调用双方都遵循这个契约来完成自己的开发、测试

![](Stubs2.png)

从图上看来，似乎`Stub`就是提供了一个按约定的、能正常访问的、返回固定值的一个接口服务而已呀？或者撇开`stub`不说，我们自己也能给予`springmvc`写一个可访问的接口，返回固定值，然后部署起来给调用方使用。这种思路是没有错，但是比较繁琐，要为接口部署一整套的环境，模拟的服务甚至还会和可用的服务冲突，如果资源紧张，就更多问题了。`Stub`提供的，就不是基于这种部署进程服务的方式，`stub`所提供的功能都是基于测试的，服务提供者使用`spring-cloud-contract`完成契约的定义后，只要把这个`jar`包提供到调用方，调用方在无需启动目标服务的情况下，就能完成服务的调用。

## 实战

使用maven构建如下项目结构（略）

![](project-tree.png)



### provider

首先来看provider，我们构建的provider项目的maven信息是

```xml
<groupId>com.tony666.examples</groupId>
<artifactId>contract-provider</artifactId>
```



要使用`spring-cloud-contract`，需要加入如下依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-contract-verifier</artifactId>
  <version>2.1.2.RELEASE</version>
  <scope>test</scope>
</dependency>
```

依赖契约插件

```xml
<plugin>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-contract-maven-plugin</artifactId>
  <version>2.1.2.RELEASE</version>
  <extensions>true</extensions>
</plugin>
```

##### 定义契约

在项目`test`目录的`resources`文件目录下新建`contracts`文件夹（契约支持`groovy`、`yml`等描述方式，这里使用`groovy`），新建契约文件: `payment.groovy`

```groovy
package contracts

import org.springframework.cloud.contract.spec.Contract
import org.springframework.http.HttpHeaders
import org.springframework.http.MediaType

Contract.make {
    description "扫码支付"

    request {
        method POST()
        url "/payment/codes"
    }

    response {
        status OK()
        headers { header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON) }
        body([
                "code"   : 100,
                "message": "success",
                "data"   : [
                        "resultCode": 100,
                        "resultMsg" : "success",
                        "payTransId": "4200000313201907049761660775",
                        "fmId"      : "7702285149085300932638824",
                        "payCode"   : "10215",
                        "userId"    : "o1cvUM2t4SgkdamVkn123um7e613qo-ssssRsssw"
                ]])
    }
}
```

##### 编写controller

```java
@RestController
@RequestMapping("payment")
@Slf4j
public class PaymentController {

    @PostMapping("codes")
    public Object codePay() {
        return null;
    }

}
```

##### 使用契约插件启用测试

![](stub-test.png)

访问接口可以看到

![](test-post-result.png)

##### 讲契约jar包安装到本地maven私服仓库

项目根目录运行

```shell
mvn clean install -DskipTests -e
```



### consumer

consumer的需要使用契约服务，需要添加依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
  <version>2.1.2.RELEASE</version>
  <scope>test</scope>
</dependency>
```

在test目录下编写JUnitTest类

```java
@Slf4j
@RunWith(SpringRunner.class)
@SpringBootTest(classes = HttpClientConfig.class, webEnvironment = SpringBootTest.WebEnvironment.NONE)
@AutoConfigureStubRunner(ids = {"com.tony666.examples:contract-provider:+:8080"}, stubsMode = StubRunnerProperties.StubsMode.LOCAL)
public class BusinessTests {

    @Autowired
    private RestTemplate restTemplate;

    @Test
    public void test() {
        ResponseEntity<String> mapResponseEntity = restTemplate.postForEntity("http://localhost:8080/payment/codes", null, String.class);
        JSONObject resp = JSON.parseObject(mapResponseEntity.getBody());
        log.info(resp.toString());
        Assert.assertEquals(100, resp.get("code"));
    }

}

```

这里主要对`@AutoConfigureStubRunner`注解内的属性解释一下

* `ids`：由一组约定的分号分割的字符来表示`provider`的信息，约定规则为：`groupId:artifactId:version:classifier:port`，代码中的`+`代表默认值`stubs`
* `stubsMode`：契约模式，可以读取本地的契约，也可以读取一个远程的契约，这里使用的是本地仓库的契约

运行上述JunitTest，在不启动以来服务的前提下，`restTemplate`正常完成了一次原创调用，log中响应信息也正常返回

```verilog
Matched response definition:
{
  "status" : 200,
  "body" : "{\"code\":100,\"data\":{\"fmId\":\"7702285149085300932638824\",\"payTransId\":\"4200000313201907049761660775\",\"resultCode\":100,\"payCode\":\"10215\",\"userId\":\"o1vUMt4SgkmVkn2um7e63qo-ssRw\",\"resultMsg\":\"success\"},\"message\":\"success\"}",
  "headers" : {
    "Content-Type" : "application/json"
  },
  "transformers" : [ "response-template" ]
}

Response:
HTTP/1.1 200
Content-Type: [application/json]
Matched-Stub-Id: [7ccf85cd-17e9-48a2-8399-a77290ed64a1]


2019-07-04 22:07:46.166  INFO 29319 --- [           main] c.t.examples.consumer.BusinessTests      : {"code":100,"data":{"fmId":"7702285149085300932638824","payTransId":"4200000313201907049761660775","resultCode":100,"payCode":"10215","userId":"o1vUMt4SgkmVkn2um7e63qo-ssRw","resultMsg":"success"},"message":"success"}

```



源码地址：https://github.com/TonyLiu0112/contract-demo