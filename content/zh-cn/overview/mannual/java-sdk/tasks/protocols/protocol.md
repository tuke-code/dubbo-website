---
description: "提供不同场景下的协议选型（triple、dubbo）指南，包含每个协议的基本配置方式、默认端口、适用场景等。"
linkTitle: 选择 RPC 协议
title: Dubbo 支持的 RPC 通信协议
type: docs
weight: 1
---

Dubbo 作为一款 RPC 框架内置了高效的 RPC 通信协议，帮助解决服务间的编码与通信问题，目前支持的协议包括：
 * triple，基于 HTTP/1、HTTP/2 的高性能通信协议，100% 兼容 gRPC，支持 Unary、Streming 等通信模式；支持发布 REST 风格的 HTTP 服务。
 * dubbo，基于 TCP 的高性能私有通信协议，缺点是通用性较差，更适合在 Dubbo SDK 间使用；
 * 任意协议扩展，通过扩展 protocol 可以之前任意 RPC 协议，官方生态库提供 JsonRPC、thrift 等支持。

## 协议概览

### 使用哪个协议？

**开发者该如何确定使用哪一种协议那？** 以下是我们从使用场景、性能、编程易用性、多语言互通等方面对多个主流协议的对比分析：

| <span style="display:inline-block;width:50px">协议</span> | 性能 | 网关友好 | 流式通信 | 多语言支持 | 编程API | 说明 |
| --- | --- | --- | --- | --- | --- | --- |
| triple | 高 | 高 | 支持，客户端流、服务端流、双向流 | 支持（Java、Go、Node.js、JavaScript、Rust） | Java Interface、Protobuf(IDL) | 在多语言兼容、性能、网关、Streaming、gRPC 等方面最均衡的协议实现，官方推荐。 |
| dubbo | 高 | 低 | 不支持 | 支持（Java、Go） | Java Interface | 性能最高的私有协议，但前端流量接入、多语言支持等成本较高 |
| rest | 低 | 高 | 不支持 | 支持 | Java Interface | rest 协议在前端接入、互通等方面具备最高的灵活性，但对比 rpc 存在性能、弱类型等缺点。**注意，rest 在 dubbo3 中仅是 triple 协议的一种发布形式** |

{{% alert title="注意" color="warning" %}}
自 3.3 版本开始，triple 协议支持以 rest 风格发布标准的 http 服务，因此框架中实际已不存在独立的 rest protocol 扩展实现。只是在文档说明上，我们依然选择将 triple 与 rest 作为独立的协议实现分开讲解，请大家注意。
{{% /alert %}}

### 协议默认值
以下是几个主要协议的具体开发、配置、运行态信息：
 | 协议名称 | 配置值 | 服务定义方式 | 默认端口 | 传输层协议 | 序列化协议 | 是否默认 |
 | --- | --- | --- | --- | --- | --- | --- |
 | **triple** | tri | - Java Interface <br/> - Protobuf(IDL) | 50051 | HTTP/1、HTTP/2 | Protobuf Binary、Protobuf JSON | 否 |
 | **dubbo** | dubbo | - Java Interface | 20880 | TCP | Hessian、Fastjson2、JSON、JDK、Avro、Kryo 等 | **是** |
 | **rest** | tri 或 rest | - Java Interface + SpringMVC <br/> - Java Interface + JAX-RS | 50051 | HTTP/1、HTTP/2 | JSON | 否 |

 {{% alert title="注意" color="info" %}}
 考虑到对过往版本的兼容性，当前 Dubbo 各个发行版本均默认使用 `dubbo` 通信协议。**对于新用户而言，我们强烈建议在一开始就明确配置使用 `triple` 协议**。老用户也尽快参考文档 [实现协议的平滑迁移](overview/mannual/java-sdk/reference-manual/upgrades-and-compatibility/migration-triple/)。
 {{% /alert %}}

接下来，我们一起看以下几个协议的基本使用方式。

## Triple 协议
### 基本配置
通过以下配置启用 triple 协议，默认端口为 50051，如果设置 `port: -1` 则会随机选取端口（从 50051 自增，直到找到第一个可用端口）。

```yaml
dubbo:
 protocol:
   name: tri
   port: 50051
```

### 服务定义方式
使用 triple 协议时，开发者可以使用 `Java Interface`、`Protobuf(IDL)` 两种方式定义 Dubbo RPC 服务，两种服务定义模式下的协议能力是对等的，仅影响开发者的编程体验，具体选用那种开发模式，取决于使用者的业务背景。

#### 1. Java Interface
即通过声明一个 Java 接口的方式定义服务，我们在快速开始一节中看到的示例即是这种模式，**适合于没有跨语言诉求的开发团队，具备学习成本低的优势，Dubbo2 老用户可以零成本切换协议**。

服务定义范例：

```java
public interface DemoService {
    String sayHello(String name);
}
```

可以说只设置 `protocol="tri"` 就可以了，其他与老版本 dubbo 协议开发没有任何区别。请通过【进阶学习 - 通信协议】查看 [java Interface + Triple 协议的具体使用示例](/zh-cn/overview/mannual/java-sdk/tasks/protocols/triple/interface/)。

#### 2. Protobuf(IDL)
使用 Protobuf(IDL) 的方式定义服务，**适合于当前或未来有跨语言诉求的开发团队，同一份 IDL 服务可同时用于 Java/Go/Node.js 等多语言微服务开发，劣势是 protobuf 学习成本较高**。

```Protobuf
syntax = "proto3";
option java_multiple_files = true;
package org.apache.dubbo.springboot.demo.idl;

message GreeterRequest {
  string name = 1;
}
message GreeterReply {
  string message = 1;
}

service Greeter{
  rpc greet(GreeterRequest) returns (GreeterReply);
}
```

通过 Dubbo 提供的 protoc 编译插件，将以上 IDL 服务定义预编译为相关 stub 代码，其中就包含 Dubbo 需要的  Interface 接口定义，因此在后续编码上区别并不大，只不过相比于前面的用户自定义 Java Interface 模式，这里由插件自动帮我们生成 Interface 定义。

```java
// Generated by dubbo protoc plugin
public interface Greeter extends org.apache.dubbo.rpc.model.DubboStub {
    String JAVA_SERVICE_NAME = "org.apache.dubbo.springboot.demo.idl.Greeter";
    String SERVICE_NAME = "org.apache.dubbo.springboot.demo.idl.Greeter";

    org.apache.dubbo.springboot.demo.idl.GreeterReply greet(org.apache.dubbo.springboot.demo.idl.GreeterRequest request);
    // more generated codes here...
}
```

Protobuf 模式支持的序列化方式有 Protobuf、Protobuf-json 两种模式。请通过【进阶学习 - 通信协议】查看 [Protobuf (IDL) + Triple 协议的具体使用示例](/zh-cn/overview/mannual/java-sdk/tasks/protocols/triple/idl/)。

#### 3. 我该使用哪种编程模式，如何选择？

|  | 是 | 否 |
| --- | --- | --- |
| 公司的业务是否有用 Java 之外的其他语言，跨语言互通的场景是不是普遍？ | Protobuf | Java 接口 |
| 公司里的开发人员是否熟悉 Protobuf，愿意接受 Protobuf 的额外成本吗？ | Protobuf | Java 接口 |
| 是否有标准 gRPC 互通诉求？ | Protobuf | Java 接口 |
| 是不是 Dubbo2 老用户，想零改造迁移到 triple 协议？ | Java 接口 | Protobuf |

### HTTP 接入方式
triple 协议支持标准 HTTP 工具的直接访问，因此前端组件如浏览器、网关等接入非常便捷，同时服务测试也变得更简单。

当服务启动后，可以使用 cURL 命令直接访问：
```shell
curl \
    --header "Content-Type: application/json" \
    --data '["Dubbo"]' \
    http://localhost:50052/org.apache.dubbo.springboot.demo.idl.Greeter/greet/
```

以上默认使用 `org.apache.dubbo.springboot.demo.idl.Greeter/greet` 这种 HTTP 访问路径，且仅支持 post 方法，如果你想对外发布 REST 风格服务，请参考下文 REST 协议小节。

也可参考[【使用教程 - 前端网关接入】](/zh-cn/overview/mannual/java-sdk/tasks/gateway/triple/)

## Dubbo 协议
### 基本配置
通过以下配置启用 dubbo 协议，默认端口为 20880，如果设置 `port: -1` 则会随机选取端口（从 20880 自增，直到找到第一个可用端口）。

```yaml
dubbo:
 protocol:
   name: dubbo
   port: 20880
```

### 服务定义方式
dubbo 协议支持使用 `Java Interface` 方式定义 Dubbo RPC 服务，即通过声明一个 Java 接口的方式定义服务。序列化方式可以选用 Hessian、Fastjson2、JSON、Kryo、JDK、自定义扩展等任意编码协议，默认序列化协议为 Fastjson2。

```java
public interface DemoService {
    String sayHello(String name);
}
```

{{% alert title="注意" color="info" %}}
自 3.2.0 版本开始，Dubbo 增加了序列化协议的自动协商机制，如果满足条件 `两端都为 Dubbo3 特定版本 + 存在 Fastjson2 相关依赖`，则会自动使用 fastjson2 序列化协议，否则使用 hessian2 协议，协商对用户透明无感。

由于 Dubbo2 默认序列化协议是 hessian2，对于部分有拦截rpc调用payload的场景，比如sidecar等对链路payload有拦截与解析，在升级过程中需留意兼容性问题。
{{% /alert %}}

* 关于 dubbo 协议的具体使用示例请参见【进阶学习 - 通信协议】中的 [dubbo 协议示例](/zh-cn/overview/mannual/java-sdk/tasks/protocols/dubbo/)。

### HTTP 接入方式
由于 dubbo 协议无法支持 http 流量直接接入，因此需要有一层网关实现前端 http 协议到后端 dubbo 协议的转换过程（`http -> dubbo`）。Dubbo 框架提供了 `泛化调用` 能力，可以让网关在无服务接口定义的情况下对后端服务发起调用。

<img style="max-width:800px;height:auto;" src="/imgs/v3/tasks/gateway/http-to-dubbo.png"/>

目前社区有很多开源网关产品（Higress、Shenyu、APISIX、Tengine等）支持 `http -> dubbo` 的，它们大部分都提供了可视化界面配置参数映射（泛化调用），同时还支持基于 Nacos、Zookeeper 等主流注册中心的自动地址发现，具体请查看 [【使用教程 - HTTP网关接入】](/zh-cn/overview/mannual/java-sdk/tasks/gateway/dubbo/)。

## REST 协议
在本文前面我们曾提到，自 3.3 版本开始 triple 协议支持以 rest 风格发布标准的 http 服务，因此，如果我们发布 rest 风格的服务等同于使用 triple 协议，只不过，我们要在服务定义上加入特定的注解（Spring Web、JAX-RS）。

```yaml
dubbo:
 protocol:
   name: tri
   port: 50051
```

{{% alert title="注意" color="info" %}}
对于老版本的 rest 用户配置是 `name: rest`，用户可以选择将改为 `name: tri`。即使不修改也没有问题，Dubbo 框架会自动将 rest 转换为 triple 协议实现。
{{% /alert %}}

所以说，rest 只是 triple 协议的一种特殊的发布形式，为了实现 rest 格式发布，我们需要为服务接口定义增加注解。

### 注解
目前 rest 协议仅支持 `Java 接口` 服务定义模式，相比于 dubbo 和 triple 协议，rest 场景下我们需要为 Interface 增加注解，支持 Spring MVC、JAX_RS 两种注解。

Spring MVC 服务定义范例：
```java
@RestController
@RequestMapping("/demo")
public interface DemoService {
    @GetMapping(value = "/hello")
    String sayHello();
}
```

JAX-RS 服务定义范例：
```java
@Path("/demo")
public interface DemoService {
    @GET
	@Path("/hello")
    String sayHello();
}
```

如果你记得 triple 协议原生支持 cURL 访问，即类似  `org.apache.dubbo.springboot.demo.idl.Greeter/greet` 的访问模式。通过增加以上注解后，即可为 triple 服务额外增加 REST 风格访问支持。

关于 rest 协议的具体使用示例请参见【使用教程 - 通信协议】中的 [rest 协议示例](../rest/)

## 多协议发布
### 多端口多协议
多协议发布是指为同一个服务同时提供多种协议访问方式，多协议可以是任意两个或多个协议的组合，比如下面的配置将同时发布了 dubbo、triple 协议：

```yaml
dubbo:
 protocols:
   tri:
     name: tri
     port: 50051
   dubbo:
     name: dubbo
	 port: 20880
```

基于以上配置，如果应用中有服务 DemoService，则既可以通过 dubbo 协议访问 DemoService，也可以通过 triple 协议访问 DemoService，其工作原理图如下：

<img alt="多协议" style="max-width:800px;height:auto;" src="/imgs/v3/tasks/protocol/multiple-protocols.png"/>

1. 提供者实例同时监听两个端口 20880 和 50051
2. 同一个实例，会在注册中心注册两条地址 url
3. 不同的消费端可以选择以不同协议调用同一个提供者发布的服务

对于消费端而言，如果用户没有明确配置，默认情况下框架会自动选择 `dubbo` 协议调用。Dubbo 框架支持配置通过哪个协议访问服务，如 `@DubboReference(protocol="tri")`，或者在 application.yml 配置文件中指定全局默认值：

```yaml
dubbo:
 consumer:
   protocol: tri
```

### 单端口多协议

除了以上发布多个端口、注册多条 url 到注册中心的方式。对于 dubbo、triple 这两个内置协议，框架提供了在单个端口上同时发布 dubbo 和 triple 协议的能力。这对于老用户来说是一个非常重要的能力，因为它可以做到不增加任何负担的情况下，让使用 dubbo 协议的用户可以额外发布 triple 协议，这样当所有的应用都实现多协议发布之后，我们就可以设置消费端去通过 triple 协议发起调用了。

<img alt="单端口多协议" style="max-width:800px;height:auto;" src="/imgs/v3/tasks/protocol/multiple-protocols-on-same-port.png"/>

单端口多协议的基本配置如下：

 ```yaml
 dubbo:
  protocol:
    name: dubbo
    ext-protocol: tri
 ```

 如果想了解更多多协议使用方法，[参考手册](overview/mannual/java-sdk/reference-manual/protocol/multi-protocols/) 和 [协议迁移指南](overview/mannual/java-sdk/reference-manual/upgrades-and-compatibility/migration-triple/) 中有关于多协议的更多示例说明。
