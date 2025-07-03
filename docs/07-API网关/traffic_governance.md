# 流量治理：API网关的核心职责

API网关作为所有请求的入口，是实施流量治理策略、保障后端服务稳定性的理想场所。本项目的Spring Cloud Gateway集成了多种流量控制机制，主要包括 **请求限流** 和 **服务熔断**。

## 1. 请求限流 (Rate Limiting)

为了防止恶意攻击或突发流量冲垮后端服务，网关层面必须进行限流。我们采用 **Redis + Lua脚本** 实现的分布式令牌桶算法，以实现精确、高效的全局限流。

### 实现原理

Spring Cloud Gateway内置了 `RequestRateLimiter` 过滤器工厂，它允许我们基于多种策略进行限流。

-   **KeyResolver**: 用于定义限流的维度。例如，可以基于用户ID、IP地址或API路径进行限流。
-   **RateLimiter**: 具体的限流算法实现。我们自定义了一个 `RedisRateLimiter`，它利用了Redis的原子性操作和Lua脚本，确保了分布式环境下限流决策的一致性。

### 配置示例

以下是 `application.yml` 中针对特定路由的限流配置：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_service_route
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - name: RequestRateLimiter
              args:
                # 指定我们自定义的Redis限流器Bean的名称
                rate-limiter: "#{@redisRateLimiter}" 
                # 指定用于从请求中提取限流键的KeyResolver Bean的名称
                key-resolver: "#{@ipKeyResolver}"
```

### 关键代码：IP地址KeyResolver

这是根据请求的IP地址来确定限流目标的 `KeyResolver` 实现。

```java
package com.nh.gateway.config;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.Objects;

@Configuration
public class RateLimiterConfig {

    /**
     * 基于请求IP地址的限流KeyResolver。
     * @return KeyResolver
     */
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getHostString());
    }
}
```
> **注意**: `RedisRateLimiter` 的具体实现依赖于 `spring-boot-starter-data-redis-reactive` 和相应的Lua脚本，脚本负责令牌的获取和补充逻辑。

## 2. 服务熔断 (Circuit Breaking)

当后端某个服务发生故障或响应缓慢时，为了防止故障蔓延（雪崩效应），网关需要快速失败，暂时切断对该服务的请求。本项目使用 **Sentinel** 作为熔断组件。

### 与Sentinel集成

Spring Cloud Gateway通过 `spring-cloud-starter-alibaba-sentinel-gateway` 实现了与Sentinel的无缝集成。Sentinel提供了更精细化的流量控制和熔断降级规则。

### 核心特性

1.  **路由维度熔断**: 可以为每个路由（即每个后端微服务）配置独立的熔断规则，如异常比例、异常数或平均响应时间。
2.  **自定义降级逻辑**: 当熔断发生时，Gateway可以执行一个预设的降级逻辑（Fallback），而不是直接报错。这可以返回一个友好的提示、默认数据或缓存内容，提升用户体验。

### 配置与实现

熔断规则通常在Sentinel的控制台中进行动态配置，这提供了极大的灵活性。当熔断触发时，我们需要提供一个`Fallback`处理程序。

```java
package com.nh.gateway.fallback;

import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.server.ServerResponse;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;
import com.alibaba.csp.sentinel.slots.block.BlockException;

import java.util.HashMap;
import java.util.Map;

@Component
public class GatewayFallbackHandler {

    /**
     * 处理Sentinel网关限流和熔断的统一Fallback。
     * @param exchange ServerWebExchange
     * @param throwable 异常
     * @return 响应
     */
    public Mono<ServerResponse> handle(ServerWebExchange exchange, Throwable throwable) {
        Map<String, Object> responseBody = new HashMap<>();
        
        // 检查是否是Sentinel的BlockException
        if (throwable instanceof BlockException) {
            responseBody.put("code", "FLOW_LIMIT");
            responseBody.put("message", "请求过于频繁，请稍后再试。");
        } else {
            // 可以从exchange中获取更详细的异常信息，例如导致熔断的异常
            Throwable originalException = exchange.getAttribute(ServerWebExchangeUtils.CIRCUITBREAKER_EXECUTION_EXCEPTION_ATTR);
            responseBody.put("code", "SERVICE_UNAVAILABLE");
            responseBody.put("message", "服务暂时不可用，熔断器已开启。");
            // 记录原始异常
            // log.error("Gateway熔断，原始异常: {}", originalException.getMessage());
        }

        return ServerResponse.status(HttpStatus.SERVICE_UNAVAILABLE)
                             .contentType(MediaType.APPLICATION_JSON)
                             .bodyValue(responseBody);
    }
}
```

在 `application.yml` 中，我们将Sentinel的处理程序指向这个Fallback：

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      gateway:
        fallback:
          # 模式: response 或 redirect
          mode: response 
          # 响应状态码
          response-status: 200 
          # 响应体
          response-body: '{"code":"SERVICE_FALLBACK","message":"服务暂时不可用"}'
          # 也可以指向一个Bean来处理更复杂的逻辑
          # fallback-route: "#{@myFallbackHandler}"
```
> **最佳实践**: 将复杂的降级逻辑交给专门的 `Fallback` Bean处理，而不是在YAML中写死一个JSON字符串，这样可以提供更灵活、更丰富的降级响应。 