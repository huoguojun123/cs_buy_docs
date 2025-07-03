# 项目部署架构

项目采用基于容器的分布式部署架构，以实现高可用、高并发和弹性伸缩。整个系统被划分为不同的层次，各层之间职责分明。

## 架构图

以下是系统部署的整体架构图，展示了从用户请求到后端服务的完整流程。

```mermaid
graph TD
    subgraph "用户访问"
        DNS
    end

    subgraph "负载均衡"
        LB[Nginx / 云服务SLB]
    end
    
    subgraph "公网暴露层"
        Frontend[前端静态资源服务器 Nginx]
        Gateway[API网关集群]
    end

    subgraph "内部服务层 (Kubernetes集群)"
        direction LR
        subgraph "Node 1"
            P1[Pod: 用户服务]
            P2[Pod: 商品服务]
        end
        subgraph "Node 2"
            P3[Pod: 订单服务]
            P4[Pod: 支付服务]
        end
        subgraph "Node n"
            P5[...]
        end
    end

    subgraph "基础设施与中间件"
        Nacos[Nacos集群]
        Seata[Seata Server集群]
        Redis[Redis集群]
        MySQL[MySQL主从]
        RocketMQ[RocketMQ集群]
        Elasticsearch[Elasticsearch集群]
        SkyWalking[SkyWalking OAP]
    end
    
    subgraph "可观测性平台"
        Prometheus[Prometheus]
        Grafana[Grafana]
        Alertmanager[Alertmanager]
    end

    DNS --> LB
    LB --> Frontend
    LB --> Gateway

    Gateway --> P1
    Gateway --> P2
    Gateway --> P3
    Gateway --> P4

    %% 服务间依赖
    P1 & P2 & P3 & P4 --> Nacos:"服务发现/配置"
    P1 & P2 & P3 & P4 --> Seata:"分布式事务协调"
    P1 & P2 & P3 & P4 --> Redis:"缓存"
    P1 & P2 & P3 & P4 --> MySQL:"数据存储"
    P1 & P2 & P3 & P4 --> RocketMQ:"消息队列"
    P1 & P2 & P3 & P4 --> Elasticsearch:"搜索/日志"
    P1 & P2 & P3 & P4 --> SkyWalking:"链路追踪"

    %% 可观测性
    Prometheus --> P1 & P2 & P3 & P4 & Nacos & Redis & MySQL:"指标采集"
    Grafana --> Prometheus:"指标可视化"
    Prometheus --> Alertmanager:"触发告警"
```

## 各层详解

-   **用户访问层**: 用户的请求首先通过DNS解析到负载均衡器的IP地址。

-   **负载均衡层 (Load Balancer)**: 使用`Nginx`或云服务商提供的`SLB`（Server Load Balancer）作为整个系统的流量入口。它负责将公网流量根据请求的域名或路径，分发到后端的不同服务上，同时实现SSL卸载和健康检查。

-   **公网暴露层 (Public-Facing Services)**:
    -   **前端静态资源**: `e-business-front`项目被`Vite`打包成静态文件（HTML, CSS, JS），由一个独立的`Nginx`服务器托管。CDN可以部署在这一层之前，用于加速静态资源的全球分发。
    -   **API网关集群**: `gateway-service`以集群模式部署，是所有后端微服务的统一入口。它处理认证、路由、限流、熔断等逻辑。

-   **内部服务层 (Internal Services)**:
    -   所有后端微服务（如用户服务、商品服务等）都被打包成**Docker镜像**。
    -   使用**Kubernetes (K8s)** 作为容器编排平台，对这些无状态的服务进行部署、管理、扩缩容和自愈。服务实例以`Pod`的形式运行在不同的`Node`（物理机或虚拟机）上。

-   **基础设施与中间件 (Infrastructure & Middleware)**:
    -   这些是支撑业务运行的关键组件，全部以**高可用集群模式**部署，以避免单点故障。
    -   **Nacos**: 服务注册与发现中心、配置中心。
    -   **Redis**: 分布式缓存、分布式锁。
    -   **MySQL**: 主业务数据库，采用主从复制架构实现读写分离和高可用。
    -   **RocketMQ**: 异步消息队列，用于服务解耦、流量削峰、最终一致性事务。
    -   **Elasticsearch**: 商品搜索、日志存储。
    -   **Seata**: 分布式事务解决方案协调者。
    -   **SkyWalking**: 分布式链路追踪系统。

-   **可观测性平台 (Observability Platform)**:
    -   **Prometheus**: 负责从各个服务和中间件拉取（Pull）或接收（Push）性能指标（Metrics）。
    -   **Grafana**: 将Prometheus收集的指标数据进行可视化，生成监控大盘。
    -   **Alertmanager**: 接收来自Prometheus的告警，并负责对告警进行分组、抑制，最终通过邮件、钉钉等方式发送通知。 