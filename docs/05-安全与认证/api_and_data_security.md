# API与数据安全加固

除了认证、授权和Web通用防护之外，保障API接口自身的健壮性和后端数据的存储安全也至关重要。

## 1. 敏感数据加密

为了防止因数据库泄露而导致的用户敏感信息裸奔，系统对数据库中存储的关键字段进行了加密处理。

### 1.1 密码加密

**策略：** 使用 **BCrypt** 算法对用户密码进行哈希存储。

BCrypt 是一种专为密码存储设计的单向哈希算法，它的特点是：
*   **加盐 (Salting):** 自动为每个密码生成一个随机盐（salt），并将其与哈希结果存在一起。这使得即使两个用户设置了相同的密码，他们在数据库中的哈希值也是不同的，有效抵御彩虹表攻击。
*   **慢速 (Slow):** BCrypt 的计算速度被故意设计得很慢，可以通过调整"计算轮次"（work factor）来增加破解单个密码所需的时间成本，有效对抗暴力破解。

**实现:**
```java
// SecurityBeanConfig.java
@Configuration
public class SecurityBeanConfig {
    
    // 将BCryptPasswordEncoder注入到Spring容器
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}

// UserServiceImpl.java (用户注册时)
@Autowired
private PasswordEncoder passwordEncoder;

public void register(UserDTO userDTO) {
    // ...
    // 使用注入的encoder对明文密码进行加密
    String hashedPassword = passwordEncoder.encode(userDTO.getRawPassword());
    user.setPassword(hashedPassword);
    userRepository.save(user);
}

// UserDetailsServiceImpl.java (用户登录验证时)
// Spring Security会自动使用该encoder来比对用户输入的密码和数据库中的哈希值
boolean matches = passwordEncoder.matches(rawPassword, hashedPasswordFromDB);
```

### 1.2 其他敏感信息加密 (如手机号)

对于手机号、身份证号、家庭住址等信息，它们需要在某些业务场景下被解密并展示。因此不能使用单向哈希，而应采用 **对称加密（如 AES）**。

**策略：**
1.  在服务端的安全位置（如配置中心、环境变量）保存一个全局的、高强度的AES密钥。
2.  在数据持久化到数据库之前，通过AOP切面或自定义的JPA转换器 (`AttributeConverter`)，自动对敏感字段进行加密。
3.  在从数据库读取数据后，自动进行解密。

这种对业务代码透明的加解密方式是最佳实践，可以避免在各个业务逻辑中重复编写加解密代码。

## 2. API 接口安全策略

### 2.1 强制 HTTPS

在生产环境中，必须通过反向代理（如 Nginx）或API网关的配置，将所有HTTP请求重定向到HTTPS。这能确保客户端与服务器之间的通信信道是加密的，有效防止中间人攻击和数据窃听。

### 2.2 API 防刷 (Rate Limiting)

为了防止恶意用户通过脚本高频调用API（例如，发送短信验证码接口、抽奖接口等），需要在API网关层实现请求频率限制。

**实现方式：**
*   **Sentinel:** 作为项目技术栈的一部分，Sentinel 提供了强大的流量控制功能，可以基于IP、用户ID、API路径等多种维度配置QPS（每秒查询率）限制。
*   **Resilience4j:** 也是一个流行的容错库，其 `RateLimiter` 模块可以实现类似的功能。

### 2.3 防止API参数篡改 (数字签名)

对于一些极其重要的操作，特别是支付场景，为防止请求在从客户端到网关的过程中被篡改（例如，修改支付金额），可以引入API签名机制。

**流程:**
1.  **客户端:**
    *   将所有请求参数（包括一个随机数`nonce`和时间戳`timestamp`）按key进行字典序排序。
    *   将排序后的键值对拼接成一个字符串。
    *   将该字符串与一个客户端与服务端预共享的密钥 (`client_secret`) 拼接。
    *   计算最终字符串的哈希值（如 SHA256），生成 `sign`。
    *   将 `sign` 和所有参数一起发送给服务端。
2.  **服务端 (API网关):**
    *   接收到请求后，以完全相同的方式重新计算一遍签名。
    *   比对客户端传来的 `sign` 和自己计算的 `sign` 是否一致。如果不一致，则拒绝请求。
    *   检查 `timestamp` 是否在合理的时间窗口内（如1分钟内），并使用Redis记录已处理的 `nonce`，以防止请求重放攻击。 