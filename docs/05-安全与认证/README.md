# 5. 安全与认证

安全是电商平台的生命线。本章详细阐述了项目为保障用户账户、交易数据和系统自身安全而构建的多层次、纵深防御体系。这套体系从用户身份认证、操作权限控制，到Web常见漏洞防护，再到数据和API接口的安全加固，力求覆盖业务流程的每一个环节。

为了便于查阅，详细内容已拆分到以下独立的专题文档中：

*   **[JWT无状态认证机制](./jwt_authentication.md)**
    *   深入解析了系统如何使用 JSON Web Token (JWT) 实现高效、可扩展的无状态用户认证，包括令牌的生成、校验、传递以及在API网关的统一处理。

*   **[RBAC授权与权限控制](./rbac_authorization.md)**
    *   详细描述了基于角色的访问控制（RBAC）模型在项目中的应用，涵盖了用户、角色、权限的数据库设计，以及如何在业务代码中进行精细化的权限校验。

*   **[Web通用安全防护](./web_security_protection.md)**
    *   集中讨论了如何防范常见的Web安全漏洞，包括跨域资源共享（CORS）的正确配置、跨站脚本（XSS）的防御策略，以及跨站请求伪造（CSRF）的防护机制。

*   **[API与数据安全加固](./api_and_data_security.md)**
    *   探讨了针对API接口和敏感数据的安全强化措施，如强制HTTPS、API接口的防刷和防重放攻击策略，以及数据库中用户密码等敏感信息的加密存储方案。


## 核心认证流程概览

下图展示了用户登录并使用JWT令牌访问受保护资源的宏观流程。

```puml
@startuml
!theme vibrant
title JWT 认证与访问流程

actor "用户" as User
participant "客户端\n(浏览器/App)" as Client
participant "API网关\n(gateway-service)" as Gateway
participant "认证服务\n(user-service)" as AuthServer
participant "资源服务\n(如 trade-service)" as ResourceServer

User -> Client: 输入用户名/密码
Client -> Gateway: 1. /login 登录请求
Gateway -> AuthServer: 2. 转发登录请求

AuthServer -> AuthServer: 3. 验证凭证
alt 凭证有效
    AuthServer -> AuthServer: 4. 生成JWT令牌
    AuthServer --> Gateway: 5. 返回JWT
    Gateway --> Client: 6. 返回JWT
    Client -> Client: 7. 存储JWT (localStorage/Cookie)
else 凭证无效
    AuthServer --> Gateway: 返回认证失败
    Gateway --> Client: 返回认证失败
end

User -> Client: 操作需要权限的业务
Client -> Gateway: 8. /trades/my-orders (携带JWT)
Gateway -> Gateway: 9. **JWT过滤器**校验令牌签名和有效期
alt 令牌有效
    Gateway -> Gateway: 10. 解析JWT, 将用户信息(userId)放入请求头
    Gateway -> ResourceServer: 11. 转发已认证的请求
    ResourceServer -> ResourceServer: 12. (可选)根据请求头中的用户信息执行业务逻辑
    ResourceServer --> Gateway: 返回业务数据
    Gateway --> Client: 返回业务数据
else 令牌无效
    Gateway --> Client: 401 Unauthorized
end

@enduml
```
*(注意: 上图是对 `jwt_structure.puml` 文件的可视化呈现)*
