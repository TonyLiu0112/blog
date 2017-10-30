---
title: Spring Cloud 访问权限控制（JWT场景）
tags: 
    - Spring Security
    - Spring Cloud
    - JWT
categories:
    - Spring CLoud
---

业务场景中精彩遇到典型的权限问题，包括用户权限，数据权限，访问权限，如果使用传统的web开发模式，可以使用Spring Security项目的相关注解完成配置，具体参考：[Spring Security Document](https://docs.spring.io/spring-security/site/docs/4.2.3.RELEASE/reference/htmlsingle/#method-security-expressions).

但是如果使用JWT(Json Web Token)方式，需要额外处理.

## 什么是JWT
参考：[JWT](http://www.jianshu.com/p/576dbf44b2ae)

## 实现权限控制

### 前提
* 有一个可以正常获取JWT的服务器，能获取到响应的JWT.[示例代码](https://github.com/TonyLiu0112/mud-microservice-demo/tree/master/mud-microservice-security/mud-microservice-security-jwt)
* JWT中包含权限信息[ROLE_USER]

这是一个JWT负载的示例
```
{
    "scope":[
        "read",
        "write"
    ],
    "organization":"acmePaPV",
    "exp":1509392738,
    "user":{
        "id":1,
        "loginName":"admin",
        "nickname":"超级管理员",
        "phone":"13333333333",
        "email":"tony.liu0112@gmail.com",
        "sex":1,
        "roles":[
            "ROLE_USER"
        ]
    },
    "jti":"c9533638-0501-4e52-b308-cf4bbc60d04d",
    "client_id":"acme"
}
```

>思路: 实现时候考虑过使用`@PreFilter`注解，然并卵... 可考虑过用Spring拦截器、Servlet过滤器，但实现上获取参数较为繁琐。目前思路比较简单，使用自定义注解，通过aspectj拦截当前注解，解析JWT的payload信息，根据注解允许权限做简单鉴权.

具体实现如下

### 自定义注解

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Roles {

    String[] values();

}
```

### AspectJ解析类

```
@Component
@Aspect
public class AccessControlChecker {

    private Logger logger = LoggerFactory.getLogger(AccessControlChecker.class);

    @Autowired
    private DefaultTokenServices defaultTokenServices;

    @Around("@annotation(Roles) && execution(public * *(..))") // 对使用Roles注解方法做切面处理
    public Object checkRole(final ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) proceedingJoinPoint.getSignature();
        Roles roles = signature.getMethod().getAnnotation(Roles.class);
        if (roles == null || roles.values().length == 0)
            return proceedingJoinPoint.proceed();
        // 获得切面方法参数
        Object[] args = proceedingJoinPoint.getArgs();
        OAuth2Authentication oAuth2Authentication = null;

        for (Object arg : args) {
            if (arg instanceof OAuth2Authentication) {
                oAuth2Authentication = (OAuth2Authentication) arg;
            }
        }

        if (oAuth2Authentication == null) {
            throw new NullPointerException("When use annotation of @Roles, you need to use parameter of OAuth2Authentication. " +
                    "Error may be in " +
                    String.format("%s.%s.%s",
                            signature.getMethod().getDeclaringClass().getPackage(),
                            signature.getMethod().getDeclaringClass().getName(),
                            signature.getMethod().getName()));
        }
        // 鉴权
        boolean hasRole = readRoles(oAuth2Authentication)
                .anyMatch(grantedAuthority ->
                        Stream.of(roles.values()).anyMatch(role -> role.equals(grantedAuthority)));
        if (!hasRole) {
            logger.error("", new AccessControlException("No permission to access."));
            return RestfulBuilder.unauthorized(null, "No user role to access.");
        }
        return proceedingJoinPoint.proceed();
    }

    // 解析JWT的Payload中的权限信息
    private Stream<String> readRoles(OAuth2Authentication oAuth2Authentication) {
        Map<String, Object> additionalInformation = defaultTokenServices.readAccessToken(getToken(oAuth2Authentication)).getAdditionalInformation();
        if (additionalInformation.get("user") != null && additionalInformation.get("user") instanceof Map) {
            Map<String, Object> user = (Map<String, Object>) additionalInformation.get("user");
            return ((List<String>) user.get("roles")).stream();
        }
        return Stream.empty();
    }

    private String getToken(OAuth2Authentication oAuth2Authentication) {
        OAuth2AuthenticationDetails details = (OAuth2AuthenticationDetails) oAuth2Authentication.getDetails();
        return details.getTokenValue();
    }

}
```

### 使用

```
@GetMapping("test")
@Roles(values = {"ROLE_USER"}) // 仅仅有ROLE_USER权限才可访问
public ResponseEntity<RestfulResponse> getShoppingcard(OAuth2Authentication oAuth2Authentication) {
    // your logic
}
```
