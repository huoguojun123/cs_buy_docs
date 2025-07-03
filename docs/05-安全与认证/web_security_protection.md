# Web通用安全防护

除了身份认证和权限控制，Web应用还面临着一系列常见的安全威胁。本章集中讨论项目如何配置和实现针对这些威胁的防护措施。

## 1. 跨域资源共享 (CORS)

由于浏览器同源策略的限制，当前端应用（如 `http://localhost:8080`）需要访问不同源的后端API（如 `http://api.hy.com`）时，就需要CORS机制的介入。

本项目在 **API网关 (`gateway-service`)** 层面进行了统一的CORS配置，这是推荐的最佳实践，可以避免在每个微服务中重复配置。

```java
// CorsConfig.java in gateway-service
@Configuration
public class CorsConfig {
    
    @Bean
    public CorsWebFilter corsWebFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        CorsConfiguration config = new CorsConfiguration();
        
        // 1. 允许携带凭证 (Credentials)，如Cookie
        config.setAllowCredentials(true);
        
        // 2. 允许的来源。生产环境应替换为具体的前端域名
        // config.addAllowedOrigin("http://your-frontend-domain.com");
        config.addAllowedOriginPattern("*"); // 开发阶段使用通配符
        
        // 3. 允许的请求头
        config.addAllowedHeader("*");
        
        // 4. 允许的HTTP方法
        config.addAllowedMethod("*");
        
        source.registerCorsConfiguration("/**", config);
        return new CorsWebFilter(source);
    }
}
```

**安全提示:** 在生产环境中，`allowedOriginPattern` 必须配置为具体的前端域名列表，使用 `*` 会带来安全风险。

## 2. 跨站脚本攻击 (XSS) 防护

XSS (Cross-Site Scripting) 攻击是指攻击者在网页中注入恶意脚本，当其他用户浏览该网页时，脚本会被执行，从而可能导致会话劫持、数据窃取等。

**核心防护原则：** 不信任任何用户输入，并对所有输出到页面的数据进行编码。

### 2.1 输入验证 (Input Validation)

在接收到用户提交的数据时（例如，商品描述、用户评论等），应在服务端进行严格的验证和过滤，移除危险的HTML标签和JavaScript事件。可以使用成熟的第三方库来完成这个任务。

**示例 (使用 OWASP Java HTML Sanitizer):**
```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.googlecode.owasp-java-html-sanitizer</groupId>
    <artifactId>owasp-java-html-sanitizer</artifactId>
    <version>20211018.2</version>
</dependency>
```
```java
// SanitizerUtil.java
import org.owasp.html.PolicyFactory;
import org.owasp.html.Sanitizers;

public class SanitizerUtil {
    // 定义一个策略，只允许加粗、斜体等基本文本格式
    private static final PolicyFactory POLICY = Sanitizers.FORMATTING.and(Sanitizers.BLOCKS);

    public static String sanitize(String untrustedHTML) {
        return POLICY.sanitize(untrustedHTML);
    }
}

// 在Service层使用
@Service
public class ReviewServiceImpl implements ReviewService {
    public void addReview(ReviewDTO reviewDTO) {
        // 在保存到数据库前，对用户评论内容进行净化
        String cleanContent = SanitizerUtil.sanitize(reviewDTO.getContent());
        review.setContent(cleanContent);
        reviewRepository.save(review);
    }
}
```

### 2.2 输出编码 (Output Encoding)

在将数据显示到前端页面时，应使用模板引擎或框架提供的功能，对数据进行HTML编码。这能确保即使用户输入了 `<script>` 标签，浏览器也只会将其作为普通文本展示，而不会执行它。

**示例 (Vue.js):**
Vue.js 的 `{{ }}` (Mustache 语法) 和 `v-text` 指令默认就会对内容进行HTML编码，是安全的。
```html
<!-- 安全的: 会将 <script>alert('xss')</script> 作为纯文本显示 -->
<div>{{ userInput }}</div>

<!-- 危险的! v-html 会直接渲染HTML，可能导致XSS攻击 -->
<div v-html="userInput"></div> 
```
应极力避免使用 `v-html`，除非你完全确定内容是安全可信的。

## 3. 跨站请求伪造 (CSRF) 防护

CSRF (Cross-Site Request Forgery) 攻击是指攻击者诱导一个已登录的用户，在不知情的情况下点击恶意链接或访问恶意网站，从而以用户的名义向目标网站发起非预期的请求（如转账、删除帖子等）。

### 3.1 防护机制分析

由于本项目核心认证机制为JWT，并且 **令牌通过 `Authorization` 请求头进行传递**，这种方式天然地对传统的、基于Cookie的CSRF攻击具有免疫力。

*   **传统CSRF攻击原理:** 依赖于浏览器在发起跨站请求时 **自动携带目标站点的Cookie**。
*   **JWT在请求头中的优势:** 浏览器不会自动在跨站请求中添加 `Authorization` 头。这个头必须由前端JavaScript代码显式地添加。恶意网站的脚本由于同源策略的限制，无法读取到存储在其他域的JWT，也无法为跨域请求伪造 `Authorization` 头。

因此，只要项目严格遵守 **将JWT存储在 `localStorage` 或 `sessionStorage` 中，并通过JS添加到请求头** 的规范，就不需要像传统Web应用那样引入CSRF Token等复杂的防护机制。

### 3.2 Cookie存储JWT的风险

如果项目出于某种原因选择将JWT存储在Cookie中，则CSRF风险依然存在。此时必须采取额外措施：
*   **设置 `SameSite` 属性:** 将JWT Cookie的 `SameSite` 属性设置为 `Strict` 或 `Lax`，可以有效阻止浏览器在跨站请求中发送该Cookie。
    *   `SameSite=Strict`: 最严格，完全禁止第三方携带。
    *   `SameSite=Lax`: 允许GET等顶层导航请求携带，但POST等会修改数据的请求会被阻止。 