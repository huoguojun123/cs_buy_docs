# 服务治理：负载均衡、降级与容错

在复杂的微服务环境中，必须有一套有效的服务治理机制来确保系统的稳定性和高可用性。本项目主要通过 Spring Cloud LoadBalancer 实现负载均衡，通过 Sentinel 实现服务降级、熔断和流量控制。

## 1. 客户端负载均衡

当一个服务（如 `trade-service`）通过 Nacos 发现了 `item-service` 的多个健康实例时，它需要在这些实例中选择一个来发送请求。这个选择的过程就是客户端负载均衡。

项目使用的是 Spring Cloud 官方推荐的 `Spring Cloud LoadBalancer`。

### 1.1 负载均衡策略

Spring Cloud LoadBalancer 内置了两种核心策略：
*   **`RoundRobinLoadBalancer` (轮询，默认):** 按顺序循环选择服务实例。这是最简单也是默认的策略，适用于所有实例性能相近的场景。
*   **`RandomLoadBalancer` (随机):** 从可用的实例列表中随机选择一个。

在未来的版本中，可以根据业务需求自定义更复杂的负载均衡策略，例如：
*   **权重轮询 (`WeightedRoundRobin`):** 在 Nacos 中为不同的实例配置不同的权重，性能更好的机器可以分配更高的权重，从而接收更多的请求。
*   **一致性哈希 (`ConsistentHash`):** 根据请求的某个特定参数（如 `userId`）来进行哈希，使得同一用户的请求总是被路由到同一个服务实例，便于利用本地缓存。

## 2. 服务降级与容错 (Sentinel)

为了防止单个服务的故障引发整个系统的雪崩效应，项目引入了 `Sentinel` 作为服务容错的核心组件。Sentinel 提供了 **流量控制**、**熔断降级**、**系统负载保护** 等多个维度的能力。

*注意: 虽然原始文档提到了 Hystrix 的 fallback 方式，但 Spring Cloud Alibaba 技术栈通常与 Sentinel 结合更紧密。以下为基于 Sentinel 的实现方案。*

### 2.1 流量控制 (限流)

流量控制用于限制访问某个API的速率，防止因瞬间流量过高而拖垮服务。

**示例:** 可以在 Sentinel 控制台针对 `ItemController` 的 `getItemById` 方法设置流控规则：
*   **资源名:** `GET:/items/{id}`
*   **QPS阈值:** 200
*   **含义:** 限制该接口每秒最多被调用200次，超过的部分将被快速失败或排队等待。

### 2.2 熔断降级

熔断降级是在检测到依赖的服务出现严重问题（如大量超时或异常）时，主动"熔断"对其的调用，在一段时间内直接返回一个降级（兜底）响应，从而避免对故障服务的持续请求，给其恢复的时间。

**Sentinel 的熔断策略:**
*   **慢调用比例:** 在指定时间内，如果请求的响应时间大于设定的阈值（"慢调用"）的比例超过设定值，则触发熔断。
*   **异常比例/数:** 在指定时间内，如果请求的异常比例或异常总数超过设定值，则触发熔断。

### 2.3 在 OpenFeign 中集成 Sentinel

当 Feign 调用触发了 Sentinel 的降级或流控规则时，可以指定一个 Fallback 类来提供兜底响应。

1.  **开启 Feign 对 Sentinel 的支持:**
    ```yaml
    feign:
      sentinel:
        enabled: true
    ```

2.  **创建 Fallback 类:**
    该类必须实现 Feign 客户端接口，并注册为一个Bean。

    ```java
    @Component
    public class ItemClientFallback implements ItemClient {
        private static final String FALLBACK_MSG = "获取商品信息失败，请稍后重试";

        @Override
        public Result<ItemDTO> getItemById(Long id) {
            log.error("触发熔断: getItemById({})", id);
            return Result.error(FALLBACK_MSG);
        }

        @Override
        public Result<List<ItemDTO>> listItemsByIds(List<Long> ids) {
            log.error("触发熔断: listItemsByIds(...)");
            return Result.error(FALLBACK_MSG);
        }

        // ...其他方法的降级实现
    }
    ```
3.  **在 FeignClient 中指定 Fallback:**
    ```java
    @FeignClient(value = "item-service", fallback = ItemClientFallback.class)
    public interface ItemClient {
        // ...
    }
    ```
现在，当对 `item-service` 的调用被熔断时，程序将不会抛出异常，而是会直接调用 `ItemClientFallback` 中对应的方法，向前端返回一个友好的错误提示。

## 3. 微服务间数据一致性

这是一个复杂的话题，本项目主要依赖在 **[第4章：分布式事务](./../04-分布式事务/README.md)** 中详细讨论的 **Seata** 方案来保证最终一致性。对于非核心业务或允许短暂不一致的场景，也会采用其他策略，例如：

*   **可靠消息最终一致性:** 通过消息队列（MQ）进行异步通知，并配合消息确认和重试机制来保证消息一定能被消费，从而实现下游服务的状态同步。
*   **定时任务校对:** 定期运行批处理任务，比对不同微服务之间的数据，找出不一致的部分并进行修复。 