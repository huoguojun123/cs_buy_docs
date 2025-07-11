# 6. 缓存策略：系统性能的加速器

在现代高性能应用中，缓存是不可或缺的一环。它位于应用与数据源（如数据库）之间，通过在内存中存储热点数据，极大降低了对数据库的直接访问，从而显著提升了应用的响应速度和吞吐能力。本项目构建了一套完善的多级缓存体系，并针对各种缓存异常场景制定了周密的解决方案。

为了便于查阅，详细内容已拆分到以下独立的专题文档中：

*   **[Redis基础应用与核心模式](./redis_basic_usage.md)**
    *   详细介绍了Redis在项目中的多种应用场景，并深入剖析了业界最主流的缓存模式——"旁路缓存（Cache-Aside Pattern）"的读写策略与实现细节。

*   **[缓存高可用设计与异常处理](./cache_high_availability.md)**
    *   集中探讨了如何通过多级缓存（本地缓存 + 分布式缓存）构建高可用缓存架构，并详细阐述了针对"缓存穿透、击穿、雪崩"这三大经典问题的解决方案和代码实现。

## 核心缓存架构概览

系统采用 **"本地缓存 + 分布式缓存"** 的多级缓存架构，以兼顾速度与共享。

```puml
@startuml
!theme vibrant
title 多级缓存读取流程

cloud "客户端请求" as Client
package "应用服务器 (JVM内)" {
  participant "Caffeine (L1)" as L1_Cache
}
package "分布式缓存集群" {
  participant "Redis (L2)" as L2_Cache
}
database "数据库" as DB

Client --> L1_Cache: 1. 请求数据
alt L1 命中
  L1_Cache --> Client: 2. 直接返回
else L1 未命中
  L1_Cache --> L2_Cache: 3. 查询 L2 缓存
  alt L2 命中
    L2_Cache --> L1_Cache: 4. 返回数据
    L1_Cache -> L1_Cache: 5. 写入 L1 缓存
    L1_Cache --> Client: 6. 返回数据
  else L2 未命中
    L2_Cache --> DB: 7. 查询数据库
    DB --> L2_Cache: 8. 返回数据
    L2_Cache -> L2_Cache: 9. 写入 L2 缓存
    L2_Cache --> L1_Cache: 10. 返回数据
    L1_Cache -> L1_Cache: 11. 写入 L1 缓存
    L1_Cache --> Client: 12. 返回数据
  end
end

@enduml
```
*(注意: 上图是对 `cache_strategy_flow.puml` 文件的可视化呈现)*

</rewritten_file>