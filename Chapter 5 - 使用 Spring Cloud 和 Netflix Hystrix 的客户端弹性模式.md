# Chapter 5 - 使用 Spring Cloud 和 Netflix Hystrix 的客户端弹性模式

Created by : Mr Dk.

2020 / 08 / 15 13:44

@Nanjing, Jiangsu, China

---

当一个服务崩溃时，很容易检测到服务已经不存在了，因此应用程序可以绕过它；而当服务运行缓慢时，检测到这个服务性能不佳并绕过它是很困难的：

1. 服务降级可能只发生在很小的爆发中，直到应用程序容器耗尽线程池并彻底崩溃
2. 对远程服务的调用是同步的，没有超时的概念来阻止服务调用被永久挂起
3. 应用程序通常被设计为处理远程资源彻底故障，而不是远程资源的降级

性能不佳的远程服务的潜在问题是，它们不仅 **难以检测**，还会触发 **连锁效应**。一个性能不佳的服务可以迅速拖垮多个应用程序。

## 5.1 什么是客户端弹性模式

**客户端弹性模式** 的重点是，在远程服务表现不佳时，保护远程资源的客户端 (另一个微服务) 免于崩溃。模式的目标是让调用远程资源的客户端 **快速失败**，从而不再消耗宝贵的资源，如数据库连接、线程池等。

### 5.1.1 客户端负载均衡 (Client Load Balance) 模式

客户端负载均衡器位于服务客户端和服务消费者之间，所以负载均衡器可以检测服务实例是否抛出错误或性能不佳，从而从可用服务位置池中移除该服务实例。

### 5.1.2 断路器 (Circuit Breaker) 模式

与电气系统中的断路器类似 - 检测是否有过多电流流过电线，如果有，则断开与电气系统其余部分的连接。软件断路器也是类似 - 监视一个调用所花的时间，如果调用时间过长，则介入并中断调用。另外，断路器还会监视全局的调用，如果对某个资源调用失败的次数足够多，那么断路器就会采取快速失败的策略。

### 5.1.3 后备 (Fallback) 模式

当远程服务调用失败时，服务消费者将执行一些后备代码，尝试通过其它方式执行操作，而不是产生一个异常。比如，在查询实时数据时，如果查询失败，那么转而查询稍微过时一些的数据 - 这会比直接的失败带给用户的体验稍好些 (当然这也要看场景和具体的后备策略)。

### 5.1.4 舱壁 (Bulkhead) 模式

这个词来源于造船。船中会划分出多个互相完全隔离的舱壁。当船体被部分击穿时，未被击穿的舱壁将不会进水，从而保证船体不容易沉没。同样的概念应用于与多个远程资源进行交互的服务。通过舱壁模式，可以把对多个远程资源的调用线程隔离到不同的线程池中，防止访问某一个资源时速度慢而占用线程池，使得其它的资源完全无法被访问。

## 5.2 为什么客户端弹性很重要

在服务表现不佳时，弹性可以免于链式反应导致服务直接崩溃，并使得表现不佳的实例有喘息的空间恢复服务。

## 5.5 使用 Hystrix 实现断路器

在 Maven 中加入 Spring Hystrix 的依赖项，并使用 `@EnableCircuitBreaker` 注解来标注引导类。Hystrix 支持包装所有对数据库的调用，也可以包装微服务之间的调用，用法一致。

Hystrix 和 Spring Cloud 使用 `@HystrixCommand` 注解来标记由断路器管理的 **类函数**。当函数被这个注解标注时，将动态生成代理，由代理通过 **专门用于处理远程调用的线程池** 管理对这个函数的调用。由断路器管理这个函数的调用后，每当这个函数的调用时间超过阈值 (1000ms) 时，断路器会中断这个函数的调用，并抛出 `HystrixRuntimeException` 异常。

可以通过将附加参数传递给 `@HystrixCommand` 注解的方式，定制 Hystrix 的远程调用超时阈值：

```java
@HystrixCommand(
    commandProperties = {
        @HystrixProperty(
            name="execution.isolation.thread.timeoutInMilliseconds", value="12000"
        )
    }
)
public List<License> getLicensesByOrg(String organizationId) {
    return licenseRepository.findByOrganizationId(organizationId);
}
```

## 5.6 后备处理

断路器本身就是远程资源与消费者的 *中间人*，所以断路器能够有机会拦截服务故障，并选择替代方案作为后备处理。使用起来很简单：

1. 在 `@HystrixCommand` 注解中添加一个 `fallbackMethod` 属性，指定一个函数名 - 该函数将在服务耗时太长被断路器切断都调用
2. 实现这个后备处理的函数 - 显然，这个函数需要与原函数有着相同的 **函数签名** (参数、返回值)

```java
@HystrixCommand(fallbackMethod = "buildFallbackLicenseList")
public List<License> getLicensesByOrg(String organizationId) {
    return licenseRepository.findByOrganizationId(organizationId);
}

private List<License> buildFallbackLicenseList(String organizationId) {
    return new License();
}
```

两个注意点：

1. 如果只是想在断路器超时后记录日志，那么只需要用 `try...catch...` 块捕获 `HystrixRuntimeException` 异常即可，没有必要在后备函数中实现日志记录逻辑
2. 如果后备策略是调用另一个微服务，那么后备函数也需要被包装 `@HystrixCommand` 注解

## 5.7 实现舱壁模式

当应用需要调用多个微服务来完成特定工作时，这些调用默认由同一批线程执行 - 这些线程是 Java 容器为处理请求而预留的。当一个服务出现性能问题时 (比如请求量高，或完成时间长)，线程池中的所有线程会逐渐被这个服务占用，阻塞新请求，最终导致整个容器崩溃。如果使用舱壁模式，可以将各种远程资源的调用隔离在各自的线程池中，以便控制单个表现不佳的服务，不会使容器崩溃。

Hystrix 也是使用线程池来委派所有对远程服务的请求。默认情况下，所有请求共享同一个线程池，请求可以是 REST 服务调用、数据库调用等。Hystrix 另外提供了易于使用的机制，能够将不同类型的请求隔离到不同的线程池中。

要使用隔离的线程池，需要给出三个参数：

1. 设置一个单独的线程池 - 实际上就是用一个唯一的名称命名一个线程池
2. 设置线程池中的线程数
3. 为线程池创建的等待队列的长度

```java
@HystrixCommand(fallbackMethod = "buildFallbackLicenseList",
               threadPoolKey = "licenseByOrgThreadPool",
               threadPoolProperties = {
                   @HystrixProperty(name = "coreSize", value = "30"),
                   @HystrixProperty(name = "maxQueueSize", value = "10")
               }
)
public List<License> getLicensesByOrg(String organizationId) {
    return licenseRepository.findByOrganizationId(organizationId);
}

private List<License> buildFallbackLicenseList(String organizationId) {
    return new License();
}
```

对于 `maxQueue` 属性：

* 如果被设置为 `-1`，那么将使用 JDK 中的 `SynchronousQueue` 来保存所有请求
* 如果被设置为大于 `1`，那么将使用 JDK 中的 `LinkedBlockingQueue` 来保存所有请求

> 这两个类还没有读 JDK 源代码，回头看一看有何区别。书上写得很含糊。

另外，这个属性只用于线程池初始化。当属性的值大于 `0` 时，Netflix 支持通过 `queueSizeRejectionThreshold` 属性动态更改队列的大小。这个值可以被微调，微调的关键指标就是，当目标远程资源健康时，服务调用依然超时。

## 5.8 基础进阶 - 微调 Hystrix

Hystrix 不仅能中止超时的调用，还会监控调用失败的次数 - 如果失败足够多次，那么 Hystrix 会直接阻止未来的调用，也就是所谓的 **快速失败**。这么做的原因：

1. 快速失败可以防止应用程序等待调用超时，降低了资源耗尽的风险
2. 快速失败给了性能下降的系统一些时间去修复，而不会快速崩溃

当一个 Hystrix 命令遇到服务错误时，将开始一个 (10s) 的计时器，用于检查服务调用失败的 **频率**。在这个时间窗口内，如果服务调用次数低于断路器启用的最小调用次数，即使这些调用全失败，断路器也不会跳闸。

当时间窗口内服务调用次数超过最小调用次数后，Hystrix 开始计算 **整体故障百分比**。如果故障百分比超过阈值，那么 Hystrix 触发断路器，将来 **几乎** 所有的调用都会直接快速失败。这里用了 **几乎** 是因为 Hystrix 会启动一个新的时间窗口 (5s)，让一个调用通过，测试服务有没有恢复。如果这个调用成功，那么断路器被重置，允许所有调用通过；否则保持断路器断开，并在一个新的时间窗口内尝试试探。

以上，对应的属性如下：

* `circuitBreaker.requestVolumeTHreshold` - 跳闸前时间窗口内必须发生的连续调用数量
* `circuitBreaker.errorThresholdPercentage` - 超过上述阈值后，跳闸前必须达到的调用失败百分比
* `circuitBreaker.sleepWindowInMilliseconds` - 允许一个调用访问服务以探测服务是否恢复的时间窗口
* `metrics.rollingStats.timeInMilliseconds` - Hystrix 用于监控服务调用问题的窗口大小
* `metrics.rollingStats.numBuckets` - 在滚动窗口中收集信息的次数

```java
@HystrixCommand(
    fallbackMethod = "buildFallbackLicenseList",
    threadPoolKey = "licenseByOrgThreadPool",
    threadPoolProperties = {
        @HystrixProperty(name = "coreSize", value = "30"),
        @HystrixProperty(name = "maxQueueSize", value = "10")
    },
    commandPoolProperties = {
        @HystrixProperty(name = "circuitBreaker.requestVolumeTHreshold", value = "10"),
        @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "75"),
        @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "7000"),
        @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "15000"),
        @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "5")
    }
)
public List<License> getLicensesByOrg(String organizationId) {
    return licenseRepository.findByOrganizationId(organizationId);
}
```

从上面的代码可以看到，Hystrix 是高度可配置的。开发人员可以使用三个配置级别：

* 整个应用程序级别的默认值
* 类级别的默认值
* 类中定义的线程池级别

每个配置都有默认值，开发人员可以分别在类级别和线程池级别显式覆盖默认配置。在生产环境中，最有可能需要调整的配置都应当被外化到 Spring Cloud Config 中。

---

