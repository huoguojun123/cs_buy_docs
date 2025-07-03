# OpenFeign: 声明式服务调用

项目使用 OpenFeign 作为服务间远程过程调用（RPC）的HTTP客户端。它极大地简化了服务调用的开发，让开发者可以像调用本地方法一样调用远程HTTP接口。

## 1. OpenFeign核心配置

### 1.1 启用Feign客户端

在每个需要调用其他服务的微服务启动类上，使用 `@EnableFeignClients` 注解来开启此功能。

```java
// 以用户服务为例
@SpringBootApplication
@EnableFeignClients(basePackages = "com.huangyuan.api.client", defaultConfiguration = FeignConfig.class)
public class UserServiceApplication {
    // ...
}
```
*   `basePackages`: 指定了存放 `@FeignClient` 接口的包路径。
*   `defaultConfiguration`: 指定了全局的Feign配置类。

### 1.2 更换HTTP客户端

默认情况下，Feign 使用 `java.net.URLConnection`。为了获得更好的性能和对连接池的支持，项目中将其替换为了 `OkHttp`。

在 `application.yml` 中配置：
```yaml
feign:
  okhttp:
    enabled: true # 启用OkHttp
  httpclient:
    enabled: false # 禁用默认的HttpClient
```
同时，需要确保 `pom.xml` 中已添加 `io.github.openfeign:feign-okhttp` 依赖。

## 2. Feign客户端定义 (API接口)

Feign的核心在于将HTTP API的定义抽象为Java接口。

```java
// 定义一个调用 item-service 服务的客户端
@FeignClient(value = "item-service", path = "/items")
public interface ItemClient {
    
    @GetMapping("/{id}")
    Result<ItemDTO> getItemById(@PathVariable("id") Long id);
    
    @GetMapping("/list")
    Result<List<ItemDTO>> listItemsByIds(@RequestParam("ids") List<Long> ids);
    
    @PutMapping("/stock/deduct")
    Result<Void> deductStock(@RequestBody List<StockDTO> stocks);
}
```
*   `@FeignClient(value = "item-service")`:
    *   `value`: 指定了要调用的微服务的名称，这个名称必须与服务在Nacos上注册的 `spring.application.name` 一致。Feign会通过Nacos发现 `item-service` 的所有实例地址。
*   **接口方法:**
    *   方法上的 `@GetMapping`, `@PostMapping` 等注解定义了要调用的远程API的HTTP方法和路径 (`path`)。
    *   方法的参数注解（如 `@PathVariable`, `@RequestParam`, `@RequestBody`）与Spring MVC中的用法完全一致，定义了参数如何传递。

## 3. Feign拦截器 (Interceptor)

拦截器是 Feign 强大的扩展点，可以用于在请求发出前进行统一处理，例如添加统一的请求头。

本项目中使用拦截器来传递 **JWT用户身份令牌** 和 **Seata分布式事务ID**。

```java
// 在全局Feign配置类 FeignConfig.java 中
public class FeignConfig {
    
    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            // 1. 获取当前请求上下文
            ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (attributes != null) {
                HttpServletRequest request = attributes.getRequest();
                
                // 2. 传递JWT令牌
                String token = request.getHeader("authorization");
                if (StringUtils.hasText(token)) {
                    requestTemplate.header("authorization", token);
                }

                // 3. 传递Seata事务ID (XID)
                String xid = RootContext.getXID();
                 if (StringUtils.hasText(xid)) {
                    requestTemplate.header(RootContext.KEY_XID, xid);
                }
            }
        };
    }
}
```
**工作原理:**
每当 Feign 客户端发起调用时，这个 `requestInterceptor` Bean都会被执行。它从当前线程的请求上下文 (`RequestContextHolder`) 中获取原始HTTP请求，然后抽取出 `authorization` 请求头和Seata的 `XID`，并将它们复制到即将发出的Feign请求的请求头中。这确保了用户的身份和当前的事务上下文能够无缝地在微服务调用链中传递下去。 