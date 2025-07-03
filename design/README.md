# 需求分析与功能设计 (UML)

本目录存放项目在需求分析和功能设计阶段产出的UML图表。这些图表遵循标准UML规范，并根据"高内聚、低耦合"的原则进行拆分，旨在从不同维度清晰地描绘系统的功能、结构和行为。

## UML 图表索引

### 1. 用例图 (Use Case Diagram)

-   **文件**: `01-use-case-diagram.puml`
-   **描述**: 此图从高层次展示了系统的核心参与者（Actors），包括游客、注册用户和管理员，以及他们可以执行的主要功能用例（Use Cases）。它定义了系统的范围和主要交互点。

![用例图](01-use-case-diagram.puml)

---
## 架构与静态结构
### 2. 组件图 (Component Diagram) - 后端服务
-   **文件**: `17-component-diagram-backend-services.puml`
-   **描述**: 宏观展示了后端微服务的组成，以及它们与数据库、中间件之间的静态依赖关系。

![组件图](17-component-diagram-backend-services.puml)

### 3. 状态机图 (State Machine Diagram) - 订单生命周期
-   **文件**: `18-statemachine-diagram-order-lifecycle.puml`
-   **描述**: 精确地描绘了"订单"对象从创建到完成或取消的完整生命周期和状态变迁。

![订单状态机图](18-statemachine-diagram-order-lifecycle.puml)

### 4. 类图 (Class Diagram) - 用户领域
-   **文件**: `04-class-diagram-user-domain.puml`
-   **描述**: 定义了与用户身份和个人信息相关的核心实体，如 `User` 和 `Address`。

![用户领域类图](04-class-diagram-user-domain.puml)

### 5. 类图 (Class Diagram) - 商品领域
-   **文件**: `07-class-diagram-product-domain.puml`
-   **描述**: 定义了与商品、分类、品牌和评价相关的实体，构成了商品中心的核心模型。

![商品领域类图](07-class-diagram-product-domain.puml)

### 6. 类图 (Class Diagram) - 订单领域
-   **文件**: `08-class-diagram-order-domain.puml`
-   **描述**: 定义了构成一笔交易的核心实体，如 `Order` 和 `OrderItem`，是交易流程的基础。

![订单领域类图](08-class-diagram-order-domain.puml)

---
## 核心用户与认证流程
### 7. 时序图 (Sequence Diagram) - 用户登录

-   **文件**: `03-sequence-diagram-user-login.puml`
-   **描述**: 此图聚焦于用户登录这一具体场景，展示了服务间的交互细节和认证流程。

![登录时序图](03-sequence-diagram-user-login.puml)

### 8. 时序图 (Sequence Diagram) - JWT 续签

-   **文件**: `16-sequence-diagram-jwt-refresh.puml`
-   **描述**: 描绘了当AccessToken过期后，客户端使用RefreshToken安全地获取新AccessToken的独立流程。

![JWT续签时序图](16-sequence-diagram-jwt-refresh.puml)

### 9. 活动图 (Activity Diagram) - 用户注册流程

-   **文件**: `10-activity-diagram-user-registration.puml`
-   **描述**: 描绘了新用户从获取验证码到成功创建账户的完整步骤。

![用户注册活动图](10-activity-diagram-user-registration.puml)

### 10. 时序图 (Sequence Diagram) - 商品搜索

-   **文件**: `11-sequence-diagram-product-search.puml`
-   **描述**: 展示了用户进行商品搜索时，前端、搜索服务及Elasticsearch之间的交互。

![商品搜索时序图](11-sequence-diagram-product-search.puml)

---

## 交易核心流程

### 11. 时序图 (Sequence Diagram) - 创建订单

-   **文件**: `15-sequence-diagram-create-order.puml`
-   **描述**: 展示了创建订单时，订单、商品、库存等微服务间的协作。

![创建订单时序图](15-sequence-diagram-create-order.puml)


### 12. 活动图 (Activity Diagram) - 购物车与结算

-   **文件**: `02-activity-diagram-cart-and-checkout.puml`
-   **描述**: 描绘了用户将商品加入购物车、在购物车中管理商品，并发起结算的流程。

![购物车与结算活动图](02-activity-diagram-cart-and-checkout.puml)

### 13. 活动图 (Activity Diagram) - 订单创建

-   **文件**: `05-activity-diagram-order-creation.puml`
-   **描述**: 聚焦于用户发起结算后的后端处理逻辑，包括验证、计算、创建订单并锁定库存。

![订单创建活动图](05-activity-diagram-order-creation.puml)

### 14. 活动图 (Activity Diagram) - 支付处理

-   **文件**: `06-activity-diagram-payment-processing.puml`
-   **描述**: 详细说明了与第三方支付网关的交互、异步回调的处理，以及后续的订单状态更新和库存变更。

![支付处理活动图](06-activity-diagram-payment-processing.puml)

---
## 业务管理流程
### 15. 活动图 (Activity Diagram) - 后台新增商品
-   **文件**: `09-activity-diagram-admin-product-management.puml`
-   **描述**: 描绘了管理员在后台进行商品新增的独立工作流。

![后台新增商品活动图](09-activity-diagram-admin-product-management.puml)

### 16. 活动图 (Activity Diagram) - 后台编辑商品
-   **文件**: `13-activity-diagram-admin-edit-product.puml`
-   **描述**: 描绘了管理员在后台编辑现有商品的独立工作流。

![后台编辑商品活动图](13-activity-diagram-admin-edit-product.puml)

### 17. 活动图 (Activity Diagram) - 库存管理
-   **文件**: `14-activity-diagram-inventory-management.puml`
-   **描述**: 描绘了不同业务场景（如下单、支付、取消、退货）如何影响库存变化。

![库存管理活动图](14-activity-diagram-inventory-management.puml)

### 18. 活动图 (Activity Diagram) - 售后退款流程
-   **文件**: `12-activity-diagram-refund-process.puml`
-   **描述**: 描绘了从用户申请退款，到客服审核、仓库收货、财务退款的完整闭环。

![售后退款流程活动图](12-activity-diagram-refund-process.puml) 