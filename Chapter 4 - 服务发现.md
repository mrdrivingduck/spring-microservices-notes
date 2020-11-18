# Chapter 4 - 服务发现

Created by : Mr Dk.

2020 / 08 / 14 13:35

@Nanjing, Jiangsu, China

---

服务发现对基于 **云** 和 **微服务** 的应用程序至关重要：

1. 能够将服务的物理位置抽象，从而快速对服务实例的数量进行水平伸缩 (添加更多服务器)，而不是垂直伸缩 (更好的服务器)
2. 提高应用程序的弹性，能够绕过不可用的服务实例

## 4.1 我的服务在哪里

当应用程序需要调用分布在多个服务器上的资源时，要做的第一件事是定位这些资源的物理位置。在非云的架构设计中，通常是通过 **DNS** 和 **网络负载均衡器** 来解决。客户端以一个通用的 DNS 名称，和需要调用服务的路径发起请求。负载均衡器会根据消费者尝试访问的路径，在路由表中定位物理地址条目，并选择其中的一个服务器，将请求转发到服务器上。服务的各个实例通常是 **静态** 和 **持久** 的。为了保证高可用，一个辅助的后备负载均衡器会处于待命状态。

对基于云的微服务应用程序来说，上述模型并不适用：

* 单点故障
* 跨多个服务器水平伸缩的负载均衡设施能力有限
* 静态管理 (无法快速注册、注销)
* 复杂 - 开发人员必须手动定义和部署服务的映射规则

## 4.2 云中的服务发现

基于云的微服务环境解决方案：使用 **服务发现** 机制。具有以下特点：

* 高可用 - 服务发现集群可以跨结点共享服务查找
* 点对点 - 服务发现集群中的每个结点共享服务实例的状态
* 负载均衡 - 在所有服务实例之间动态负载均衡
* 有弹性 - 客户端在本地缓存服务信息，使得服务发现能够逐步降级
* 容错 - 服务发现能够检测服务实例的健康性，并在没有人为干预的情况下对故障采取行动

### 4.2.1 服务发现架构

服务发现的架构包括以下四个概念：

* 服务注册
* 服务地址查找
* 信息共享
* 健康检测

当服务实例启动时，将向一个或多个 **服务发现实例** 来注册自身可以被访问的物理位置、路径和端口。同一个服务的服务实例具有唯一的 IP 地址和端口，但都是以相同的 **服务 ID** 进行注册。服务 ID 是标识一组相同服务实例的 key。服务通常只会在一个服务发现实例中注册，然后通过多点广播协议传遍服务发现集群的所有其它结点。

每个服务实例会向服务发现实例推送自身的健康状态，或是服务发现实例从服务实例拉取状态。未能返回良好健康信息的实例将从可用服务实例池中删除。

客户端可以只依赖于服务发现引擎来解析服务位置 - 每当调用某个微服务时，服务发现引擎就会被调用。这种方法很脆弱，因为客户端完全依赖于服务发现引擎。一种较为健壮的方法是 **客户端负载均衡**：

1. 客户端联系服务发现实例，请求它获取所有的服务实例，在客户端本地缓存服务实例数据
2. 客户端直接从缓存中查找服务的位置信息，并使用简单的负载均衡算法
3. 客户端定期与服务发现实例联系，更新本地的缓存；如果服务调用失败，那么本地缓存立刻失效，需要重新获取缓存

### 4.2.2 使用 Spring 和 Netflix Eureka 进行服务发现实战

1. 服务启动，实例通过 Eureka 服务进行注册，告诉 Eureka 本实例的物理位置、端口号、服务 ID
2. 客户端使用 Netflix Ribbon 去联系 Eureka 服务检索服务位置信息，然后在本地进行缓存
3. Netflix Ribbon 定期对 Eureka 服务进行 ping，刷新本地缓存

## 4.3 构建 Spring Eureka 服务

在 Maven 中添加 Eureka 依赖的 JAR 包，然后在 `src/main/resouces/application.yml` 中添加 Eureka 服务所需要的配置：

* Eureka 服务运行的端口
* 该服务自身不注册
* 该服务自身不缓存本地注册表
* 在服务开始接收请求前等待的初始时间

> 每次服务注册默认需要 30s 才能显示在 Eureka 实例给出的服务列表中 - 这是因为 Eureka 需要接收 3 次连续的 ping，每次间隔 10s，才能开始使用这个服务。

启动 Eureka 服务也需要一个引导类。在引导类上，需要添加 `@SpringBootApplication` 和 `@EnableEurekaServer` 两个注解。

## 4.4 通过 Spring Eureka 注册服务

在其它实例的 Maven 依赖中，也需要添加 Eureka 依赖，使其能够向 Eureka 服务实例注册。然后需要去 `src/main/java/resource/` 目录下的 `application.yml` 与 `bootstrap.yml` 中配置访问 Eureka 服务实例的方式。

每个通过 Eureka 注册的服务都包含两个 ID：

* 应用程序 ID - 由 `spring.application.name` 指定，用于表示一组提供相同服务的实例
* 实例 ID - 一个随机数，用于代表单个服务实例

其余的配置包括：

* `eureka.instance.preferIpAddress` - 用当前实例的 IP 地址注册，而不是主机名 (对 docker 可能会有问题)
* `eureka.client.registerWithEureka` - 触发器，告诉当前实例通过 Eureka 注册它本身
* `eureka.client.fetchRegistry` - 告知 Eureka 客户端，在本地缓存注册表
* `eureka.client.serviceUrl.defaultZone` - 包含客户端可访问的所有 Eureka 实例的列表

这样，服务实例作为 Eureka 客户端，就知道去哪里访问，以及如何访问 Eureka 服务了。

## 4.5 使用服务发现来查找服务

在服务被注册到 Eureka 上后，其它服务可以向 Eureka 查找服务，并调用。有三种不同层次的服务调用方法。

### 4.5.1 使用 Spring DiscoveryClient 查找服务实例

这是最低层次的访问。需要在引导类中加入 `@EnableDiscoveryClient` 注解，是应用程序能够使用 DiscoveryClient 和 Ribbon 库。一个 DiscoveryClient 类的实例会被注入用户类中，这样用户代码可以获取到服务的所有实例列表 (包含了主机名、端口、URI)。用户代码可以自行选择要调用的实例，然后使用标准的 Spring REST 模板类去调用服务。

由于这里是由用户代码自行决定调用哪一个实例，因此没有使用到 Ribbon 的客户端负载均衡功能。由于访问层次较低，用户代码做了很多的工作。

### 4.5.2 使用带有 Ribbon 功能的 Spring RestTemplate 调用服务

需要使用 Spring Cloud 的注解 `@LoadBalanced` 来定义 `RestTemplate` bean 的构造函数。将 `RestTemplate` 类的实例装配到用户类中，然后直接用这个实例发起服务调用。在使用这个类时，实际的服务位置和端口与开发人员完全抽象隔离。

另外，通过使用 `RestTemplate`，Ribbon 将在所有服务实例之间轮询负载均衡。

### 4.5.3 使用 Netflix Feign 客户端调用服务

由开发人员定义一个 Java 接口，然后使用 Spring Cloud 注解来标注接口，映射将要调用的服务。Spring Cloud 框架将动态生成代理类，调用目标 REST 服务。除了编写接口定义，开发人员不需要编写任何调用服务的代码。

在引导类上，需要添加 `@EnableFeignClients` 注解。然后定义用于调用服务的接口。接口需要用 `@FeignClient` 注解标识。在接口中的具体函数中，用 `@RequestMapping` 注解来定义服务端点的路径和动作，使用 `@PathVariable` 来定义传入函数的参数：

```java
@FeignClient("organizationservice")
public interface OrganizationFeignClient {
    
    @RequestMapping(
    	Method=RequestMethod.GET,
        value="/v1/organizations/{organizationId}",
        consumes="application/json"
    )
    Organization getOrganization(
        @PathVariable("organizationId") String organizationId);
}
```

开发人员只需要自动装配并使用这个类即可。通过 Feign 客户端，只要被调用的服务返回了 4xx-5xx 的状态码，都会被映射到 `FeignException`。

---

