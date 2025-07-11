# Spring Security FilterChain 配置步骤与 Filter 组装机制详解

---

## 一、SecurityFilterChain 配置步骤

### 1. 配置入口

**Java Config 方式（推荐）**
```java
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        // 配置逻辑
        return http.build();
    }
}
```

**XML 方式（老项目）**
```xml
<http>
    <!-- 配置逻辑 -->
</http>
```

### 2. 配置内容

- **请求匹配规则**：通过 `securityMatcher()` 或 `antMatchers()` 定义哪些请求走这个链。
- **认证方式**：如 `formLogin()`、`httpBasic()`、`oauth2Login()` 等。
- **授权规则**：如 `authorizeHttpRequests()` 定义访问控制。
- **其他安全功能**：如 `csrf()`、`sessionManagement()`、`headers()` 等。

### 3. 构建过程

调用 `http.build()` 时，Spring Security 会：
1. 收集所有配置的 Filter。
2. 按预定义顺序组装成 Filter 链。
3. 创建 `DefaultSecurityFilterChain` 实例。

---

## 二、Filter 组装机制详解

### 1. Filter 顺序

Spring Security 定义了严格的 Filter 顺序，通过 `SecurityFilters` 枚举类定义：

```java
public enum SecurityFilters {
    FIRST(100),
    CHANNEL_FILTER(200),
    SECURITY_CONTEXT(300),
    CONCURRENT_SESSION(400),
    WEB_ASYNC_MANAGER_INTEGRATION(500),
    HEADERS(600),
    CSRF(700),
    LOGOUT(800),
    X509(900),
    PRE_AUTH(1000),
    CAS(1100),
    FORM_LOGIN(1200),
    OPENID(1300),
    LOGIN_PAGE(1400),
    HTTP_BASIC(1500),
    REQUEST_CACHE(1600),
    SERVLET_API_SUPPORT(1700),
    JAAS_API_SUPPORT(1800),
    REMEMBER_ME(1900),
    ANONYMOUS(2000),
    SESSION_MANAGEMENT(2100),
    EXCEPTION_TRANSLATION(2200),
    AUTHORIZATION(2300),
    SWITCH_USER(2400),
    LAST(2500);
}
```

### 2. 组装过程

**步骤 1：配置收集**
```java
// 当你调用 http.formLogin() 时
http.formLogin()
    .and()
    .authorizeHttpRequests()
    .and()
    .csrf();
```

**步骤 2：Filter 创建**
每个配置方法会创建对应的 Filter：
- `formLogin()` → `UsernamePasswordAuthenticationFilter`
- `authorizeHttpRequests()` → `AuthorizationFilter`
- `csrf()` → `CsrfFilter`
- `httpBasic()` → `BasicAuthenticationFilter`

**步骤 3：顺序组装**
```java
// 伪代码展示组装过程
List<Filter> filters = new ArrayList<>();
filters.add(new CsrfFilter());           // 顺序 700
filters.add(new BasicAuthenticationFilter()); // 顺序 1500
filters.add(new UsernamePasswordAuthenticationFilter()); // 顺序 1200
filters.add(new AuthorizationFilter()); // 顺序 2300
// 按 SecurityFilters 定义的顺序排序
Collections.sort(filters, SecurityFiltersOrderComparator);
```

**步骤 4：链创建**
```java
// 创建 DefaultSecurityFilterChain
SecurityFilterChain chain = new DefaultSecurityFilterChain(
    requestMatcher,  // 请求匹配器
    filters          // 排序后的过滤器列表
);
```

### 3. 源码关键点

**HttpSecurity.build() 方法**
```java
public SecurityFilterChain build() throws Exception {
    // 1. 应用所有配置器
    for (SecurityConfigurer<Filter, HttpSecurity> configurer : this.configurers) {
        configurer.configure(this);
    }
    
    // 2. 收集所有 Filter
    List<Filter> filters = new ArrayList<>();
    for (Filter filter : this.filters) {
        filters.add(filter);
    }
    
    // 3. 排序
    AnnotationAwareOrderComparator.sort(filters);
    
    // 4. 创建链
    return new DefaultSecurityFilterChain(this.requestMatcher, filters);
}
```

---

## 三、实际配置示例

### 1. 基础配置
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests((authz) -> authz
            .anyRequest().authenticated()
        )
        .formLogin()
        .csrf();
    return http.build();
}
```

**生成的 Filter 链（按顺序）**：
1. `CsrfFilter` - CSRF 防护
2. `UsernamePasswordAuthenticationFilter` - 表单登录
3. `AuthorizationFilter` - 授权检查

### 2. 复杂配置
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests((authz) -> authz
            .anyRequest().authenticated()
        )
        .formLogin()
        .httpBasic()
        .csrf()
        .sessionManagement()
        .headers();
    return http.build();
}
```

**生成的 Filter 链**：
1. `HeaderWriterFilter` - 安全头
2. `CsrfFilter` - CSRF 防护
3. `BasicAuthenticationFilter` - HTTP Basic
4. `UsernamePasswordAuthenticationFilter` - 表单登录
5. `SessionManagementFilter` - 会话管理
6. `AuthorizationFilter` - 授权检查

---

## 四、自定义 Filter 插入

### 1. 在指定位置插入
```java
http.addFilterAt(new CustomFilter(), SecurityFilters.AUTHORIZATION);
```

### 2. 在指定 Filter 前后插入
```java
http.addFilterBefore(new CustomFilter(), UsernamePasswordAuthenticationFilter.class);
http.addFilterAfter(new CustomFilter(), UsernamePasswordAuthenticationFilter.class);
```

---

## 五、调试技巧

### 1. 查看 Filter 链
```java
@EventListener
public void handleFilterChainBuilt(FilterChainBuiltEvent event) {
    SecurityFilterChain chain = event.getSecurityFilterChain();
    System.out.println("Filters: " + chain.getFilters());
}
```

### 2. 开启 Debug 日志
```properties
logging.level.org.springframework.security=DEBUG
```

---

## 六、多 SecurityFilterChain 配置

### 1. 同时配置多个链
```java
@Configuration
public class MultiSecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().hasRole("API_USER")
            )
            .httpBasic();
        return http.build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain formLoginChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().authenticated()
            )
            .formLogin();
        return http.build();
    }
}
```

### 2. 注意事项
- **顺序很重要**：Spring 会按 `@Order` 或 XML 配置顺序查找匹配的链，**只会应用第一个匹配的 SecurityFilterChain**。
- **每个链独立配置**：每个链可以有不同的认证方式、授权规则、CSRF 策略等，互不影响。
- **适合多入口场景**：如前后台分离、API 与 Web 管理后台共存、不同子系统不同安全策略等。

---

**总结**：SecurityFilterChain 的配置是通过 HttpSecurity 的 DSL 收集配置，然后按预定义顺序组装 Filter，最终创建 DefaultSecurityFilterChain 实例。整个过程是声明式的，开发者只需关注业务逻辑，Spring Security 负责底层的 Filter 组装和排序。

如需更详细的源码解读或实战案例，请告知！ 