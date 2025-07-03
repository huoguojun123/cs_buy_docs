# 4. 分布式事务：保障微服务数据一致性

在微服务架构中，一个单一的业务操作（如"创建订单"）常常需要跨越多个独立的服务（交易服务、库存服务、用户服务等），每个服务都有自己的数据库。如何保证这些跨库操作要么全部成功，要么全部失败，是维持数据一致性的核心挑战。本项目采用阿里巴巴开源的 **Seata** 作为分布式事务解决方案。

## 4.1 Seata 核心架构

Seata 通过协调者、管理器和资源管理器三个核心组件的协作来实现分布式事务控制。

*   **TC (Transaction Coordinator) - 事务协调者:** 独立部署的服务，作为全局事务的"大脑"，负责维护全局事务的状态，并决定最终是提交还是回滚。
*   **TM (Transaction Manager) - 事务管理器:** 嵌入在业务服务中，用于定义一个全局事务的开始、结束，并向TC上报最终状态。在代码中，`@GlobalTransactional` 注解的切面就是TM。
*   **RM (Resource Manager) - 资源管理器:** 同样嵌入在业务服务中，负责管理本地事务（分支事务），并与TC通信，根据TC的指令来提交或回滚分支事务。

## 4.2 AT 模式：无侵入的自动补偿方案

AT (Automatic Transaction) 模式是 Seata 的主打方案，它基于两阶段提交协议，并实现了对业务的"无侵入"，开发者只需关注自己的业务SQL。

### 4.2.1 AT 模式工作原理

AT 模式的精髓在于 **自动生成反向补偿SQL**。

1.  **阶段一 (执行):**
    *   TM 开启一个全局事务，并将 XID（全局事务ID）通过上下文传播到整个调用链。
    *   每个服务的 RM 会代理其数据源。当执行业务SQL（如 `UPDATE product SET stock = stock - 1`）时，RM会：
        a.  **解析SQL:** 获取表名、条件等信息。
        b.  **查询前置镜像:** 在执行业务SQL前，根据条件查询出要修改数据的当前样貌。
        c.  **执行业务SQL:** 正常执行业务的 `UPDATE` 操作。
        d.  **查询后置镜像:** 在执行业务SQL后，再次查询数据的样貌。
        e.  **插入 `undo_log`:** 将前置和后置镜像、业务SQL等信息组合成一条 `undo_log` 记录，并与业务SQL在同一个 **本地事务** 中提交。这条 `undo_log` 意味着："我知道如何撤销刚才的操作"。
    *   每个服务的分支事务执行成功后，都会向 TC 注册并报告OK。

2.  **阶段二 (提交/回滚):**
    *   如果所有分支事务都成功，TM 通知 TC **全局提交**。TC 会异步地通知所有 RM 删除对应的 `undo_log` 记录，完成事务。
    *   如果任何一个分支事务失败，TM 通知 TC **全局回滚**。TC 会通知所有 **已成功** 的 RM，要求它们进行回滚。RM 收到指令后，会找到对应的 `undo_log` 记录，利用其中的"前置镜像"生成反向的补偿SQL（如 `UPDATE product SET stock = stock + 1`）来恢复数据，然后删除 `undo_log`。

### 4.2.2 订单场景下的 AT 模式应用

在"创建订单"的场景中，交易服务需要调用库存服务扣减库存、调用用户服务扣减余额。

```java
// TradeServiceImpl.java
@Override
@GlobalTransactional(name = "create-order-tx", rollbackFor = Exception.class)
public Result<Long> createTrade(TradeDTO tradeDTO) {
    // 1. (RM-Trade) 在交易库创建订单记录
    tradeMapper.insert(trade);
    
    // 2. (RM-Stock) 调用库存服务，扣减库存
    // Feign拦截器会自动传递XID
    stockClient.deductStock(tradeDTO.getProductId(), tradeDTO.getQuantity());
    
    // 3. (RM-User) 调用用户服务，扣减余额
    userClient.deductBalance(tradeDTO.getUserId(), tradeDTO.getAmount());

    // 模拟一个异常，触发全局回滚
    // if (true) { throw new RuntimeException("mock exception"); }
    
    return Result.success(trade.getId());
}
```
开发者只需在一个入口方法上标注 `@GlobalTransactional`，Seata框架会自动处理后续所有分支事务的提交和回滚，无需关心底层细节。

## 4.3 TCC 模式：业务侵入的手动补偿方案

TCC (Try-Confirm-Cancel) 模式提供了更高的灵活性，但代价是需要对业务代码进行改造，手动实现三个阶段的逻辑。它适用于需要防止资源被长时间锁定，或需要集成非关系型数据库（如 Redis）等AT模式无法覆盖的场景。

### 4.3.1 TCC 模式工作原理

TCC 将一个业务操作拆分为三个独立的方法：
*   **Try:** 尝试执行业务，**预留业务资源**。例如，不是直接扣减库存，而是将主库存中的数量转移到"冻结库存"中。此阶段只做检查和资源预留。
*   **Confirm:** 如果所有服务的 `Try` 阶段都成功，则执行 `Confirm` 逻辑，真正地完成业务操作。例如，将"冻结库存"清空。
*   **Cancel:** 如果任何一个服务的 `Try` 阶段失败，则执行 `Cancel` 逻辑，**释放预留的资源**。例如，将"冻结库存"中的数量返还给主库存。

### 4.3.2 使用用户积分场景下的 TCC 模式应用

假设下单时允许用户使用积分抵扣金额，积分存储在 Redis 中。

1.  **定义 TCC 接口 (`PointTccAction`):**

    ```java
    @LocalTCC
    public interface PointTccAction {
        @TwoPhaseBusinessAction(name = "pointTccAction", commitMethod = "confirm", rollbackMethod = "cancel")
        boolean tryDeduct(BusinessActionContext context, @BusinessActionContextParameter(paramName = "userId") String userId, @BusinessActionContextParameter(paramName = "points") int points);

        boolean confirm(BusinessActionContext context);
        
        boolean cancel(BusinessActionContext context);
    }
    ```

2.  **实现 TCC 接口 (`PointTccActionImpl`):**

    ```java
    @Component
    public class PointTccActionImpl implements PointTccAction {
        @Autowired
        private StringRedisTemplate redisTemplate;

        @Override
        @Transactional
        public boolean tryDeduct(BusinessActionContext context, String userId, int points) {
            // Try: 检查积分是否足够，并预留（冻结）积分
            int availablePoints = Integer.parseInt(redisTemplate.opsForValue().get("points:available:" + userId));
            if (availablePoints < points) {
                throw new IllegalStateException("积分不足");
            }
            // 将积分从可用转移到冻结
            redisTemplate.opsForValue().increment("points:available:" + userId, -points);
            redisTemplate.opsForValue().increment("points:frozen:" + userId, points);
            // 将事务ID存入Redis，用于防悬挂
            redisTemplate.opsForValue().set("tcc:xid:" + context.getXid(), "trying");
            return true;
        }

        @Override
        @Transactional
        public boolean confirm(BusinessActionContext context) {
            // Confirm: 清理冻结的积分，正式完成扣减
            String xidStatus = redisTemplate.opsForValue().get("tcc:xid:" + context.getXid());
            if (xidStatus == null) { return true; } // 幂等性：Cancel已执行，直接返回

            int points = Integer.parseInt(context.getActionContext("points").toString());
            String userId = context.getActionContext("userId").toString();
            redisTemplate.opsForValue().increment("points:frozen:" + userId, -points);
            // 可在此处记录积分消费日志...
            
            // 删除XID标记
            redisTemplate.delete("tcc:xid:" + context.getXid());
            return true;
        }

        @Override
        @Transactional
        public boolean cancel(BusinessActionContext context) {
            // Cancel: 释放冻结的积分
            String xidStatus = redisTemplate.opsForValue().get("tcc:xid:" + context.getXid());
            if (xidStatus == null) { return true; } // 幂等性：Confirm已执行或Try未执行，直接返回

            int points = Integer.parseInt(context.getActionContext("points").toString());
            String userId = context.getActionContext("userId").toString();
            // 将积分从冻结返还到可用
            redisTemplate.opsForValue().increment("points:frozen:" + userId, -points);
            redisTemplate.opsForValue().increment("points:available:" + userId, points);
            
            // 删除XID标记
            redisTemplate.delete("tcc:xid:" + context.getXid());
            return true;
        }
    }
    ```
通过这种方式，TCC模式将补偿逻辑交由业务代码自行控制，提供了更高的自由度。

## 4.4 分布式锁：并发控制的补充

无论是 AT 还是 TCC 模式，Seata 解决的是**事务原子性**问题，而不是**并发隔离**问题。在高并发场景下，多个事务同时操作同一资源（如同一件商品的库存）时，仍需要分布式锁来保证数据的一致性。

例如，在扣减库存的 `tryDeduct` 方法中，"读-改-写" 的操作就不是原子的，需要使用分布式锁（如 Redisson）包裹起来，确保同一时间只有一个事务能操作该商品的库存。 