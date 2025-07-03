# 高级模式：动态路由与API聚合

除了基础的路由和流量治理，现代API网关还承担着更复杂的职责，以简化客户端逻辑、提升系统灵活性。本章将介绍项目中应用的两种高级网关模式：**动态路由** 和 **API聚合**。

## 1. 动态路由 (Dynamic Routing)

在微服务架构中，服务实例的上线和下线是常态。为了避免每次服务变更都去修改网关的静态配置并重启，我们必须实现动态路由。路由信息应该集中存储和管理，网关能够自动感知变化并刷新内存中的路由表。

### 实现方案：Nacos配置中心

本项目利用 **Nacos** 作为动态路由的配置中心。

1.  **路由配置存储**: 所有的路由规则（ID, URI, Predicates, Filters）都以JSON格式存储在Nacos的一个配置文件中。
2.  **动态刷新**: Spring Cloud Gateway与Spring Cloud Alibaba Nacos Config深度集成。当Nacos中的路由配置发生变化时，Nacos会主动通知网关实例。
3.  **自动更新**: 网关监听到配置变更后，会自动刷新其应用上下文，加载新的路由规则，整个过程无需重启，对线上流量无影响。

#### Nacos中的路由配置示例 (`gateway-router.json`)

```json
[
  {
    "id": "item_service_route_dynamic",
    "uri": "lb://item-service",
    "predicates": [
      {
        "name": "Path",
        "args": {
          "pattern": "/api/item/**"
        }
      }
    ],
    "filters": [
      {
        "name": "StripPrefix",
        "args": {
          "parts": "2"
        }
      }
    ]
  },
  {
    "id": "trade_service_route_dynamic",
    "uri": "lb://trade-service",
    "predicates": [
      {
        "name": "Path",
        "args": {
          "pattern": "/api/trade/**"
        }
      }
    ]
  }
]
```

#### 网关`bootstrap.yaml`配置

为了让Gateway能从Nacos加载配置，需要在 `bootstrap.yaml` 中进行设置。

```yaml
spring:
  application:
    name: gateway-service
  cloud:
    nacos:
      config:
        server-addr: localhost:8848
        # 监听的文件配置
        shared-configs:
          - data-id: gateway-router.json # 路由配置文件
            group: DEFAULT_GROUP
            refresh: true # 开启动态刷新
```

## 2. API聚合 (API Aggregation)

在复杂的业务场景中，前端页面可能需要展示来自多个微服务的数据。例如，一个商品详情页需要同时调用商品服务获取基本信息、库存服务获取库存、以及评论服务获取用户评论。如果让客户端分别调用这三个API，会增加网络开销和客户端的逻辑复杂度。

**API聚合** 模式就是在API网关层提供一个统一的、粗粒度的API端点，网关在内部并行调用多个下游微服务，将结果聚合并处理后，一次性返回给客户端。

### 实现方案：自定义聚合过滤器

Spring Cloud Gateway的灵活性允许我们创建自定义的全局过滤器（`GlobalFilter`）来实现API聚合。

#### 聚合流程

1.  **定义聚合路由**: 在网关配置一个特殊的路由，例如 `/api/aggregate/product-details/{id}`。
2.  **创建聚合过滤器**:
    *   这个过滤器会拦截指向聚合路由的请求。
    *   它从请求路径中解析出商品ID。
    *   使用 `WebClient` 并行地向商品服务、库存服务和评论服务发起异步请求。
    *   利用 `Mono.zip` 或 `Flux.zip` 等Reactor操作符等待所有下游响应返回。
    *   将各个响应的数据提取出来，组合成一个新的、统一的数据结构（DTO）。
    *   将聚合后的结果写入响应体，并结束请求-响应链。

#### 关键代码示例：ProductDetailsAggregationFilter

```java
package com.nh.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.Map;

@Component
public class ProductDetailsAggregationFilter implements GlobalFilter, Ordered {

    private final WebClient.Builder webClientBuilder;

    public ProductDetailsAggregationFilter(WebClient.Builder webClientBuilder) {
        this.webClientBuilder = webClientBuilder;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // 仅对特定聚合路径生效
        if (!path.startsWith("/api/aggregate/product-details/")) {
            return chain.filter(exchange);
        }

        String productId = path.substring(path.lastIndexOf('/') + 1);

        // 并行调用多个服务
        Mono<Map> itemInfoMono = webClientBuilder.build()
                .get().uri("http://item-service/api/items/{id}", productId)
                .retrieve().bodyToMono(Map.class);

        Mono<Map> stockInfoMono = webClientBuilder.build()
                .get().uri("http://stock-service/api/stock/{id}", productId)
                .retrieve().bodyToMono(Map.class);
        
        Mono<Map> reviewInfoMono = webClientBuilder.build()
                .get().uri("http://review-service/api/reviews/summary/{id}", productId)
                .retrieve().bodyToMono(Map.class);

        // 使用Mono.zip聚合结果
        return Mono.zip(itemInfoMono, stockInfoMono, reviewInfoMono)
                .flatMap(tuple -> {
                    Map<String, Object> aggregatedResponse = new HashMap<>();
                    aggregatedResponse.put("item", tuple.getT1());
                    aggregatedResponse.put("stock", tuple.getT2());
                    aggregatedResponse.put("reviews", tuple.getT3());

                    // 将聚合后的JSON写入响应
                    exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
                    String jsonBody = convertMapToJson(aggregatedResponse); // 伪代码：需要一个JSON转换库
                    var buffer = exchange.getResponse().bufferFactory().wrap(jsonBody.getBytes(StandardCharsets.UTF_8));
                    
                    return exchange.getResponse().writeWith(Mono.just(buffer));
                });
    }

    @Override
    public int getOrder() {
        // 需要在路由过滤器之前执行
        return -1;
    }
    
    // 伪代码，实际应使用如Jackson等库
    private String convertMapToJson(Map<String, Object> map) {
        // Implement JSON serialization
        return "{\"item\":{...},\"stock\":{...},\"reviews\":{...}}";
    }
}

```
> **注意**: 为了使 `WebClient` 能够解析 `lb://` 协议，需要用 `@LoadBalanced` 注解来创建 `WebClient.Builder` Bean。API聚合虽然强大，但也增加了网关的复杂性和业务耦合度，需要谨慎使用。

通过这些高级模式，API网关从一个简单的请求转发器演变成了一个强大的流量调度与编排中心，极大地提升了整个微服务架构的鲁棒性和可维护性。 