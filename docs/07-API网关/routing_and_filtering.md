# 路由与过滤：网关的基础

路由（Routing）和过滤（Filtering）是 API 网关最核心的两个功能，它们共同决定了一个外部请求如何被处理并转发到哪个后端服务。

## 1. 核心概念

*   **路由 (Route):** 网关的基本构建块。它由一个唯一的ID、一个目标URI、一组用于匹配请求的 **断言（Predicates）** 和一组用于在请求前后进行处理的 **过滤器（Filters）** 组成。
*   **断言 (Predicate):** 一个返回布尔值的函数，用于匹配HTTP请求中的任何信息，如路径（Path）、请求头（Header）、Cookie、查询参数（Query）等。只有当所有断言都为真时，路由才会生效。
*   **过滤器 (Filter):** 在请求被路由到目标URI之前或之后执行的逻辑。过滤器可以修改请求头、请求体、响应头等。过滤器分为两种：
    *   `GatewayFilter`: 针对单个路由生效。
    *   `GlobalFilter`: 对所有路由都生效。

## 2. 静态路由配置

最直接的路由配置方式是在网关服务的 `application.yml` 文件中进行定义。

```yaml
# application.yml
spring:
  cloud:
    gateway:
      # 默认过滤器，对所有路由都会应用
      default-filters:
        - AddResponseHeader=X-Powered-By, Hy-Gateway

      # 路由规则集合
      routes:
        # 用户服务路由
        - id: user-service-route
          uri: lb://user-service # "lb"前缀表示从Nacos进行负载均衡
          predicates:
            - Path=/api/user/**   # 断言：匹配所有以/api/user/开头的路径
          filters:
            - StripPrefix=1     # 过滤器：将路径中的第一个部分(/api)剥离

        # 商品服务路由，带多个断言
        - id: item-service-route
          uri: lb://item-service
          predicates:
            - Path=/api/item/**
            - Header=X-Client-Version, v1\.\d+ # 断言：要求请求头有X-Client-Version且匹配v1.x.x格式
```

## 3. 动态路由配置

将路由规则硬编码在配置文件中缺乏灵活性。当服务或路由策略变更时，需要修改配置并重启网关。更优的方案是 **动态路由**，即网关可以实时地从配置中心（如 Nacos）获取路由规则并动态加载。

### 3.1 Nacos 配置

1.  **添加依赖:** 确保 `gateway-service` 的 `pom.xml` 中包含了 `spring-cloud-starter-alibaba-nacos-config`。
2.  **在 Nacos 控制台创建配置:**
    *   **Data ID:** `gateway-routes.json`
    *   **Group:** `DEFAULT_GROUP`
    *   **配置格式:** `JSON`
    *   **配置内容:**
        ```json
        [
          {
            "id": "dynamic-trade-route",
            "uri": "lb://trade-service",
            "predicates": [
              {
                "name": "Path",
                "args": {
                  "pattern": "/api/trade/**"
                }
              }
            ],
            "filters": [
              {
                "name": "StripPrefix",
                "args": {
                  "parts": "1"
                }
              }
            ]
          }
        ]
        ```
3.  **`bootstrap.yml` 配置:** 告知网关服务去 Nacos 读取配置。
    ```yaml
    spring:
      application:
        name: gateway-service
      cloud:
        nacos:
          config:
            server-addr: 127.0.0.1:8848
            file-extension: json
            # 监听 gateway-routes.json 文件
            shared-configs[0]:
              data-id: gateway-routes.json
              refresh: true # 开启动态刷新
    ```

完成以上配置后，当你在 Nacos 中修改 `gateway-routes.json` 并发布后，API 网关无需重启即可自动加载最新的路由规则。

## 4. 全局过滤器 (GlobalFilter)

全局过滤器对所有路由都生效，非常适合实现认证、日志、安全等横切关注点。

### 4.1 全局认证过滤器

在"安全与认证"章节中已详细讨论，此过滤器用于校验JWT令牌，并为请求添加用户信息头。它是网关最重要的安全屏障。

```java
// AuthGlobalFilter.java
@Component
@Order(-1) // 使用@Order注解设置过滤器优先级，数字越小，优先级越高
public class AuthGlobalFilter implements GlobalFilter {
    // ... 校验JWT的逻辑 ...
}
```

### 4.2 全局日志/审计过滤器

记录所有进出网关的请求日志，对于问题排查和安全审计至关重要。

```java
// AuditLogGlobalFilter.java
@Component
@Order(-10) // 确保在认证过滤器之前执行，以便记录所有请求
public class AuditLogGlobalFilter implements GlobalFilter {

    private static final Logger logger = LoggerFactory.getLogger("audit");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        long startTime = System.currentTimeMillis();

        return chain.filter(exchange).then(
            Mono.fromRunnable(() -> {
                long duration = System.currentTimeMillis() - startTime;
                ServerHttpResponse response = exchange.getResponse();
                // 构建包含请求路径、方法、IP、用户ID（从请求头获取）、状态码、响应时间等信息的日志对象
                String logMessage = String.format(
                    "IN: %s %s from %s | OUT: status=%s, duration=%dms",
                    request.getMethod(),
                    request.getURI(),
                    request.getRemoteAddress(),
                    response.getStatusCode(),
                    duration
                );
                logger.info(logMessage);
            })
        );
    }
}
```
通过为不同的日志目的（如 `audit`、`access`）使用不同的 `Logger`，可以结合 Logback 等日志框架的配置，将不同类型的日志输出到不同的文件中。 