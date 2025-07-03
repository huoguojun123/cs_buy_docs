# 7. API 网关：微服务的大门

API 网关是整个微服务架构的统一入口，它封装了系统的内部结构，为客户端提供了一个单一、聚合的 API 端点。在本项目中，API 网关（`gateway-service`）基于 Spring Cloud Gateway 构建，它扮演着路由、安全、监控和治理等多重角色。

为了便于查阅，详细内容已拆分到以下独立的专题文档中：

*   **[路由与过滤：网关的基础](./routing_and_filtering.md)**
    *   深入探讨了网关最核心的两个功能：**路由（Routing）** 和 **过滤（Filtering）**。内容涵盖了静态路由配置、全局过滤器的实现（如认证、日志），并重点阐述了如何集成 Nacos 实现 **动态路由**。

*   **[流量治理：系统的稳定器](./traffic_governance.md)**
    *   集中讨论了网关如何对流量进行管控以保障后端服务的稳定性。内容包括 **请求限流（Rate Limiting）** 和 **服务熔断（Circuit Breaking）** 的原理与配置。

*   **[高级模式：聚合与转换](./advanced_patterns.md)**
    *   介绍 API 网关的一些高级应用模式，例如 **API 聚合（API Aggregation）**，即由网关调用多个下游服务并组合其结果，从而对客户端提供更粗粒度的接口，降低前端复杂度。

## 核心逻辑流

下图展示了API网关处理单个请求时，如何通过路由断言匹配，并依次经过过滤器链的核心处理流程。

```puml
@startuml
!theme vibrant
title API Gateway 核心请求流程

cloud "客户端" as Client

package "API 网关 (Spring Cloud Gateway)" {
  participant "路由断言\n(Route Predicates)" as Predicates
  participant "过滤器链\n(Filter Chain)" as Filters
}

package "后端微服务集群" {
  participant "用户服务" as UserService
  participant "商品服务" as ItemService
  participant "订单服务" as TradeService
}

Client --> Predicates: 1. 发起请求 (e.g., /api/item/123)
Predicates -> Predicates: 2. 匹配路径规则 (Path=/api/item/**)
Predicates --> Filters: 3. 找到匹配路由，将请求交给过滤器链

group 过滤器链处理
  Filters -> Filters: 4. [Global] 认证过滤 (AuthFilter)
  Filters -> Filters: 5. [Global] 日志过滤 (LoggingFilter)
  Filters -> Filters: 6. [Route] 移除前缀 (StripPrefix)
  Filters -> Filters: 7. [Route] 限流过滤 (RateLimiter)
end

Filters --> ItemService: 8. 转发请求到商品服务
ItemService --> Filters: 9. 返回响应
Filters --> Client: 10. 经过过滤器链后，返回最终响应

@enduml
```

## 系统架构图

下图从更高层次描绘了API网关在整个系统中的位置，展示了它与客户端、负载均衡、注册中心（Nacos）、以及后端微服务集群的交互关系。

![API网关系统架构图](./api_gateway_architecture.puml) 