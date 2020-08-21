# Chapter 8 - 使用 Spring Cloud Stream 的事件驱动架构

Created by : Mr Dk.

2020 / 08 / 21 20:59

@Nanjing, Jiangsu, China

---

## 8.1 为什么使用消息传递、EDA 和微服务

事件驱动架构 (Event Driven Architecture, EDA) / 消息驱动架构 (Message Driven Architecture, MDA)。

假设有一个场景，服务 A 中需要用到服务 B 的数据，因此使用缓存 (比如 *Redis*) 维护了一份服务 B 的数据。如果服务 B 被其它服务调用后，数据发生了变化，那么就需要使服务 A 缓存中的旧数据变为无效。有很多种方法可以实现这样的功能。

### 8.1.1 使用同步请求 - 响应方式来传达状态变化

服务 A 在接收到客户端请求后，才是查找 Redis。如果 Redis 中没有缓存，则调用服务 B 的 REST end point 查询数据，并将数据放到 Redis 中后，将客户端要查询的数据返回。现在，如果有人直接使用了服务 B 的 REST end point 对服务 B 的数据进行了修改，那么服务 B 需要调用服务 A 暴露出的 end point，通知服务 A 缓存数据失效。

这种设计具有几个问题。

1. 服务之间紧耦合 - 服务 A 为了服务 B 专门暴露了 end point；服务 B 直接与 Redis 通信在微服务环境中是个禁忌
2. 服务之间的脆弱性 - 两个服务之间出现了依赖关系，如果服务 A 运行缓慢，那么服务 B 可能也会被拖垮
3. 如果其它服务对服务 B 的数据变化感兴趣，那么需要在服务 B 的代码中加入更多的逻辑，形成服务间的网状依赖

### 8.1.2 使用消息传递在服务之间传达状态更改

服务 B 的数据发生变化时，服务 B 发送一条消息到队列中。服务 A 监视消息队列，当检测到想要的消息时，将 Redis 中的缓存清除。这种方法的好处有以下四点：

1. 松耦合 - 涉及传达状态更改时，两个服务都不知道彼此，只知道消息队列
2. 耐久性 - 队列能够保证即使服务 A 已经关闭，服务 B 依然可以发送消息；消息保存在队列中直到服务 A 重新可用
3. 可伸缩性 - 消息发送者不必等待消息消费者的响应，可以直接继续工作；如果消息消费者没有足够能力处理，那么启动更多的消费者实例即可
4. 灵活性 - 开发人员可以轻松添加新的消息消费者，而不影响原始发送服务

当然，消息传递架构也有缺点：

1. 消息处理语义 - 不仅仅需要直到如何发布和消费消息，还需要知道消息没有按序处理时会发生什么问题
2. 消息可见性 - 消息的发送与消费是异步的，可能不会立刻被接收或处理
3. 消息编排 - 难以按照应用程序执行顺序进行业务逻辑推理，因为不再是请求-响应模型的线性处理方式

综上，消息能够使开发人员将服务 **挂钩** 在一起，而不是 **硬编码** 在一起。

## 8.2 Spring Cloud Stream 简介

Spring Cloud Stream 是一个由注解驱动的框架，允许开发人员在 Spring 应用程序中轻松构建发布者和消费者。另外，Spring Cloud Stream 还允许开发人员抽象出正在使用的消息传递平台的具体实现 - *Apache Kafka* 或 *RabbitMQ*。平台的具体实现代码被排除在应用程序代码以外，消息发布和消费是通过平台无关的 Spring 接口实现的。

在架构上，有四个组件涉及消息的发布与接收：

* 发射器 (source) - 接收 Java POJO，将其序列化并发送到通道
* 通道 (channel) - 对队列的抽象，通道名称始终与目标队列名称相关联，可以通过配置文件修改要读取或写入的队列
* 绑定器 (binder) - 与特定消息平台对话的 Spring 代码，允许开发人员使用通用接口处理消息
* 接收器 (sink) - 监听传入消息的通道，将消息反序列化为 Java POJO

## 8.3 编写简单的消息生产者和消费者

### 8.3.1 在组织服务中编写消息生产者

使服务 B 能够在数据发生变化时，向 *Kafka* 的特定主题发布一条信息。

首先需要添加 Maven 依赖项，然后在服务的引导类上添加 `@EnableBinding(Source.class)` 注解。`Source` 类定义了通道与消息代理进行通信的方式，开发人员可以自行实现。向消息代理发布消息的逻辑实现如下：

```java
@Component
public class SimpleSourceBean {
    private Source source;
    
    private static final Logger logger = LoggerFactory.getLogger(SimpleSourceBean.class);
    
    // 该接口将被自动注入
    @Autowired
    public SimpleSourceBean(Source source) {
        this.source = source;
    }
    
    public void publishOrgChange(String action, String orgId) {
        OrganizationChangeModel change = new OrganizationChangeModel(
            OrganizationChangeModel.class.getTYpeName(),
            action,
            orgId,
            UserContext.getCorrelationId());
        // 发送
        source.output().send(MessageBuilder.withPayload(change).build());
    }
}
```

`Source` 时 Spring Cloud 定义的接口，其中公开了 `output()` 函数，用于将消息发送。

而如何将当前服务绑定到特定的消息队列呢？通过配置来完成。配置能够将 `Source` 映射到 Kafka 消息代理中的特定主题上：

```yaml
spring:
  application:
    name: organizationservice
    stream:
      binding:
        output:
          destination: orgChangeTopic
          content-type: application/json
      kafka:
        binder:
          zkNodes: localhost
          brokers: localhost
```

上述配置体现了将 `Source` 中的 `output` 通道映射到 `orgChangeTopic` 消息队列上，消息类型为 JSON。并告诉 Spring 使用了 *Kafka* 消息队列，以及 *Kafka* 和 *ZooKeeper* 的网络位置。

在需要发布消息的类中，Spring 将自动装配用于发送消息的上述类。调用该类中实现的函数即可：

```java
@Service
public class OrganizationService {
    
    @Autowired
    SimpleSourceBean simpleSourceBean;
    
    public void saveOrg(Organization org) {
        // ...
        SimpleSourceBean.publishOrgChange("SAVE", org.getId());
        // ...
    }
}
```

### 8.3.2 在许可证服务中编写消息消费者

消息消费者监听消息队列中的消息来对组织服务中的更改做出反应。首先还是需要添加 Maven 依赖项，然后标记引导类。标记的方式与生产者有些不同：

```java
@EnableBinding(Sink.class)
public class Application {
    
    @StreamListener(Sink.INPUT)
    public void loggerSink(OrganizationChangeModel orgChange) {
        // ...
    }
}
```

这里将会默认使用 Spring 的 `Sink` 接口，`Sink` 接口中公开了一个默认的 `input` 通道。每次从 `input` 通道中接收到消息，都会调用上述函数，并自动将消息反序列化为 `OrganizationChangeModel` 的 Java POJO。

同样，消息队列到 `input` 通道的映射是由配置完成的：

```yaml
spring:
  application:
    name: licensingservice
  cloud:
    stream:
      bindings:
        input:
          destination: orgChangeTopic
          content-type: application/json
          group: licensingGroup
        binder:
          zkNodes: localhost
          brokers: localhost
```

将 `input` 通道与远程消息队列关联。另外，注意这里的 `group` 属性，定义了将要消费消息的组名称。每个服务可能会有多个实例正在监听消息队列，这些实例同属一个组。Spring Cloud Stream 和底层消息代理将保证，只有消息的 **一个副本** 会被属于该组的服务实例所使用。换句话说，消费者组可以强制跨多个服务消费的信息 **只被消费一次**。

## 8.4 Spring Cloud Stream 用例：分布式缓存

### 8.4.2 定义自定义通道

开发人员可以为应用程序公开任意数量和名称的输入和输出通道。以消息消费者为例，需要定义一个接口，并用 `@Input` 函数级注解标记一个返回 `SubscribableChannel` 类对象的函数，通过这个注解指明通道的名称；消息生产者也是类似，只不过注解变为 `@OutputChannel`，返回的对象类为 `MessageChannel`：

```java
public interface CustomChannels {
    @Input("inboundOrgChanges")
    SubscribableChannel orgs();
    
    @OutputChannel("outboundOrg")
    MessageChannel outboundOrg();
}
```

还需要在配置中将通道名与消息队列名进行映射：

```yaml
spring:
  stream:
    bindings:
      inboundOrgChanges:
        destination: orgChangeTopic
        content-type: application/json
        group: licensingGroup
```

而使用自定义通道时，在应用程序中传入的就是上面定义的接口类了：

```java
@EnableBinding(CustomChannels.class) // 接口类
public class OrganizationChangeHandler {
    
    @StreamListener("inboundOrgChanges") // 通道名
    public void loggerSink(OrganizationChangeModel orgChange) {
        // ...
    }
}
```

---

