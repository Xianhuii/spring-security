# Spring Security Web模块核心功能与使用步骤

---

## 一、核心功能梳理

1. **请求过滤链（FilterChainProxy）**
   - 所有 HTTP 请求统一经过安全过滤器链，按配置顺序依次处理安全相关逻辑。

2. **认证（Authentication）**
   - 支持多种认证方式：表单登录（UsernamePasswordAuthenticationFilter）、HTTP Basic、RememberMe、匿名认证等。
   - 认证信息通过 AuthenticationManager（通常由 core 模块实现）校验，并存入 SecurityContextHolder。

3. **授权（Authorization）**
   - 通过 AuthorizationFilter（或旧版 FilterSecurityInterceptor）结合 AuthorizationManager 判断用户是否有权访问资源。
   - 支持基于 URL、方法、表达式等多种授权方式。

4. **CSRF 防护**
   - 默认启用 CSRF 过滤器，防止跨站请求伪造攻击。

5. **Session 管理**
   - 支持会话并发控制、会话失效处理、Session Fixation 防护等。

6. **异常处理**
   - 通过 ExceptionTranslationFilter 统一处理认证失败、授权失败等异常，支持自定义跳转或响应。

7. **安全头部管理**
   - 自动添加常见安全 HTTP 头（如 X-Frame-Options、X-XSS-Protection 等）。

8. **多环境适配**
   - 支持 Servlet、Reactive(WebFlux)、Server 等多种 Web 环境。

---

## 二、典型使用步骤

### 1. 引入依赖

在 `build.gradle` 或 `pom.xml` 中引入 spring-security-web 及相关依赖。

### 2. 配置安全过滤链

**方式一：Java Config（推荐）**
```java
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().and()
            .authorizeHttpRequests((authz) -> authz
                .antMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin().and()
            .httpBasic();
        return http.build();
    }
}
```

**方式二：XML 配置（老项目）**
```xml
<http>
    <csrf/>
    <intercept-url pattern="/public/**" access="permitAll"/>
    <intercept-url pattern="/**" access="isAuthenticated()"/>
    <form-login/>
    <http-basic/>
</http>
```

### 3. 认证流程

- 用户访问受保护资源时，未认证会被重定向到登录页或弹出 Basic 认证框。
- 登录表单提交后，UsernamePasswordAuthenticationFilter 拦截请求，调用 AuthenticationManager 认证。
- 认证成功后，信息写入 SecurityContextHolder，后续请求自动携带认证信息。

### 4. 授权流程

- 每次请求，AuthorizationFilter 会根据配置和当前用户权限判断是否允许访问。
- 授权失败会抛出异常，由 ExceptionTranslationFilter 处理（如跳转到 403 页面）。

### 5. 其他安全功能

- 默认启用 CSRF 防护，表单需携带 CSRF Token。
- 可配置 Session 策略、RememberMe、异常处理、定制安全头等。

---

## 三、常见扩展点

- **自定义认证逻辑**：实现 AuthenticationProvider 并注册到 AuthenticationManager。
- **自定义授权规则**：实现 AuthorizationManager 或使用表达式扩展。
- **自定义过滤器**：可在 SecurityFilterChain 中插入自定义 Filter。
- **自定义异常处理**：实现 AuthenticationEntryPoint、AccessDeniedHandler 等。

---

如需具体某一功能的详细用法或代码示例，请告知！ 