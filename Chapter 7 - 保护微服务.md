# Chapter 7 - 保护微服务

Created by : Mr Dk.

2020 / 08 / 19 23:15

@Nanjing, Jiangsu, China

---

如何验证调用服务的用户具有权限执行他们请求的操作？

## 7.1 OAuth 2 简介

OAuth 2 是一个基于 token 的安全验证和授权框架，分为以下四部分：

1. 受保护资源 - 比如一个微服务，只有经过授权的微服务才能调用这个微服务
2. 资源所有者 - 定义哪些应用可以调用受保护的微服务
3. 应用程序 - 代表用户调用微服务
4. OAuth 2 验证服务器 - 向用户颁发 token；或帮助微服务验证 token

OAuth 2 规范支持四种类型的授权：

- Password (密码)
- Client credential (客户端凭据)
- Authorization code (授权码)
- Implicit (隐式)

## 7.2 从小事做起：使用 Spring 和 OAuth 2 来保护单个端点

建立起一个 OAuth 2 的验证服务；另外建立一个已授权的应用程序。通过 OAuth 2 密码授权，这个应用程序的用户中只有验证通过的用户才能访问被保护的服务。

### 7.2.1 建立 OAuth 2 验证服务

OAuth 2 验证服务也是一个 Spring Boot 服务。这个服务将验证用户提交的凭据，并向用户颁发 token；当用户使用 token 调用被保护的服务时，验证服务将验证 token 是否有效。

对于验证服务，需要引入 Maven 依赖项，并在其引导类上添加注解 `@EnableAuthorizationServer`。

### 7.2.2 使用 OAuth 2 服务注册应用程序

需要定义哪些应用程序能够通过 OAuth 2 认证服务访问被保护的资源。首先，需要继承 `AuthorizationServerConfigurerAdapter` 类，并重写其中的 `configure()` 函数。这个函数中定义了哪些应用程序 (客户端) 能够访问被 OAuth 2 认证服务保护的服务。

```java
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    clients.inmemory()
        .withClient("eagleeye")
        .secret("thisissecret")
        .authorizedGrantTypes("refresh_token", "password", "client_credentials")
        .scopes("webclient", "mobileclient");
}
```

对应用程序的信息，这里支持内存存储和 JDBC 存储 - 代码中的 `inmemory()` 表示使用内存存储。`secret()` 中的密钥会在颁发 token 时被使用到。`authorizedGrantTypes()` 中定义了以逗号分隔的授权类型列表，`scopes()` 定义了获取 token 时允许的作用域。通过定义作用域，可以细化不同客户端访问服务时的规则。

### 7.2.3 配置 EagleEye 用户

需要定义个人用户的 **凭据** 和 **角色**。Spring 可以从内存、JDBC 关系数据库或 LDAP 服务器中存储和检索用户信息 (凭据 + 角色)。这里也使用内存存储。与上述类似，要继承 `WebSecurityConfigurer`，并重写 `configure()` 函数。

```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.inMemoryAuthentication()
        .withUser("john.carnell")
        .password("password1")
        .roles("User")
        .and()
        .withUser("william.woodward")
        .password("password2")
        .roles("USER", "ADMIN");
}
```

这里就类似数据库了。里面存储了两个用户的用户名和密码，以及他们所属的角色。

### 7.2.4 验证用户

在 OAuth 2 服务被配置并运行后，就能够开始执行用户验证与 token 颁发流程。这里由于认证服务已经被配置了用户的密码，所以用户可以直接使用密码授权方式进行认证，获取 token。

用户可以请求 OAuth 2 验证服务的 `/auth/oauth/token` end point，同时给出：

- `Authorization` 头部需要设置为应用程序的 ID 和密钥
- `grant_type` - OAuth 2 授权类型 (password 授权)
- `scope` - 应用程序的作用域
- `username` + `password`

除了 `Authorization` 头部，其余部分使用 HTTP 表单传递。在验证服务发回的响应中，将包含：

- `access_token` - OAuth 2 token，随用户每次访问受保护资源时一起出示
- `token_type` - Token 类型
- `refresh_token` - 用于在 token 过期后重新颁发 token
- `expires_in` - Token 过期前的秒数
- `scope` - Token 的有效作用域

有了有效的 token，用户可以直接请求验证服务的 `/auth/user`，并附带 token。认证服务将会确认 token，并检索用户信息。每个受保护的服务都是通过调用验证服务的 `/auth/user` 来确认 token 并检索用户信息。在向 `/auth/user` 发送请求时，需要创建 `Authorization` HTTP header，并设置值为 `Bearer <token>`。如果 token 有效，那么这个 end point 将会返回用户信息 (角色等)。

## 7.3 使用 OAuth 2 保护组织服务

目前，已经在验证服务中注册了应用程序，并创建了拥有角色的个人用户。在 Spring 中，定义哪个用户角色有权执行哪些操作是在 **单个服务级别** 上发生的。也就是说，要在单个服务中定义谁可以访问本服务。

### 7.3.1 将 Spring Security 和 OAuth 2 jar 添加到各个服务

向需要将自己保护起来的服务添加 Maven 依赖项。

### 7.3.2 配置服务以指向 OAuth 2 验证服务

当服务被创建为受保护资源，每次调用这个服务时，调用者必须将 OAuth 2 token 放置在 `Authorization` 头部中。在收到 token 后，受保护的服务必须调用 OAuth 2 验证服务来查看 token 是否有效。因此，需要在受保护服务的配置文件中，设置验证服务的 URL：

```yaml
security:
  oauth2:
    resource:
      userInfoUri: http://localhost:8901/auth/user
```

另外，还需要告诉服务自身是一个受保护资源。这是通过设置引导类注解 `@EnableResourceServer` 实现的。这个注解会强制执行一个过滤器，拦截对该服务的所有传入调用，检查其中是否包含 OAuth 2 token，然后调用 `userInfoUri` 来查看 token 是否有效。如果 token 有效，那么该注解将会进一步应用预定义的访问控制规则，以控制哪些人可以访问服务。

### 7.3.3 定义谁可以访问服务

定义访问规则的方式与之前类似：继承 `ResourceServerConfigurerAdapter`，并重写 `configure()` 函数。通过定义访问控制规则，可以使：

- 只有已通过验证的用户才能访问服务
- 只有具有特定角色的用户才能访问服务

#### 7.3.3.1 通过验证用户保护服务

仅限已通过身份验证的用户访问服务。

```java
@Configuration
public class ResourceServerConfiguration extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().anyRequest().authenticated();
    }
}
```

#### 7.3.3.2 通过特定角色保护服务

可以锁定对服务调用的 HTTP method 和 URL，仅限具有特定 **角色** 的用户访问。

```java
@Configuration
public class ResourceServerConfiguration extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.
            .authorizeRequests()
            .antMatchers(HttpMethod.DELETE, "/v1/organizations/**")
            .hasRole("ADMIN")
            .anyRequest()
            .authenticated();
    }
}
```

### 7.3.4 传播 OAuth 2 Token

在微服务环境中，会有多个服务调用来执行一个事务。需要保证 token 在服务调用之间被传播。假设一个用户已经获得了 token，所有服务运行在 Zuul 网关之后。那么接下来：

1. 应用程序在 `Authorization` HTTP header 中添加 token，访问服务
2. Zuul 查找服务 end point，并转发到其中的一个服务实例上 - 服务网关需要保证 `Authorization` header 也被转发
3. 受保护的服务实例接收到请求后，调用验证服务确认 token；之后，该服务可能需要调用另一个服务，该服务要保证 token 也在请求时一并发送
4. 另一个服务接收到请求后，再次向验证服务确认 token

在默认情况下，Zuul 不会将敏感的 HTTP header 转发。黑名单包含以下三种头部：

- `Cookie`
- `Set-Cookie`
- `Authorization`

那么只需要将 `Authorization` 从黑名单中移除即可：

```yaml
zuul.sensitiveHeaders: Cookie, Set-Cookie
```

另外，当一个受保护的服务要调用另一个受保护的服务时，也需要手动确保将 `Authorization` 注入对服务的调用请求中。

## 7.4 JSON Web Token 与 OAuth 2

OAuth 2 是一个基于 token 的验证框架，但是并没有给出如何定义 token 的任何标准。而 JSON Web Token (JWT, RFC-7519) 旨在为 OAuth 2 token 提供标准的结构。JWT 具有以下特点：

- 小巧 - 由 Base64 编码，可以通过 URL、HTTP header 或 request body 轻松传递
- 密码签名 - JWT token 由颁发它的服务器签名，保证 token 不会被篡改
- 自包含 - 不需要调用验证服务来确认 token 的内容 (因为已经被签过名)
- 可扩展 - 可以在 token 被密封之前，在 token 中放置一些额外的信息

> 具体都是使用方法，不展开了。

## 7.5 关于微服务安全的总结

- 为所有业务通信使用 HTTPS / 安全套接字层 (SSL)
- 使用服务网关访问服务
- 将服务划分到公共 API 和私有 API (?)
- 封锁不需要的网络端口，限制微服务的攻击面
