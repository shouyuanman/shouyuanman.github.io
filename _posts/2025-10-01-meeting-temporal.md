---
title: 玩转Temporal
date: 2025-10-01 20:02:00 +0800
categories: [后端, Temporal]
tags: [后端, 分布式微服务, Temporal, 工作流引擎]
music-id: 31514407
---

## **印象——Temporal**

![Desktop View](/assets/img/20250930/temporal_icon.png){: width="700" height="400" }

`Temporal`是一个开源的分布式工作流编排框架，用于构建可靠、可扩展的分布式应用。它源自`Uber`的`Cadence`项目，由原核心团队于`2019`年创建，已成为云原生工作流编排的事实标准。

作为一个构建、运行和管理分布式工作流的平台，它提供了强大的工作流引擎和`SDK`，支持多种编程语言。

`Temporal`的核心概念包括工作流（`Workflow`）和活动（`Activity`），工作流定义了业务流程的逻辑和执行顺序，活动则表示具体的操作或任务。`Temporal`保证工作流的执行是可靠的，即使在面对故障和网络中断时，也能恢复并继续执行。

如果你在工作中遇到过以下这些场景，都可以了解下`Temporal`引擎。
1. 跨服务、跨时间周期的复杂业务流程
2. 业务工作流建模（`BPM`）
3. `DevOps`工作流
4. `Saga`分布式事务
5. `BigData`数据处理和分析`Pipeline`
6. `Serverless`[函数编排](https://github.com/serverlessworkflow/specification)

这些场景看上去互相没有太大关联，但有一个共同点：需要编排（`Orchestration`）。`Temporal`解决的关键痛点，就是分布式系统中的编排问题。

>
**What?**<br/>
Temporal is the open source runtime for managing distributed applicatioin state at scale.<br/>
<br/>
**Why?**<br/>
The most valuable, mission-critical workloads in any software company are long-running and tie together multiple services.<br/>
<br/>
**Requirement**
```
1. Workflows as Code
Because this work is complex:
You want to easily model dynamic asynchronous logic...
... and reuse, test, version and migrate it.
2. Orchestration 
Because this work relies on unreliable systems:
You want to standardize timeouts and retries.
You want offer "reliability on rails" to every team.
3. Event Sourcing
Because this work is so important:
You must never drop any work.
You must log all progress.
You must be able to scale it up without replatforming.
```
<br/>
**Outcomes**
```
1. More reliable
Fail to execute/drop data less often: from 1 production incident a week to ~0
When parts of application do fail, always recover to consistent state.
2. More productive
40-60% fewer lines of code and infra when writing features
DistSys/Orchestration concerns outsourced to Temporal
3. Easier to operate
Temporal consolidates errors, lets you make fixes without downtime
Event sourced system is highly observable by default
```
<br/>
**摘自 Temporal GitHub README**<br/>
<br/>
Temporal is a durable execution platform that enables developers to build scalable applications without sacrificing productivity or reliability. The Temporal server executes units of application logic called Workflows in a resilient manner that automatically handles intermittent failures, and retries failed operations.<br/>
<br/>
`Temporal`使开发人员能够在不牺牲生产力或可靠性的情况下构建可扩展的应用程序。`Temporal`服务器以弹性的方式执行应用程序逻辑单元，工作流，自动处理间歇故障，并重试失败的操作。
{: .prompt-tip }

## **理解——编排本质**
要理解编排，可以借助和`Orchestration`对应的另一个概念：`Choreography`。

![Desktop View](/assets/img/20250930/orchestration_vs_choreography.png){: width="700" height="400" }
_choreography .vs. orchestration（图片来自网络）_

看图理解下`Choreography`，

举个例子，在开发微服务时，经常借助消息队列`MQ`做事件驱动的业务逻辑，实现最终一致的、跨多个服务的数据流，这属于`Choreography`。而一旦引入了`MQ`，可能会遇到下面一系列问题，
- 消息时序问题
- 重试幂等问题
- 事件和消息链路追踪问题
- 业务逻辑过于分散的问题
- 数据已经不一致的校正对账问题
- ...

在复杂微服务系统中，`MQ`是一个很有用的组件，但`MQ`不是银弹，这些问题经历过的人会懂。如果过度依赖类似`MQ`的方案事件驱动，但又没有足够强大的消息治理方案，整个分布式系统将嘈杂不堪，难以维护。

如果转换思路，找一个“调度主体”，让所有消息的流转，都由这个"指挥家"来控制怎么样呢？对，这就是`Orchestration`的含义。
- `Choreography`是无界上下文，去中心化，每个组件只关注和发布自己的事件，完全异步，注重的是解耦；
- `Orchestration`是有界上下文，存在全局编排者，从全局建模成状态机，注重的是内聚。

`Temporal`的所有应用场景，都是有全局上下文、高内聚的编排场景。比如`BPM`有明确的流程图，`DevOps`和`BigData Pipeline`有明确的`DAG`，长活事务有明确的执行和补偿流程。

`Temporal`让我们像写正常的代码一样，可以写一段工作流代码，但并不一定是在本机执行，哪一行在什么时间`yield`，由服务端信令统一控制，很多分布式系统韧性问题也被封装掉了，比如分布式锁、宕机导致的重试失败、过期重试导致的数据错误，并发消息的处理时间差问题等等。

## **枚举——Temporal关键概念**

1. **`Workflow`（工作流）**

    `Workflow`是在编排层的关键概念，每种类型是注册到服务端的一个`WorkflowType`，每个`WorkflowType`可以创建任意多的运行实例，即`WorkflowExecution`，每个`Execution`有唯一的`WorkflowID`，如果是`Cron/Continue-as-New`, 每次执行还会有唯一的`RunID`。`Workflow`可以有环，可以嵌套子工作流（`ChildWorkflow`）。
    - 是一个长时间运行的过程，可能持续数分钟、数小时甚至数月；
    - 必须是确定性的（`deterministic`） —— 相同输入、相同执行顺序必须产生相同结果；
    - 工作流不能执行`IO`操作，所有的外部操作必须交给`Activity`；
    - 工作流是可以被暂停、恢复、重放的；
    - 工作流中的所有状态、步骤都会被记录在执行历史中（用于溯源和重放）。

2. **`Activity`（活动）**

    `Workflow`所编排的对象主要就是`Activity`，编排`Activity`就像正常写代码一样，可以用`if/for`甚至`while(true)`等各种逻辑结构来调用`Activity`方法，只要具备确定性即可。<br/>`Activity`是工作流中调用的实际执行任务，比如发送邮件、调用`HTTP`接口、写数据库等。
    - 可以包含非确定性代码，如网络请求、数据库操作；
    - 运行失败会自动重试；
    - 可以配置超时、重试策略、幂等性保障；
    - `Activity`的执行是由`Temporal Worker`执行的，可以分布式并行部署。
    - 长时间运行的`Activity`可以通过心跳机制报告其进度，这些信息可以在`Temporal UI`中查看。<br/>一个工作流通常会包含多个`Activity`调用。

3. **`Signal`（信号）**

    对于正在运行的`WorkflowExecution`，可以发送携带参数的信号，`Workflow`中可以等待或根据条件处理信号，动态控制工作流的执行逻辑。<br/>`Signal`是一种机制，允许外部事件或系统向正在运行的工作流发送数据或通知。
    - 工作流可以`Workflow.await()`等待某个`Signal`；
    - 实现了事件驱动型工作流；
    - 类似于“向正在执行的流程推送消息”；
    - 比如可以在审批过程中，让前端传递参数作为`Singnal`，而工作流会在收到`Signal`后才会继续向后执行

4. **`Worker`（工作节点）**

    `Worker`是真正运行工作流代码或`Activity`代码的程序进程。
    - 有两类`Worker`：`Workflow Worker`和`Activity Worker`；
    - 通常你用同一个程序注册`workflow + activity`； 
    - `Worker`会从`Temporal Server`拉取任务并执行，然后汇报执行结果；
        - `Work`会启动多个线程池，用于
            - `Workflow`执行
            - `Activity`执行
            - `Poller`（拉取任务）
    - 可以弹性扩展多个实例，提高并发执行能力。

5. **`Task Queue`（任务队列）**

    工作流或活动的执行请求是放入某个`Task Queue`中，由`Worker`拉取任务执行。
    - 每个`Worker`监听一个或多个任务队列；
    - 支持将不同的`Workflow/Activity`绑定到不同的`Task Queue`；
    - 支持路由、负载均衡。

6. **`Temporal Server`（核心引擎）**

    `Temporal`的后端服务，负责协调整个系统。
    - 负责调度工作流、持久化历史、发送任务给`Worker`；
    - 是高可用的服务集群，包含`Frontend`、`History`、`Matching`、`Worker`等子模块；
    - 数据存储在数据库中（`MySQL/Postgres/Cassandra`等）；
    - 你只需写业务逻辑，`Server`负责调度、重放、持久化等系统复杂性。

7. **`Query`（查询）**

    允许从外部查询工作流的当前状态，不会影响其执行状态。
    - 是只读操作；
    - 常用于前端`UI`查询进度、状态。

8. **`Child Workflow`（子工作流）**

    一个工作流可以启动另一个工作流作为“子工作流”。
    - 子工作流是独立的，有自己的执行历史；
    - 可并行运行多个子工作流；
    - 主工作流可以监听子工作流完成、失败等事件。

9. **`Execution History`（执行历史）**

    `Temporal`会将工作流的每一步操作记录为事件，形成完整的执行日志。
    - 用于恢复（`crash-safe`）、重放、`Debug`；
    - 工作流恢复时会从历史中“回放”之前的操作，重新构建当前状态；
    - 开发者无需手动管理。

## **剖析——Temporal原理**
在业务模块中按规则编写`Workflow`流程以及其具体的`Activity`，并注册到`worker`中，启动`worker`，当外部用户触发`Workflow`，`Temporal`编排`workflow`形成⼀系列的`task`送到队列中，`worker`去队列取任务，执行后将结果返回给`Temporal`。

### **具体的流程描述**
1. 启动`Temporal Server`；
2. 启动`Worker`监听`Temporal Server`，循环获取待执行的工作流；
3. `Starter`创建一个工作流，封装参数，调用`sdk`的`api`发送到`Temporal Server`；
4. `Worker`拉取到工作流开始逻辑处理。

![Desktop View](/assets/img/20250930/temporal_data_flow.png){: width="500" height="300" }
_Temporal 具体流程（图片来自网络）_

### **Workflow中编排的Activity是怎么执行的？**
1. `Workflow`代码调用`Activity`
- 你的`Workflow`代码里写调用`Activity`的代码（比如`Java`里通过`ActivityStub`调用）。
- 这时，`Workflow Worker`并不是立即执行`Activity`代码，而是把调用请求封装成一个任务（`ActivityTask`）。
2. `Workflow Worker`将`Activity`调度请求发送给`Temporal Server`
- `Workflow Worker`通过`RPC`调用，将`Activity`执行请求（包括方法名、参数等）发送给`Temporal Server`。
- `Temporal Server`把这个`Activity`任务写入事件历史和对应的`Task Queue`。
3. `Activity Worker`监听`Task Queue`执行`Activity`
- 专门跑`Activity`代码的`Worker`（`Activity Worker`）会监听这个`Task Queue`。
- 它接收到`Activity`任务后，调用对应的`Activity`实现代码，完成具体业务操作。
4. `Activity`执行结果返回给`Temporal Server`
- `Activity`执行完成后，结果（成功或失败）通过`RPC`返回给`Temporal Server`。
- `Temporal Server`更新事件历史，记录`Activity`执行状态和结果。
5. `Temporal Server`通知`Workflow Worker`继续执行
- `Temporal Server`通知`Workflow Worker Activity`执行完成。
- `Workflow Worker`继续重放`Workflow`代码，拿到`Activity`结果后，继续往下走。

### **Temporal部署架构**

![Desktop View](/assets/img/20250930/deploy_self_hosted_temporal_service.png){: width="700" height="400" }
_Connect your application instances to your self-hosted Temporal Service（来自官方文档）_

![Desktop View](/assets/img/20250930/deploy_temporal_cloud.png){: width="700" height="400" }
_Connect your application instances to Temporal Cloud（来自官方文档）_

具体参见[Temporal生产环境部署手册](https://docs.temporal.io/production-deployment)

### **从部署角度出发，看Temporal Service核心组件**
#### **Temporal Service**
`Temporal`集群是具备持久特性的，由四个独立的可扩展服务组成，
- `Frontend gateway`: 限速、路由、授权
- `History subsystem`: 维护一些数据(`mutable state`, `queues`, `and timers`)
- `Matching subsystem`: `hosts Task Queues for dispatching`
- `Worker service`: `for internal background workflows`
例如，在实际生产部署中，每个集群可以有`5`个`Frontend`、`15`个`History`、`17`个`Matching`和`3`个`Worker`服务。

![Desktop View](/assets/img/20250930/temporal_service.png){: width="700" height="400" }
_Temporal Service（来自官方文档）_

`Temporal`服务可以独立运行，也可以在一个或多个物理或虚拟机上组合成共享进程。对于实时(生产)环境，建议每个服务独立运行，因为每个服务都有不同的伸缩需求，并且故障排除变得更容易。`History`、`Matching`和`Worker`服务可以在集群内可以水平扩展，`Frontend`服务的可伸缩性与其他服务不同，因为它没有分片/分区，只是无状态的。

每个服务通过`Ringpop`的成员协议感知其他服务，包括扩展的实例。

具体参见[Temporal Service手册](https://docs.temporal.io/temporal-service)

#### **Temporal Server**

##### **Frontend Service**
前端服务是一个公开强类型`Proto API`的无状态网关服务。前端服务负责速率限制、授权、验证和路由请求。

请求包括以下几种，
- Domain CRUD
- External events
- Worker polls
- Visibility requests
- Admin operations via the CLI
- Multi-cluster Replication related calls from a remote Cluster

与`Workflow Execution`相关的每个请求都必须有一个`Workflow Id`，为了路由目的，该`Id`被散列处理。`Frontend Service`可以访问维护服务成员信息的散列环，包括集群中有多少节点(每个服务的实例)。所有请求的限速适用于每个`host`、每个`namespace`。

前端服务能够和`Matching service`, `History service`, `Worker service`, `the database`, `and Elasticsearch`（如果用到的话）通信。

它使用`grpcPort 7233`来托管服务处理程序。

它使用`6933`端口进行成员相关的通信

![Desktop View](/assets/img/20250930/temporal_frontend_service.png){: width="700" height="400" }
_Temporal Frontend Service（来自官方文档）_

##### **History Service**

追踪工作流执行的状态

`History service`通过单个分片水平扩展，这些分片是在集群创建期间配置的。在集群的生命周期内，分片的数量保持不变(因此您应该计划扩展和过度供应)。

每个分片维护一些数据(例如路由`id`、可变的状态)和队列。`History`分片维护的队列有三种类型，
- 转移队列：用于将内部任务转移到Matching Service。每当需要调度一个新的工作流任务时，History服务就会以事务方式将其分派给Matching服务。
- 计时器队列：用于持久地持久化计时器。
- Replicator队列：这只用于实验性的多集群特性。

`History`服务与`Matching`服务、数据库通信。

它使用`grpcPort` `7234`来托管服务处理程序。

它使用`6934`端口进行成员相关的通信。

![Desktop View](/assets/img/20250930/temporal_history_service.png){: width="700" height="400" }
_Temporal History Service（来自官方文档）_

##### **Matching Service**

`Matching`服务负责托管任务调度的任务队列。
它负责将`worker`与`Tasks`匹配，并将新任务路由到适当的队列。该服务可以通过拥有多个实例实现内部扩展。

它与前端服务、历史服务和数据库通信。

它使用`grpcPort` `7235`来托管服务处理程序。

它使用`6935`端口进行成员相关的通信。

![Desktop View](/assets/img/20250930/temporal_matching_service.png){: width="700" height="400" }
_Temporal Matching Service（来自官方文档）_

##### **Worker Service**

`Worker Service`为复制队列、系统工作流以及`1.5.0`以上版本的`Kafka`可视性处理器运行后台处理。

它与`Frontend`服务对话。

它使用`grpcPort` `7239`来托管服务处理程序。

它使用`6939`端口进行成员相关的通信。

![Desktop View](/assets/img/20250930/temporal_worker_service.png){: width="700" height="400" }
_Temporal Worker Service（来自官方文档）_

#### **Database 持久化**
数据库为系统提供存储。

支持`Cassandra`、`MySQL`和`PostgreSQL`模式，因此可以用作服务器的数据库。

数据库存储的数据类型如下，
- 任务：待分配的任务。
- 工作流执行的状态
    - 执行表：工作流执行的可变状态的捕获。
    - 历史记录表：仅追加工作流执行历史事件的日志。
- Namespace元数据：集群中各Namespace的元数据。
- 可见性数据：允许“显示所有正在运行的工作流执行”这样的操作。 对于生产环境，建议使用ElasticSearch。

![Desktop View](/assets/img/20250930/temporal_persistent.png){: width="700" height="400" }
_Temporal Database持久化（来自官方文档）_

#### **Visibility 可视化**
在`Temporal`平台中，[Visibility](https://docs.temporal.io/visibility)指的是那些能够使操作员查看、过滤并搜索当前存在于`Temporal`服务中的`Workflow`执行记录的子系统与`API`。
`Temporal`服务中的可见性存储用于保存持久化的`Workflow`执行事件历史数据，并作为持久化存储的一部分进行配置，以便能够列出并过滤`Temporal`服务上存在的`Workflow`执行详情。

- [如何设置可见性存储](https://docs.temporal.io/self-hosted-guide/visibility)

在`Temporal Server v1.21`版本中，您可以通过设置[双可见性](https://docs.temporal.io/dual-visibility)功能，将您的可见性存储从一个数据库迁移到另一个数据库。
从`Temporal Server v1.21`版本开始，对独立标准版和高级版可见性设置的支持将被弃用。请查看[支持的数据库](https://docs.temporal.io/self-hosted-guide/visibility)以获取更新信息。

#### **Archival 归档**

什么是`Archival`（归档）？

`Archival`功能可将`Temporal`服务持久化存储中的[事件历史](https://docs.temporal.io/workflow-execution/event#event-history)和可见性记录自动备份至自定义的二进制大对象存储。

- [如何创建自定义压缩工具](https://docs.temporal.io/self-hosted-guide/archival#custom-archiver)
- [如何设置归档](https://docs.temporal.io/self-hosted-guide/archival#set-up-archival)

工作流执行事件历史记录在[保留期](https://docs.temporal.io/temporal-service/temporal-server#retention-period)结束后会进行备份。当工作流执行状态变为已关闭时，可见性记录会立即进行备份。

归档功能确保工作流执行数据能够按需持久化存储，同时避免对`Temporal`服务的持久化存储造成过大压力。
此功能有助于合规性和故障排除。

`Temporal`的存档功能被视为实验性功能，不适用常规[版本管理和支持政策](https://docs.temporal.io/temporal-service/temporal-server#versions-and-support)。

通过`Docker`运行`Temporal`时，不支持归档功能。手动安装系统或通过[`Helm`图表](https://github.com/temporalio/helm-charts/blob/main/charts/temporal/templates/server-configmap.yaml)部署时，该功能默认禁用。可通过[配置文件](https://github.com/temporalio/temporal/blob/main/config/development.yaml)启用。

#### **Configuration 配置**

[Temporal服务配置](https://docs.temporal.io/temporal-service/configuration)是您自托管`Temporal`服务的设置与配置详情，采用`YAML`格式定义。在部署自托管`Temporal`服务时，您必须定义其配置。

有关使用`Temporal Cloud`的详细信息，请参阅[`Temporal Cloud`文档](https://docs.temporal.io/cloud)。

`Temporal Service`配置由两种类型的配置组成：[静态配置](https://docs.temporal.io/temporal-service/configuration#static-configuration)和[动态配置](https://docs.temporal.io/temporal-service/configuration#dynamic-configuration)。

通过加密网络通信并为`API`调用设置身份验证和授权协议，确保`Temporal`服务（自托管和`Temporal Cloud`）的安全。有关设置`Temporal Service`安全性的详细信息，请参阅[`Temporal Platform`安全功能](https://docs.temporal.io/security)。

#### **Observability 可观测性**

您可以使用自托管的`Temporal Service`或`Temporal Cloud`发出的指标来监控和观察性能。

默认情况下，`Temporal`以`Prometheus`支持的格式发出`metrics`。可以使用支持相同格式的任何`metrics`软件。目前，我们使用以下`Prometheus`和`Grafana`版本进行测试，
- `Prometheus >= v2.0`
- `Grafana >= v2.5`

`Temporal Cloud`通过`Prometheus HTTP API`端点发出`metrics`，该端点可以直接用作`Grafana`中的`Prometheus`数据源，也可以查询`Cloud metrics`并将其导出到任何可观测性平台。有关云指标和设置的详细信息，请参阅以下内容：
- [Temporal Cloud metrics参考](https://docs.temporal.io/cloud/metrics)
- [用Temporal Cloud 可观测性设置Grafana来查看metrics](https://docs.temporal.io/cloud/metrics/prometheus-grafana#grafana-data-sources-configuration)

在自托管的`Temporal Service`上，在`Temporal Service configuration`中公开`Prometheus`端点，并配置`Prometheus`从端点中抓取指标。然后，您可以设置您的可观测性平台（如`Grafana`），使用`Prometheus`作为数据源。有关自托管时态服务度量和设置的详细信息，请参阅以下内容：
- [Temporal Service OSS指标参考](https://docs.temporal.io/references/cluster-metrics)
- [设置Prometheus和Grafana查看SDK和自托管的Temporal Service指标](https://docs.temporal.io/self-hosted-guide/monitoring)

#### **Multi-Cluster Replication**
多集群复制(`Multi-Cluster Replication`)是一种将`Workflow Executions`从活动集群异步复制到其他被动集群的功能，用于备份和状态重建。必要时，为了获得更高的可用性，集群操作可以故障转移到任何备份集群。

`Temporal`的多集群复制功能被认为是实验性的，不受正常[版本控制和支持策略的约束](https://docs.temporal.io/temporal-service/temporal-server#versions-and-support)。

`Temporal`会自动将`Start`、`Signal`和`Query`请求转发到活动集群。必须通过每个[全局命名空间](https://docs.temporal.io/global-namespace)的动态配置标志启用此功能。启用该功能后，任务将被发送到与该命名空间匹配的父任务队列分区（如果存在）。

所有可见性`API`都可以用于活动和备用集群。这使得[`Temporal UI`](https://docs.temporal.io/web-ui)能够无缝地用于全局命名空间。即使全局命名空间处于待机模式，直接对`Temporal Visibility API`进行`API`调用的应用程序仍将继续工作。但是，在从备用集群查询`Workflow Execution`状态时，由于复制延迟，可能会看到延迟。

## **实践——Temporal 部署**
### **几种部署搭建方式**
#### **1.本地调试，使用Temporalite**
本地调试一般没有性能和稳定性要求，可以下载运行`All in one`的`Binary`：[Temporalite](https://github.com/temporalio/temporalite-archived)。

下载`Binary`添加到`Path`环境变量后，一行命令启动就能连`localhost` `7233`端口使用`Temporal`服务，也可以打开浏览器进入`Dashboard`查看运行状态：`http://127.0.0.1:8233`。
```terminal
$ temporalite start --namespace default
```
#### **2.开发或生产环境Helm部署分布式Temporal**
`Dev/Prod`环境建议用`Helm + Kubernetes`部署，存储层准备独立运维的`MySQL`或`PostgreSQL`。
提前创建好数据库，等`Helm install`完成后，通过`admintools`进去初始化数据库`Schema`，

```terminal
helm dependency build  # optional

helm install -f values/values.mysql.yaml my-temporal . \
  --namespace temporal --create-namespace=true
  --kube-context *** \
  --set elasticsearch.enabled=false \
  --set server.config.persistence.default.sql.user=*** \
  --set server.config.persistence.default.sql.password="***" \
  --set server.config.persistence.visibility.sql.user=*** |
  --set server.config.persistence.visibility.sql.password="***" \
  --set server.config.persistence.default.sql.host=*** \
  --set server.config.persistence.visibility.sql.host=***

# 更新版本执行 helm upgrade 同理
```

安装如果遇到`helm dependency`的问题，可以注释掉`Prometheus`、`Grafana`、`ES`等没有用到的依赖`Chart`。
注意：如果连接`AWS Aurora`数据库，需要在`values.mysql.yaml`下面需要加上`connectAttributes`，
```yaml
server:
  config:
    persistence:
      default:
        sql:
          connectAttributes:
            tx_isolation: 'READ-COMMITTED'
```

第一次`Install`后，`admintools`这个`Pod`会正常运行，其他`Pod`会找不到数据库表失败，这时可以进去`admintools`的`Pod shell`，执行命令更新`DB Schema`，参见[`Schema`的源文件](https://github.com/temporalio/temporal/tree/master/schema/mysql/v57)。

```terminal
export SQL_PLUGIN=mysql
export SQL_HOST=mysql_host
export SQL_PORT=3306
export SQL_USER=mysql_user
export SQL_PASSWORD=mysql_password

cd /etc/temporal/schema/mysql/v57/temporal
temporal-sql-tool --connect-attributes "tx_isolation=READ-COMMITTED" --ep mysql-endpoint -u *** --password "***" --db temporal setup-schema -o -v 0.0 -f ./schema.sql

cd /etc/temporal/schema/mysql/v57/visibility
temporal-sql-tool --connect-attributes "tx_isolation=READ-COMMITTED" --ep mysql-endpoint -u *** --password "***" --db temporal_visibility setup-schema -o -v 0.0 -f ./schema.sql
```

等其他`Pod`自动重启或手动删除后，所有`Temporal`组件都会正常运行，可以`Forward`一个`temporal-web`的`8080`端口，检查`Temporal`服务是否运行正常。

#### **3. Docker Compose部署**

不管是本地调试，还是`Dev/Prod`环境部署，都可以用`docker compose`部署，配置可以参考[temporalio/docker-compose](https://github.com/temporalio/docker-compose)。如果官方镜像不能满足你的需求，还可以使用[temporalio/docker-builds](https://github.com/temporalio/docker-builds)定制自己的`docker`镜像。

##### **存储层**

拿默认的`docker-compose-mysql.yml`配置文件举例，该配置中的持久化存储层只用到了`MySQL`，每次执行`docker-compose up -d`命令启动`docker`容器时，都会重新启动`MySQL`容器和`Temporal`相关容器，而在启动`Temporal Server`容器时，默认都会重新初始化`MySQL Schema`和对应元数据。

但这样的数据配置仅能满足`Temporal`功能体验的需求，实际项目中为了保证数据的完整性和安全性，都会选择单独运维`Temporal MySQL`数据。这样一来就得修改`yaml`配置去适配，比如去掉`MySQL`容器的配置，选择单独运维`MySQL`；在`Temporal`容器配置中指定对应的`DB ip`、`port`、`user`、`pwd`等配置信息，同时指定`SKIP_SCHEMA_SETUP=true`跳过`MySQL schema`构建环节等。修改后的配置示例如下，

```yaml
version: "3.5"
services:
  # mysql:
  #   container_name: temporal-mysql-5.7
  #   environment:
  #     - MYSQL_ROOT_PASSWORD=root
  #   image: mysql:${MYSQL_VERSION}
  #   networks:
  #     - temporal-network
  #   ports:
  #     - 3306:3306
  #   volumes:
  #     - /var/lib/mysql
  temporal:
    container_name: temporal-v1.23.1
    # depends_on:
    #   - mysql
    environment:
      - DB=mysql
      - DB_PORT=3306
      - MYSQL_USER=root
      - MYSQL_PWD=123456
      # - MYSQL_SEEDS=mysql
      - MYSQL_SEEDS=172.31.75.79
      - DYNAMIC_CONFIG_FILE_PATH=config/dynamicconfig/development-sql.yaml
      - SKIP_SCHEMA_SETUP=true
      - LOG_LEVEL=debug,info
      # - TEMPORAL_CLI_SHOW_STACKS=1
    # image: temporalio/auto-setup:${TEMPORAL_VERSION}
    image: temporalio/auto-setup-my:${TEMPORAL_VERSION}
    networks:
      - temporal-network
    ports:
      - 7233:7233
    volumes:
      - ./dynamicconfig:/etc/temporal/config/dynamicconfig
    labels:
      kompose.volume.type: configMap
  temporal-admin-tools:
    container_name: temporal-admin-tools-v1.23.0
    depends_on:
      - temporal
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - TEMPORAL_CLI_ADDRESS=temporal:7233
    image: temporalio/admin-tools:${TEMPORAL_ADMINTOOLS_VERSION}
    networks:
      - temporal-network
    stdin_open: true
    tty: true
  temporal-ui:
    container_name: temporal-ui-v2.17.1
    depends_on:
      - temporal
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - TEMPORAL_CORS_ORIGINS=http://localhost:3000
    image: temporalio/ui:${TEMPORAL_UI_VERSION}
    networks:
      - temporal-network
    ports:
      - 8080:8080
networks:
  temporal-network:
    driver: bridge
    name: temporal-network
```

>注：`Temporal Server`连接`MySQL`需要使用的连接信息，参见[SQL Configuration](https://docs.temporal.io/references/configuration#sql)。
![Desktop View](/assets/img/20250930/temporal_mysql_config_info.png){: width="800" height="500" }
_Temporal MySQL Connection Config（来自官方文档）_
{: .prompt-tip }

>如果公司`MySQL`使用了`Atlas proxy`，上面指定的`ip+port`是`proxy`的，在使用`proxy`连接操作`MySQL`时，会报错，
```markdown
Error Error 1243: Unknown prepared statement handler (11) given to mysqld_stmt_execute
```
但有趣的是，直连`MySQL`就没有报错，排查后的结论是，`Atlas proxy`不支持`go prepare`。
{: .prompt-tip }

独立运维`MySQL`存储组件，需要提前准备好`SQL`来创建数据库表，详细`SQL`如下，

```sql
# temporal database

CREATE DATABASE temporal character set utf8;

CREATE TABLE `temporal`.`activity_info_maps` (
`shard_id` int(11) NOT NULL,
`namespace_id` binary(16) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`run_id` binary(16) NOT NULL,
`schedule_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) DEFAULT NULL,
PRIMARY KEY (`shard_id`,`namespace_id`,`workflow_id`,`run_id`,`schedule_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`buffered_events` (
`shard_id` int(11) NOT NULL,
`namespace_id` binary(16) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`run_id` binary(16) NOT NULL,
`id` bigint(20) NOT NULL AUTO_INCREMENT,
`data` mediumblob NOT NULL,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`shard_id`,`namespace_id`,`workflow_id`,`run_id`,`id`),
UNIQUE KEY `id` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`build_id_to_task_queue` (
`namespace_id` binary(16) NOT NULL,
`build_id` varchar(255) NOT NULL,
`task_queue_name` varchar(255) NOT NULL,
PRIMARY KEY (`namespace_id`,`build_id`,`task_queue_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`child_execution_info_maps` (
`shard_id` int(11) NOT NULL,
`namespace_id` binary(16) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`run_id` binary(16) NOT NULL,
`initiated_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) DEFAULT NULL,
PRIMARY KEY (`shard_id`,`namespace_id`,`workflow_id`,`run_id`,`initiated_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`cluster_membership` (
`membership_partition` int(11) NOT NULL,
`host_id` binary(16) NOT NULL,
`rpc_address` varchar(128) DEFAULT NULL,
`rpc_port` smallint(6) NOT NULL,
`role` tinyint(4) NOT NULL,
`session_start` timestamp NOT NULL DEFAULT '1970-01-02 00:00:01',
`last_heartbeat` timestamp NOT NULL DEFAULT '1970-01-02 00:00:01',
`record_expiry` timestamp NOT NULL DEFAULT '1970-01-02 00:00:01',
PRIMARY KEY (`membership_partition`,`host_id`),
KEY `role` (`role`,`host_id`),
KEY `role_2` (`role`,`last_heartbeat`),
KEY `rpc_address` (`rpc_address`,`role`),
KEY `last_heartbeat` (`last_heartbeat`),
KEY `record_expiry` (`record_expiry`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`cluster_metadata` (
`metadata_partition` int(11) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL DEFAULT 'Proto3',
`version` bigint(20) NOT NULL DEFAULT '1',
PRIMARY KEY (`metadata_partition`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`cluster_metadata_info` (
`metadata_partition` int(11) NOT NULL,
`cluster_name` varchar(255) NOT NULL,
`data` mediumblob NOT NULL,
`data_encoding` varchar(16) NOT NULL,
`version` bigint(20) NOT NULL,
PRIMARY KEY (`metadata_partition`,`cluster_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`current_executions` (
`shard_id` int(11) NOT NULL,
`namespace_id` binary(16) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`run_id` binary(16) NOT NULL,
`create_request_id` varchar(255) DEFAULT NULL,
`state` int(11) NOT NULL,
`status` int(11) NOT NULL,
`start_version` bigint(20) NOT NULL DEFAULT '0',
`last_write_version` bigint(20) NOT NULL,
PRIMARY KEY (`shard_id`,`namespace_id`,`workflow_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`executions` (
`shard_id` int(11) NOT NULL,
`namespace_id` binary(16) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`run_id` binary(16) NOT NULL,
`next_event_id` bigint(20) NOT NULL,
`last_write_version` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
`state` mediumblob,
`state_encoding` varchar(16) NOT NULL,
`db_record_version` bigint(20) NOT NULL DEFAULT '0',
PRIMARY KEY (`shard_id`,`namespace_id`,`workflow_id`,`run_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`history_immediate_tasks` (
`shard_id` int(11) NOT NULL,
`category_id` int(11) NOT NULL,
`task_id` bigint(20) NOT NULL,
`data` mediumblob NOT NULL,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`shard_id`,`category_id`,`task_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`history_node` (
`shard_id` int(11) NOT NULL,
`tree_id` binary(16) NOT NULL,
`branch_id` binary(16) NOT NULL,
`node_id` bigint(20) NOT NULL,
`txn_id` bigint(20) NOT NULL,
`data` mediumblob NOT NULL,
`data_encoding` varchar(16) NOT NULL,
`prev_txn_id` bigint(20) NOT NULL DEFAULT '0',
PRIMARY KEY (`shard_id`,`tree_id`,`branch_id`,`node_id`,`txn_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`history_scheduled_tasks` (
`shard_id` int(11) NOT NULL,
`category_id` int(11) NOT NULL,
`visibility_timestamp` datetime(6) NOT NULL,
`task_id` bigint(20) NOT NULL,
`data` mediumblob NOT NULL,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`shard_id`,`category_id`,`visibility_timestamp`,`task_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`history_tree` (
`shard_id` int(11) NOT NULL,
`tree_id` binary(16) NOT NULL,
`branch_id` binary(16) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`shard_id`,`tree_id`,`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`namespace_metadata` (
`partition_id` int(11) NOT NULL,
`notification_version` bigint(20) NOT NULL,
PRIMARY KEY (`partition_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`namespaces` (
`partition_id` int(11) NOT NULL,
`id` binary(16) NOT NULL,
`name` varchar(255) NOT NULL,
`notification_version` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
`is_global` tinyint(1) NOT NULL,
PRIMARY KEY (`partition_id`,`id`),
UNIQUE KEY `name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`queue` (
`queue_type` int(11) NOT NULL,
`message_id` bigint(20) NOT NULL,
`message_payload` mediumblob,
`message_encoding` varchar(16) NOT NULL DEFAULT 'Json',
PRIMARY KEY (`queue_type`,`message_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`queue_messages` (
`queue_type` int(11) NOT NULL,
`queue_name` varchar(255) NOT NULL,
`queue_partition` bigint(20) NOT NULL,
`message_id` bigint(20) NOT NULL,
`message_payload` mediumblob NOT NULL,
`message_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`queue_type`,`queue_name`,`queue_partition`,`message_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`queue_metadata` (
`queue_type` int(11) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL DEFAULT 'Json',
`version` bigint(20) NOT NULL DEFAULT '0',
PRIMARY KEY (`queue_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`queues` (
`queue_type` int(11) NOT NULL,
`queue_name` varchar(255) NOT NULL,
`metadata_payload` mediumblob NOT NULL,
`metadata_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`queue_type`,`queue_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`replication_tasks` (
`shard_id` int(11) NOT NULL,
`task_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`shard_id`,`task_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`replication_tasks_dlq` (
`source_cluster_name` varchar(255) NOT NULL,
`shard_id` int(11) NOT NULL,
`task_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`source_cluster_name`,`shard_id`,`task_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`request_cancel_info_maps` (
`shard_id` int(11) NOT NULL,
`namespace_id` binary(16) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`run_id` binary(16) NOT NULL,
`initiated_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) DEFAULT NULL,
PRIMARY KEY (`shard_id`,`namespace_id`,`workflow_id`,`run_id`,`initiated_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`schema_update_history` (
`version_partition` int(11) NOT NULL,
`year` int(11) NOT NULL,
`month` int(11) NOT NULL,
`update_time` datetime(6) NOT NULL,
`description` varchar(255) DEFAULT NULL,
`manifest_md5` varchar(64) DEFAULT NULL,
`new_version` varchar(64) DEFAULT NULL,
`old_version` varchar(64) DEFAULT NULL,
PRIMARY KEY (`version_partition`,`year`,`month`,`update_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`schema_version` (
`version_partition` int(11) NOT NULL,
`db_name` varchar(255) NOT NULL,
`creation_time` datetime(6) DEFAULT NULL,
`curr_version` varchar(64) DEFAULT NULL,
`min_compatible_version` varchar(64) DEFAULT NULL,
PRIMARY KEY (`version_partition`,`db_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `temporal`.`schema_version` (version_partition, db_name, creation_time, curr_version, min_compatible_version) VALUES (0, "temporal", CURRENT_TIMESTAMP, "1.11", "1.0");

CREATE TABLE `temporal`.`shards` (
`shard_id` int(11) NOT NULL,
`range_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`shard_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`signal_info_maps` (
`shard_id` int(11) NOT NULL,
`namespace_id` binary(16) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`run_id` binary(16) NOT NULL,
`initiated_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) DEFAULT NULL,
PRIMARY KEY (`shard_id`,`namespace_id`,`workflow_id`,`run_id`,`initiated_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`signals_requested_sets` (
`shard_id` int(11) NOT NULL,
`namespace_id` binary(16) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`run_id` binary(16) NOT NULL,
`signal_id` varchar(255) NOT NULL,
PRIMARY KEY (`shard_id`,`namespace_id`,`workflow_id`,`run_id`,`signal_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`task_queue_user_data` (
`namespace_id` binary(16) NOT NULL,
`task_queue_name` varchar(255) NOT NULL,
`data` mediumblob NOT NULL,
`data_encoding` varchar(16) NOT NULL,
`version` bigint(20) NOT NULL,
PRIMARY KEY (`namespace_id`,`task_queue_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`task_queues` (
`range_hash` int(10) unsigned NOT NULL,
`task_queue_id` varbinary(272) NOT NULL,
`range_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`range_hash`,`task_queue_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`tasks` (
`range_hash` int(10) unsigned NOT NULL,
`task_queue_id` varbinary(272) NOT NULL,
`task_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`range_hash`,`task_queue_id`,`task_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`timer_info_maps` (
`shard_id` int(11) NOT NULL,
`namespace_id` binary(16) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`run_id` binary(16) NOT NULL,
`timer_id` varchar(255) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) DEFAULT NULL,
PRIMARY KEY (`shard_id`,`namespace_id`,`workflow_id`,`run_id`,`timer_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`timer_tasks` (
`shard_id` int(11) NOT NULL,
`visibility_timestamp` datetime(6) NOT NULL,
`task_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`shard_id`,`visibility_timestamp`,`task_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`transfer_tasks` (
`shard_id` int(11) NOT NULL,
`task_id` bigint(20) NOT NULL,
`data` mediumblob,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`shard_id`,`task_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal`.`visibility_tasks` (
`shard_id` int(11) NOT NULL,
`task_id` bigint(20) NOT NULL,
`data` mediumblob NOT NULL,
`data_encoding` varchar(16) NOT NULL,
PRIMARY KEY (`shard_id`,`task_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `temporal`.`schema_update_history` (version_partition, year, month, update_time, description, manifest_md5, new_version, old_version) VALUES 
(0, 2025, 9, CURRENT_TIMESTAMP, "initial version", "", "0.0", "0"),
(0, 2025, 9, CURRENT_TIMESTAMP+1, "base version of schema", "55b84ca114ac34d84bdc5f52c198fa33", "1.0", "0.0"),
(0, 2025, 9, CURRENT_TIMESTAMP+2, "schema update for cluster metadata", "58f06841bbb187cb210db32a090c21ee", "1.1", "1.0"),
(0, 2025, 9, CURRENT_TIMESTAMP+3, "schema update for RPC replication and blob size adjustments", "d0980c1ffb9ffa6e3ab6f84e285ffa9d", "1.2", "1.1"),
(0, 2025, 9, CURRENT_TIMESTAMP+4, "schema update for kafka deprecation", "3beee7d470421674194475f94b58d89b", "1.3", "1.2"),
(0, 2025, 9, CURRENT_TIMESTAMP+5, "schema update for cluster metadata cleanup", "c53e2e9cea5660c8a1f3b2ac73cdb138", "1.4", "1.3"),
(0, 2025, 9, CURRENT_TIMESTAMP+6, "schema update for cluster_membership, executions and history_node tables", "bfb307ba10ac0fdec83e0065dc5ffee4", "1.5", "1.4"),
(0, 2025, 9, CURRENT_TIMESTAMP+7, "schema update for queue_metadata", "978e1a6500d377ba91c6e37e5275a59b", "1.6", "1.5"),
(0, 2025, 9, CURRENT_TIMESTAMP+8, "create cluster metadata info table to store cluster information and executions to store tiered storage queue", "366b8b49d6701a6a09778e51ad1682ed", "1.7", "1.6"),
(0, 2025, 9, CURRENT_TIMESTAMP+9, "drop unused tasks table; expand VARCHAR columns governed by maxIDLength to VARCHAR(255)", "bc0761b792a339f7e1e29e00c4fd3e64", "1.8", "1.7"),
(0, 2025, 9, CURRENT_TIMESTAMP+10, "add history tasks table", "b62e4e5826967e152e00b75da42d12ea", "1.9", "1.8"),
(0, 2025, 9, CURRENT_TIMESTAMP+11, "add storage for update records and create task_queue_user_data table", "2b0c361b0d4ab7cf09ead5566f0db520", "1.10", "1.9"),
(0, 2025, 9, CURRENT_TIMESTAMP+12, "add queues and queue_messages tables", "94a91a900aa29ec3d81eb361ab7770be", "1.11", "1.10");

INSERT INTO `temporal`.`namespace_metadata` (partition_id, notification_version) VALUES (54321, 1);
```
{: file='temporal.sql' .nolineno }

```sql
# temporal_visibility database

CREATE DATABASE temporal_visibility character set utf8;

CREATE TABLE `temporal_visibility`.`executions_visibility` (
`namespace_id` char(64) NOT NULL,
`run_id` char(64) NOT NULL,
`start_time` datetime(6) NOT NULL,
`execution_time` datetime(6) NOT NULL,
`workflow_id` varchar(255) NOT NULL,
`workflow_type_name` varchar(255) NOT NULL,
`status` int(11) NOT NULL,
`close_time` datetime(6) DEFAULT NULL,
`history_length` bigint(20) DEFAULT NULL,
`memo` blob,
`encoding` varchar(64) NOT NULL,
`task_queue` varchar(255) NOT NULL DEFAULT '',
PRIMARY KEY (`namespace_id`,`run_id`),
KEY `by_type_start_time` (`namespace_id`,`workflow_type_name`,`status`,`start_time`,`run_id`),
KEY `by_workflow_id_start_time` (`namespace_id`,`workflow_id`,`status`,`start_time`,`run_id`),
KEY `by_status_by_start_time` (`namespace_id`,`status`,`start_time`,`run_id`),
KEY `by_type_close_time` (`namespace_id`,`workflow_type_name`,`status`,`close_time`,`run_id`),
KEY `by_workflow_id_close_time` (`namespace_id`,`workflow_id`,`status`,`close_time`,`run_id`),
KEY `by_status_by_close_time` (`namespace_id`,`status`,`close_time`,`run_id`),
KEY `by_close_time_by_status` (`namespace_id`,`close_time`,`run_id`,`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal_visibility`.`schema_update_history` (
`version_partition` int(11) NOT NULL,
`year` int(11) NOT NULL,
`month` int(11) NOT NULL,
`update_time` datetime(6) NOT NULL,
`description` varchar(255) DEFAULT NULL,
`manifest_md5` varchar(64) DEFAULT NULL,
`new_version` varchar(64) DEFAULT NULL,
`old_version` varchar(64) DEFAULT NULL,
PRIMARY KEY (`version_partition`,`year`,`month`,`update_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `temporal_visibility`.`schema_version` (
`version_partition` int(11) NOT NULL,
`db_name` varchar(255) NOT NULL,
`creation_time` datetime(6) DEFAULT NULL,
`curr_version` varchar(64) DEFAULT NULL,
`min_compatible_version` varchar(64) DEFAULT NULL,
PRIMARY KEY (`version_partition`,`db_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `temporal_visibility`.`schema_version` (version_partition, db_name, creation_time, curr_version, min_compatible_version) VALUES (0, "temporal_visibility", CURRENT_TIMESTAMP, "1.1", "1.0");

INSERT INTO `temporal_visibility`.`schema_update_history` (version_partition, year, month, update_time, description, manifest_md5, new_version, old_version) VALUES 
(0, 2025, 9, CURRENT_TIMESTAMP, "initial version", "", "0.0", "0"),
(0, 2025, 9, CURRENT_TIMESTAMP+1, "base version of visibility schema", "698373883c1c0dd44607a446a62f2a79", "1.0", "0.0"),
(0, 2025, 9, CURRENT_TIMESTAMP+2, "add close time & status index", "e286f8af0a62e291b35189ce29d3fff3", "1.1", "1.0");
```
{: file='temporal_visibility.sql' .nolineno }

### **组件选型**
如果你的公司`MySQL`标准只支持`MySQL 5.7`，不支持`MySQL 8`。为了适配支持`MySQL 5.7`，需要选择支持`MySQL 5.7`的组件最新`Tag`，如下，

|组件                       |tag              |remark          |
| :------------------------| :--------------- | :--------------------------------------- |
|temporal auto-setup	|1.23.1	|`Temporal 1.24.0`（`1.23.1`的下一版本）开始，`temporal`源码`mysql schema`已经没有`5.7`的版本了<br/>[temporalio/temporal](https://github.com/temporalio/temporal.git)<br/>[temporal v1.23.1](https://github.com/temporalio/temporal/tree/fad6bdc0e9c0911f28829f3c47285357554e2567)<br/>[temporal/proto/api](https://github.com/temporalio/api/tree/2e47a76a0ab078dbe9a4e3336304bda6871d46f8)<br/>[docker-build release/v1.23.x](https://github.com/temporalio/docker-builds/tree/release/v1.23.x)|
|temporal-admin-tools  |1.23.0  |`admin-tools 1.23.1`不存在，选择`1.23.0`|
|temporal-ui    |2.17.1    |`UI`从`2.19.0`（`2.17.1`的下一版本）开始，增加`BatchOperation`功能页，这个功能页只支持`MySQL8`，不支持`MySQL 5.7`；<br/>官方和`Temporal 1.23.1`适配的`UI Version`为`2.26.2`，如果可以容忍`BatchOperation`功能页存在，但报错`400`，则可以使用官方适配的`UI`版本。<br/>[temporalio/ui](https://github.com/temporalio/ui)<br/>[temporalio/ui-server](https://github.com/temporalio/ui-server)|

#### **Temporal Github**

|topic|remark|commit_id|
| :--------------------------| :----------------------------- | :--------------------------------------- |
|[Bump Server version to 1.23.0](https://github.com/temporalio/temporal/commit/668a6379f097903082d55286a957fd3e4c01c3d5)|Temporal 1.23.0<br/>支持MySQL 5.7|668a6379f097903082d55286a957fd3e4c01c3d5|
|[Bump Server version to 1.23.1](https://github.com/temporalio/temporal/commit/fad6bdc0e9c0911f28829f3c47285357554e2567)|Temporal 1.23.1<br/>支持MySQL 5.7|fad6bdc0e9c0911f28829f3c47285357554e2567|
|[Bump Server version to 1.24.0](https://github.com/temporalio/temporal/commit/04b6c5e9fcd8ab9d1505c0ccfce802817d0d6e5e)|Temporal 1.24.0<br/>不支持MySQL 5.7|04b6c5e9fcd8ab9d1505c0ccfce802817d0d6e5e|

#### **MySQL 5.7 Schema**

|topic|link|
| :--------------------------| :--------------------------------------- |
|MySQL 5.7 Schema<br/>Temporal 1.23.0|[Temporal 1.23.0 MySQL 5.7 Schema](https://github.com/temporalio/temporal/tree/668a6379f097903082d55286a957fd3e4c01c3d5/schema/mysql/v57)|
|MySQL 5.7 Schema<br/>Temporal 1.23.1|[Temporal 1.23.1 MySQL 5.7 Schema](https://github.com/temporalio/temporal/tree/fad6bdc0e9c0911f28829f3c47285357554e2567/schema/mysql/v57)|

#### **Docker Compose Github**

|topic|remark|commit_id|
| :--------------------------| :----------------------------- | :--------------------------------------- |
|[Update MySQL and PosgreSQL to the latest (#55)](https://github.com/temporalio/docker-compose/commit/79c581ff1fea7afd1938cc9fb3b33ccf434c1108)|升级MySQL版本（5.7->8）<br/>升级postgres版本（9.6->13）<br/>此时Temporal版本如下，<br/>temporalio/auto-setup:1.13.1<br/>temporalio/admin-tools:1.13.1<br/>temporalio/web:1.13.0|79c581ff1fea7afd1938cc9fb3b33ccf434c1108|
|[Elasticsearch should use version 7.16.2 or newer (#57)](https://github.com/temporalio/docker-compose/commit/ad1de17c82e20b833fa48d415506025df1c70f85)|升级ES|ad1de17c82e20b833fa48d415506025df1c70f85|
|[moving to .env file to manage tags (#73)](https://github.com/temporalio/docker-compose/commit/8700c7b0ba0dc8c6491672b39efd5f15b3814484)|在.env统一管理image tag|8700c7b0ba0dc8c6491672b39efd5f15b3814484|
|[Update configs removing standard visibility support (#205)](https://github.com/temporalio/docker-compose/commit/1fff9dd2ffc584ae4be1e8531a83a5905ea12306)|这个commit之后的版本（包含这个版本），docker-compose开始不支持MySQL 5.7<br/>此时Temporal版本如下，<br/>temporalio/auto-setup: 1.23.0<br/>temporal-admin-tools: 1.23.0<br/>temporalio/ui:2.26.2<br/><br/>Temporal 1.23.0的schema还存在MySQL 5.7版本|1fff9dd2ffc584ae4be1e8531a83a5905ea12306|
|[Bump Server version to 1.23.1](https://github.com/temporalio/docker-compose/commit/54934438b8f0f020c9682efd3f2731025ad10dce)|从这个commit开始，docker-compose中只有MySQL8，但是Temporal的schema还存在MySQL 5.7版本<br/>此时Temporal版本如下，<br/>temporalio/auto-setup: 1.23.1<br/>temporal-admin-tools: 1.23.1<br/>temporalio/ui: 2.26.2|54934438b8f0f020c9682efd3f2731025ad10dce|
|[Bump Server version to 1.24.0](https://github.com/temporalio/docker-compose/commit/1f95f487f846c2565f984645b79f3b2c5b469e07)|此时Temporal版本如下，<br/>temporalio/auto-setup: 1.24.0<br/>temporal-admin-tools: 1.24.0-tctl-1.18.1-cli-0.12.0<br/>temporalio/ui: 2.26.2<br/><br/>Temporal 1.24.0的schema已经不存在MySQL 5.7版本了|1f95f487f846c2565f984645b79f3b2c5b469e07<br/>52f74c55f33ba4354bb7c0dbe5691e525ce463cc|

#### **docker-build**
比如上述`Atlas proxy`不支持`go prepare`，需要二次定制开发`temporal`的`MySQL`驱动层代码来进行适配，自然也得重新对`temporal auto-setup`模块重新打镜像。

|[docker-build](https://github.com/temporalio/docker-builds)|docker-build github|
|[docker-build v1.23.x](https://github.com/temporalio/docker-builds/tree/e7a73a0315dafc161dca23e9df78f8355e1e5ae7)|v1.23.x对应的docker-build库|
|[CLI](https://github.com/temporalio/cli/tree/7267df668277a9b43c50d454af50ff5e2b9f8e90)|Use the CLI to run a Temporal Server and interact with it.|
|[tctl](https://github.com/temporalio/tctl/tree/57f54ad72a8170e7e65b04df62c8a906f77df698)|The Temporal CLI is a command-line tool you can use to perform various tasks on a Temporal Server.|

#### **附录**
##### **A. Temporal 1.23.1使用MySQL 5.7，遇到的问题**

|MYSQL_VERSION|5.7|
|TEMPORAL_VERSION|1.23.1|
|TEMPORAL_ADMINTOOLS_VERSION|~~<font color="red">1.23.1</font>~~|
|TEMPORAL_UI_VERSION|2.26.2|

![Desktop View](/assets/img/20250930/appendix_a_pic.png){: width="500" height="300" }
_上述组件版本，启动报错_

`TEMPORAL_ADMINTOOLS_VERSION`改成`1.23.0`后，正常启动，但`BatchOperation`功能页不可用。

|TEMPORAL_ADMINTOOLS_VERSION|<font color="red">1.23.0</font>|

![Desktop View](/assets/img/20250930/appendix_a_batch_operation_error.png){: width="500" height="300" }
_BatchOperation功能页不可用_

MYSQL_VERSION改成v8，BatchOperation没有问题。

|MYSQL_VERSION|<font color="red">8</font>|

![Desktop View](/assets/img/20250930/appendix_a_batch_operation.png){: width="500" height="300" }
_BatchOperation功能正常_

##### **B. Temporal 1.23.0使用MySQL 5.7，遇到的问题**

|MYSQL_VERSION|5.7|
|TEMPORAL_VERSION|1.23.0|
|TEMPORAL_UI_VERSION|2.26.2|

`BatchOperation`功能页不可用。

##### **C. Temporal 1.22.4使用MySQL 5.7，遇到的问题**

|MYSQL_VERSION|5.7|
|TEMPORAL_VERSION|1.22.4|
|TEMPORAL_UI_VERSION|2.21.0|

`BatchOperation`功能页不可用。

##### **D. Temporal 1.22.1使用MySQL 5.7，遇到的问题**

|MYSQL_VERSION|5.7|
|TEMPORAL_VERSION|1.22.1|
|TEMPORAL_UI_VERSION|2.19.0|

`BatchOperation`功能页不可用。

<font color="red">UI从2.19.0开始，增加BatchOperation功能页，而这个功能页只支持MySQL8，不支持MySQL 5.7。</font>

##### **E. Temporal 1.22.0，UI 2.17.1，使用MySQL 5.7**

|MYSQL_VERSION|5.7|
|TEMPORAL_VERSION|1.22.0|
|EMPORAL_UI_VERSION|2.17.1|

![Desktop View](/assets/img/20250930/appendix_e_pic.png){: width="500" height="300" }
_功能正常，缺少BatchOperation功能_

##### **F. Temporal 1.21.0使用MySQL 5.7**

|MYSQL_VERSION|5.7|
|TEMPORAL_VERSION|1.21.0|
|TEMPORAL_UI_VERSION|2.16.1|

![Desktop View](/assets/img/20250930/appendix_f_pic.png){: width="500" height="300" }
_功能正常，功能较少，没有Schedules功能页_

### **部署运行**

#### **1. 验证MySQL schema**

```terminal
mysql> show databases;
+---------------------+
| Database            |
+---------------------+
| information_schema  |
| mysql               |
| performance_schema  |
| sys                 |
| temporal            |
| temporal_visibility |
+---------------------+
10 rows in set (0.01 sec)
mysql> use temporal;
mysql> show tables;
+---------------------------+
| Tables_in_temporal        |
+---------------------------+
| activity_info_maps        |
| buffered_events           |
| build_id_to_task_queue    |
| child_execution_info_maps |
| cluster_membership        |
| cluster_metadata          |
| cluster_metadata_info     |
| current_executions        |
| executions                |
| history_immediate_tasks   |
| history_node              |
| history_scheduled_tasks   |
| history_tree              |
| namespace_metadata        |
| namespaces                |
| queue                     |
| queue_messages            |
| queue_metadata            |
| queues                    |
| replication_tasks         |
| replication_tasks_dlq     |
| request_cancel_info_maps  |
| schema_update_history     |
| schema_version            |
| shards                    |
| signal_info_maps          |
| signals_requested_sets    |
| task_queue_user_data      |
| task_queues               |
| tasks                     |
| timer_info_maps           |
| timer_tasks               |
| transfer_tasks            |
| visibility_tasks          |
+---------------------------+
34 rows in set (0.00 sec)

mysql> use temporal_visibility;
mysql> show tables;
+-------------------------------+
| Tables_in_temporal_visibility |
+-------------------------------+
| executions_visibility         |
| schema_update_history         |
| schema_version                |
+-------------------------------+
3 rows in set (0.00 sec)
```

#### **2. 启动docker，输出下面的log说明成功了**

```
# docker-compose -f docker-compose-mysql.yml up -d
[+] Running 4/4
 ✔ Network temporal-network                Created                                                                                                                                        0.0s
 ✔ Container temporal-v1.23.1              Started                                                                                                                                        0.3s
 ✔ Container temporal-admin-tools-v1.23.0  Started                                                                                                                                        0.4s
 ✔ Container temporal-ui-v2.17.1           Started                                                                                                                                        0.5s
# docker-compose ps
NAME                           IMAGE                             COMMAND                  SERVICE                CREATED         STATUS         PORTS
temporal-admin-tools-v1.23.0   temporalio/admin-tools:1.23.0     "tini -- sleep infin…"   temporal-admin-tools   3 seconds ago   Up 2 seconds
temporal-ui-v2.17.1            temporalio/ui:2.26.2              "./start-ui-server.sh"   temporal-ui            3 seconds ago   Up 2 seconds   0.0.0.0:8080->8080/tcp
temporal-v1.23.1               temporalio/auto-setup-my:1.23.1   "/etc/temporal/entry…"   temporal               3 seconds ago   Up 3 seconds   6933-6935/tcp, 6939/tcp, 7234-7235/tcp, 7239/tcp, 0.0.0.0:7233->7233/tcp
# docker logs temporal-v1.23.1
2025/09/30 09:29:24 Loading config; env=docker,zone=,configDir=config
2025/09/30 09:29:24 Loading config files=[config/docker.yaml]
TEMPORAL_ADDRESS is not set, setting it to 172.19.0.2:7233
Temporal CLI address: 172.19.0.2:7233.
{"level":"info","ts":"2025-09-30T09:29:24.092Z","msg":"Build info.","git-time":"2024-04-30T17:37:12.000Z","git-revision":"e7a73a0315dafc161dca23e9df78f8355e1e5ae7","git-modified":true,"go-arch":"amd64","go-os":"linux","go-version":"go1.24.0","cgo-enabled":false,"server-version":"1.23.1","debug-mode":false,"logging-call-at":"main.go:148"}
{"level":"info","ts":"2025-09-30T09:29:24.096Z","msg":"dynamic config changed for the key: limit.maxidlength oldValue: nil newValue: { constraints: {} value: 255 }","logging-call-at":"file_based_client.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.096Z","msg":"dynamic config changed for the key: system.forcesearchattributescacherefreshonread oldValue: nil newValue: { constraints: {} value: true }","logging-call-at":"file_based_client.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.096Z","msg":"Updated dynamic config","logging-call-at":"file_based_client.go:195"}
{"level":"warn","ts":"2025-09-30T09:29:24.096Z","msg":"Not using any authorizer and flag `--allow-no-auth` not detected. Future versions will require using the flag `--allow-no-auth` if you do not want to set an authorizer.","logging-call-at":"main.go:178"}
Waiting for Temporal server to start...
{"level":"info","ts":"2025-09-30T09:29:24.116Z","msg":"Use rpc address 127.0.0.1:7233 for cluster active.","component":"metadata-initializer","logging-call-at":"fx.go:724"}
{"level":"info","ts":"2025-09-30T09:29:24.176Z","msg":"historyClient: ownership caching disabled","service":"history","logging-call-at":"client.go:91"}
{"level":"info","ts":"2025-09-30T09:29:24.184Z","msg":"creating new visibility manager","visibility_plugin_name":"mysql","visibility_index_name":"","logging-call-at":"factory.go:166"}
{"level":"info","ts":"2025-09-30T09:29:24.187Z","msg":"Initialized lazy loaded OwnershipBasedQuotaScaler","service":"history","service":"history","logging-call-at":"fx.go:95"}
{"level":"info","ts":"2025-09-30T09:29:24.187Z","msg":"Initialized service resolver for persistence rate limiting","service":"history","service":"history","logging-call-at":"fx.go:92"}
{"level":"info","ts":"2025-09-30T09:29:24.187Z","msg":"Created gRPC listener","service":"history","address":"172.19.0.2:7234","logging-call-at":"rpc.go:153"}
{"level":"info","ts":"2025-09-30T09:29:24.197Z","msg":"Initialized service resolver for persistence rate limiting","service":"matching","service":"matching","logging-call-at":"fx.go:92"}
{"level":"info","ts":"2025-09-30T09:29:24.197Z","msg":"Created gRPC listener","service":"matching","address":"172.19.0.2:7235","logging-call-at":"rpc.go:153"}
{"level":"info","ts":"2025-09-30T09:29:24.197Z","msg":"historyClient: ownership caching disabled","service":"matching","logging-call-at":"client.go:91"}
{"level":"info","ts":"2025-09-30T09:29:24.201Z","msg":"creating new visibility manager","visibility_plugin_name":"mysql","visibility_index_name":"","logging-call-at":"factory.go:166"}
{"level":"info","ts":"2025-09-30T09:29:24.212Z","msg":"Initialized service resolver for persistence rate limiting","service":"frontend","service":"frontend","logging-call-at":"fx.go:92"}
{"level":"info","ts":"2025-09-30T09:29:24.212Z","msg":"historyClient: ownership caching disabled","service":"frontend","logging-call-at":"client.go:91"}
{"level":"info","ts":"2025-09-30T09:29:24.212Z","msg":"Created gRPC listener","service":"frontend","address":"172.19.0.2:7233","logging-call-at":"rpc.go:153"}
{"level":"info","ts":"2025-09-30T09:29:24.216Z","msg":"creating new visibility manager","visibility_plugin_name":"mysql","visibility_index_name":"","logging-call-at":"factory.go:166"}
{"level":"info","ts":"2025-09-30T09:29:24.216Z","msg":"Service is not requested, skipping initialization.","service":"internal-frontend","logging-call-at":"fx.go:522"}
{"level":"info","ts":"2025-09-30T09:29:24.223Z","msg":"Initialized service resolver for persistence rate limiting","service":"worker","service":"worker","logging-call-at":"fx.go:92"}
{"level":"info","ts":"2025-09-30T09:29:24.224Z","msg":"historyClient: ownership caching disabled","service":"worker","logging-call-at":"client.go:91"}
{"level":"info","ts":"2025-09-30T09:29:24.226Z","msg":"creating new visibility manager","visibility_plugin_name":"mysql","visibility_index_name":"","logging-call-at":"factory.go:166"}
{"level":"info","ts":"2025-09-30T09:29:24.227Z","msg":"PProf not started due to port not set","logging-call-at":"pprof.go:68"}
{"level":"info","ts":"2025-09-30T09:29:24.227Z","msg":"Starting server for services","value":{"frontend":{},"history":{},"matching":{},"worker":{}},"logging-call-at":"server_impl.go:93"}
{"level":"info","ts":"2025-09-30T09:29:24.234Z","msg":"fifo scheduler started","service":"history","logging-call-at":"fifo_scheduler.go:96"}
{"level":"info","ts":"2025-09-30T09:29:24.234Z","msg":"interleaved weighted round robin task scheduler started","service":"history","logging-call-at":"interleaved_weighted_round_robin.go:129"}
{"level":"info","ts":"2025-09-30T09:29:24.235Z","msg":"fifo scheduler started","service":"history","logging-call-at":"fifo_scheduler.go:96"}
{"level":"info","ts":"2025-09-30T09:29:24.235Z","msg":"interleaved weighted round robin task scheduler started","service":"history","logging-call-at":"interleaved_weighted_round_robin.go:129"}
{"level":"info","ts":"2025-09-30T09:29:24.236Z","msg":"fifo scheduler started","service":"history","component":"memory-scheduled-queue-processor","logging-call-at":"fifo_scheduler.go:96"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"fifo scheduler started","service":"history","logging-call-at":"fifo_scheduler.go:96"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"RuntimeMetricsReporter started","service":"worker","logging-call-at":"runtime.go:138"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"RuntimeMetricsReporter started","service":"frontend","logging-call-at":"runtime.go:138"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"RuntimeMetricsReporter started","service":"matching","logging-call-at":"runtime.go:138"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"worker starting","service":"worker","component":"worker","logging-call-at":"service.go:343"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"frontend starting","service":"frontend","logging-call-at":"service.go:350"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"interleaved weighted round robin task scheduler started","service":"history","logging-call-at":"interleaved_weighted_round_robin.go:129"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"matching starting","service":"matching","logging-call-at":"service.go:90"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"Starting to serve on frontend listener","service":"frontend","logging-call-at":"service.go:368"}
{"level":"info","ts":"2025-09-30T09:29:24.237Z","msg":"Starting to serve on matching listener","service":"matching","logging-call-at":"service.go:102"}
{"level":"info","ts":"2025-09-30T09:29:24.238Z","msg":"fifo scheduler started","service":"history","logging-call-at":"fifo_scheduler.go:96"}
{"level":"info","ts":"2025-09-30T09:29:24.238Z","msg":"interleaved weighted round robin task scheduler started","service":"history","logging-call-at":"interleaved_weighted_round_robin.go:129"}
{"level":"info","ts":"2025-09-30T09:29:24.239Z","msg":"RuntimeMetricsReporter started","service":"history","logging-call-at":"runtime.go:138"}
{"level":"info","ts":"2025-09-30T09:29:24.240Z","msg":"sequential scheduler started","logging-call-at":"sequential_scheduler.go:96"}
{"level":"info","ts":"2025-09-30T09:29:24.240Z","msg":"history starting","service":"history","logging-call-at":"service.go:90"}
{"level":"info","ts":"2025-09-30T09:29:24.240Z","msg":"Replication task fetchers started.","logging-call-at":"task_fetcher.go:142"}
{"level":"info","ts":"2025-09-30T09:29:24.240Z","msg":"none","component":"shard-controller","address":"172.19.0.2:7234","lifecycle":"Started","logging-call-at":"controller_impl.go:135"}
{"level":"info","ts":"2025-09-30T09:29:24.240Z","msg":"Starting to serve on history listener","service":"history","logging-call-at":"service.go:101"}
{"level":"info","ts":"2025-09-30T09:29:24.242Z","msg":"Membership heartbeat upserted successfully","address":"172.19.0.2","port":6935,"hostId":"f3cbc5eb-9ddf-11f0-adc1-a6a295d3aa2b","logging-call-at":"monitor.go:306"}
{"level":"info","ts":"2025-09-30T09:29:24.246Z","msg":"bootstrap hosts fetched","bootstrap-hostports":"172.19.0.2:6934,172.19.0.2:6935","logging-call-at":"monitor.go:348"}
{"level":"info","ts":"2025-09-30T09:29:24.246Z","msg":"Membership heartbeat upserted successfully","address":"172.19.0.2","port":6934,"hostId":"f3c10e5a-9ddf-11f0-adc1-a6a295d3aa2b","logging-call-at":"monitor.go:306"}
{"level":"info","ts":"2025-09-30T09:29:24.248Z","msg":"bootstrap hosts fetched","bootstrap-hostports":"172.19.0.2:6934,172.19.0.2:6935","logging-call-at":"monitor.go:348"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"Current reachable members","component":"service-resolver","service":"history","addresses":["172.19.0.2:7234"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"none","component":"shard-controller","address":"172.19.0.2:7234","component":"shard-controller","address":"172.19.0.2:7234","shard-update":"RingMembershipChangedEvent","number-processed":1,"number-deleted":0,"logging-call-at":"ownership.go:116"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","lifecycle":"Started","component":"shard-context","logging-call-at":"context_impl.go:1563"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"none","component":"shard-controller","address":"172.19.0.2:7234","numShards":1,"logging-call-at":"controller_impl.go:285"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","lifecycle":"Started","component":"shard-context","logging-call-at":"context_impl.go:1563"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"none","component":"shard-controller","address":"172.19.0.2:7234","numShards":2,"logging-call-at":"controller_impl.go:285"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","lifecycle":"Started","component":"shard-context","logging-call-at":"context_impl.go:1563"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"none","component":"shard-controller","address":"172.19.0.2:7234","numShards":3,"logging-call-at":"controller_impl.go:285"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","lifecycle":"Started","component":"shard-context","logging-call-at":"context_impl.go:1563"}
{"level":"info","ts":"2025-09-30T09:29:24.249Z","msg":"none","component":"shard-controller","address":"172.19.0.2:7234","numShards":4,"logging-call-at":"controller_impl.go:285"}
{"level":"info","ts":"2025-09-30T09:29:24.253Z","msg":"Membership heartbeat upserted successfully","address":"172.19.0.2","port":6933,"hostId":"f3ce1151-9ddf-11f0-adc1-a6a295d3aa2b","logging-call-at":"monitor.go:306"}
{"level":"info","ts":"2025-09-30T09:29:24.254Z","msg":"bootstrap hosts fetched","bootstrap-hostports":"172.19.0.2:6935,172.19.0.2:6933,172.19.0.2:6934","logging-call-at":"monitor.go:348"}
{"level":"info","ts":"2025-09-30T09:29:24.256Z","msg":"Current reachable members","component":"service-resolver","service":"history","addresses":["172.19.0.2:7234"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.256Z","msg":"Current reachable members","component":"service-resolver","service":"frontend","addresses":["172.19.0.2:7233"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.256Z","msg":"Frontend is now healthy","service":"frontend","logging-call-at":"workflow_handler.go:227"}
{"level":"info","ts":"2025-09-30T09:29:24.256Z","msg":"Current reachable members","component":"service-resolver","service":"frontend","addresses":["172.19.0.2:7233"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"Range updated for shardID","shard-id":1,"address":"172.19.0.2:7234","shard-range-id":2,"previous-shard-range-id":1,"logging-call-at":"context_impl.go:1190"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"Task key range updated","shard-id":1,"address":"172.19.0.2:7234","number":2097152,"next-number":3145728,"logging-call-at":"task_key_generator.go:177"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"Acquired shard","shard-id":1,"address":"172.19.0.2:7234","logging-call-at":"context_impl.go:1930"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","lifecycle":"Starting","component":"shard-engine","logging-call-at":"context_impl.go:1419"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"history-engine","lifecycle":"Starting","logging-call-at":"history_engine.go:308"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Starting","logging-call-at":"queue_immediate.go:112"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"Task rescheduler started.","shard-id":1,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Started","logging-call-at":"queue_immediate.go:121"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Starting","logging-call-at":"queue_scheduled.go:155"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"Task rescheduler started.","shard-id":1,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Started","logging-call-at":"queue_scheduled.go:164"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Starting","logging-call-at":"queue_immediate.go:112"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"Task rescheduler started.","shard-id":1,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"queue reader started","shard-id":1,"address":"172.19.0.2:7234","component":"visibility-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Started","logging-call-at":"queue_immediate.go:121"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Starting","logging-call-at":"queue_scheduled.go:155"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"Task rescheduler started.","shard-id":1,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Started","logging-call-at":"queue_scheduled.go:164"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","service":"history","component":"memory-scheduled-queue-processor","lifecycle":"Starting","logging-call-at":"memory_scheduled_queue.go:103"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","service":"history","component":"memory-scheduled-queue-processor","lifecycle":"Started","logging-call-at":"memory_scheduled_queue.go:108"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","component":"history-engine","lifecycle":"Started","logging-call-at":"history_engine.go:317"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"none","shard-id":1,"address":"172.19.0.2:7234","lifecycle":"Started","component":"shard-engine","logging-call-at":"context_impl.go:1422"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"queue reader started","shard-id":1,"address":"172.19.0.2:7234","component":"archival-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"queue reader started","shard-id":1,"address":"172.19.0.2:7234","component":"transfer-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.262Z","msg":"queue reader started","shard-id":1,"address":"172.19.0.2:7234","component":"timer-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.277Z","msg":"Membership heartbeat upserted successfully","address":"172.19.0.2","port":6939,"hostId":"f3cfd813-9ddf-11f0-adc1-a6a295d3aa2b","logging-call-at":"monitor.go:306"}
{"level":"info","ts":"2025-09-30T09:29:24.291Z","msg":"Current reachable members","component":"service-resolver","service":"history","addresses":["172.19.0.2:7234"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.291Z","msg":"Current reachable members","component":"service-resolver","service":"frontend","addresses":["172.19.0.2:7233"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.291Z","msg":"Current reachable members","component":"service-resolver","service":"matching","addresses":["172.19.0.2:7235"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.292Z","msg":"Range updated for shardID","shard-id":3,"address":"172.19.0.2:7234","shard-range-id":2,"previous-shard-range-id":1,"logging-call-at":"context_impl.go:1190"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"Task key range updated","shard-id":3,"address":"172.19.0.2:7234","number":2097152,"next-number":3145728,"logging-call-at":"task_key_generator.go:177"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"Acquired shard","shard-id":3,"address":"172.19.0.2:7234","logging-call-at":"context_impl.go:1930"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","lifecycle":"Starting","component":"shard-engine","logging-call-at":"context_impl.go:1419"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"history-engine","lifecycle":"Starting","logging-call-at":"history_engine.go:308"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Starting","logging-call-at":"queue_immediate.go:112"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"Task rescheduler started.","shard-id":3,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Started","logging-call-at":"queue_immediate.go:121"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Starting","logging-call-at":"queue_scheduled.go:155"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"Task rescheduler started.","shard-id":3,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Started","logging-call-at":"queue_scheduled.go:164"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Starting","logging-call-at":"queue_immediate.go:112"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"Task rescheduler started.","shard-id":3,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Started","logging-call-at":"queue_immediate.go:121"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Starting","logging-call-at":"queue_scheduled.go:155"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"Task rescheduler started.","shard-id":3,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Started","logging-call-at":"queue_scheduled.go:164"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","service":"history","component":"memory-scheduled-queue-processor","lifecycle":"Starting","logging-call-at":"memory_scheduled_queue.go:103"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","service":"history","component":"memory-scheduled-queue-processor","lifecycle":"Started","logging-call-at":"memory_scheduled_queue.go:108"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","component":"history-engine","lifecycle":"Started","logging-call-at":"history_engine.go:317"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"none","shard-id":3,"address":"172.19.0.2:7234","lifecycle":"Started","component":"shard-engine","logging-call-at":"context_impl.go:1422"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"queue reader started","shard-id":3,"address":"172.19.0.2:7234","component":"visibility-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"queue reader started","shard-id":3,"address":"172.19.0.2:7234","component":"archival-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"queue reader started","shard-id":3,"address":"172.19.0.2:7234","component":"timer-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.293Z","msg":"queue reader started","shard-id":3,"address":"172.19.0.2:7234","component":"transfer-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.294Z","msg":"bootstrap hosts fetched","bootstrap-hostports":"172.19.0.2:6934,172.19.0.2:6935,172.19.0.2:6933,172.19.0.2:6939","logging-call-at":"monitor.go:348"}
{"level":"info","ts":"2025-09-30T09:29:24.295Z","msg":"Current reachable members","component":"service-resolver","service":"matching","addresses":["172.19.0.2:7235"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.296Z","msg":"Current reachable members","component":"service-resolver","service":"history","addresses":["172.19.0.2:7234"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.296Z","msg":"Current reachable members","component":"service-resolver","service":"worker","addresses":["172.19.0.2:7239"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.296Z","msg":"Current reachable members","component":"service-resolver","service":"frontend","addresses":["172.19.0.2:7233"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.296Z","msg":"Current reachable members","component":"service-resolver","service":"matching","addresses":["172.19.0.2:7235"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.296Z","msg":"Current reachable members","component":"service-resolver","service":"worker","addresses":["172.19.0.2:7239"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.297Z","msg":"Range updated for shardID","shard-id":4,"address":"172.19.0.2:7234","shard-range-id":2,"previous-shard-range-id":1,"logging-call-at":"context_impl.go:1190"}
{"level":"info","ts":"2025-09-30T09:29:24.297Z","msg":"Task key range updated","shard-id":4,"address":"172.19.0.2:7234","number":2097152,"next-number":3145728,"logging-call-at":"task_key_generator.go:177"}
{"level":"info","ts":"2025-09-30T09:29:24.297Z","msg":"Acquired shard","shard-id":4,"address":"172.19.0.2:7234","logging-call-at":"context_impl.go:1930"}
{"level":"info","ts":"2025-09-30T09:29:24.297Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","lifecycle":"Starting","component":"shard-engine","logging-call-at":"context_impl.go:1419"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"history-engine","lifecycle":"Starting","logging-call-at":"history_engine.go:308"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Starting","logging-call-at":"queue_immediate.go:112"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"Task rescheduler started.","shard-id":4,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Started","logging-call-at":"queue_immediate.go:121"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Starting","logging-call-at":"queue_scheduled.go:155"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"Task rescheduler started.","shard-id":4,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Started","logging-call-at":"queue_scheduled.go:164"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Starting","logging-call-at":"queue_immediate.go:112"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"Task rescheduler started.","shard-id":4,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Started","logging-call-at":"queue_immediate.go:121"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Starting","logging-call-at":"queue_scheduled.go:155"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"Task rescheduler started.","shard-id":4,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Started","logging-call-at":"queue_scheduled.go:164"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","service":"history","component":"memory-scheduled-queue-processor","lifecycle":"Starting","logging-call-at":"memory_scheduled_queue.go:103"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","service":"history","component":"memory-scheduled-queue-processor","lifecycle":"Started","logging-call-at":"memory_scheduled_queue.go:108"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","component":"history-engine","lifecycle":"Started","logging-call-at":"history_engine.go:317"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"none","shard-id":4,"address":"172.19.0.2:7234","lifecycle":"Started","component":"shard-engine","logging-call-at":"context_impl.go:1422"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"queue reader started","shard-id":4,"address":"172.19.0.2:7234","component":"archival-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"queue reader started","shard-id":4,"address":"172.19.0.2:7234","component":"timer-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"queue reader started","shard-id":4,"address":"172.19.0.2:7234","component":"visibility-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.298Z","msg":"queue reader started","shard-id":4,"address":"172.19.0.2:7234","component":"transfer-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"Range updated for shardID","shard-id":2,"address":"172.19.0.2:7234","shard-range-id":2,"previous-shard-range-id":1,"logging-call-at":"context_impl.go:1190"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"Task key range updated","shard-id":2,"address":"172.19.0.2:7234","number":2097152,"next-number":3145728,"logging-call-at":"task_key_generator.go:177"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"Acquired shard","shard-id":2,"address":"172.19.0.2:7234","logging-call-at":"context_impl.go:1930"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","lifecycle":"Starting","component":"shard-engine","logging-call-at":"context_impl.go:1419"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"history-engine","lifecycle":"Starting","logging-call-at":"history_engine.go:308"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Starting","logging-call-at":"queue_immediate.go:112"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"Task rescheduler started.","shard-id":2,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"transfer-queue-processor","lifecycle":"Started","logging-call-at":"queue_immediate.go:121"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Starting","logging-call-at":"queue_scheduled.go:155"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"Task rescheduler started.","shard-id":2,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"archival-queue-processor","lifecycle":"Started","logging-call-at":"queue_scheduled.go:164"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","service":"history","component":"memory-scheduled-queue-processor","lifecycle":"Starting","logging-call-at":"memory_scheduled_queue.go:103"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","service":"history","component":"memory-scheduled-queue-processor","lifecycle":"Started","logging-call-at":"memory_scheduled_queue.go:108"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Starting","logging-call-at":"queue_immediate.go:112"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"Task rescheduler started.","shard-id":2,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"visibility-queue-processor","lifecycle":"Started","logging-call-at":"queue_immediate.go:121"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Starting","logging-call-at":"queue_scheduled.go:155"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"Task rescheduler started.","shard-id":2,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Started","logging-call-at":"rescheduler.go:127"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"timer-queue-processor","lifecycle":"Started","logging-call-at":"queue_scheduled.go:164"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","component":"history-engine","lifecycle":"Started","logging-call-at":"history_engine.go:317"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"none","shard-id":2,"address":"172.19.0.2:7234","lifecycle":"Started","component":"shard-engine","logging-call-at":"context_impl.go:1422"}
{"level":"info","ts":"2025-09-30T09:29:24.303Z","msg":"queue reader started","shard-id":2,"address":"172.19.0.2:7234","component":"transfer-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.304Z","msg":"queue reader started","shard-id":2,"address":"172.19.0.2:7234","component":"archival-queue-processor","queue-reader-id":0,"lifecycle":"Started","logging-call-at":"reader.go:182"}
{"level":"info","ts":"2025-09-30T09:29:24.316Z","msg":"temporal-sys-tq-scanner-workflow workflow successfully started","service":"worker","logging-call-at":"scanner.go:292"}
{"level":"info","ts":"2025-09-30T09:29:24.368Z","msg":"Current reachable members","component":"service-resolver","service":"matching","addresses":["172.19.0.2:7235"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.368Z","msg":"Current reachable members","component":"service-resolver","service":"worker","addresses":["172.19.0.2:7239"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.375Z","msg":"Current reachable members","component":"service-resolver","service":"worker","addresses":["172.19.0.2:7239"],"logging-call-at":"service_resolver.go:275"}
{"level":"info","ts":"2025-09-30T09:29:24.407Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"f5f2d9e27d78:e4b67e5f-a407-49cc-b7b6-3e50e75bfac1","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.407Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"temporal-sys-tq-scanner-taskqueue-0","WorkerID":"1@f5f2d9e27d78@","logging-call-at":"scanner.go:239"}
{"level":"info","ts":"2025-09-30T09:29:24.408Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-tq-scanner-taskqueue-0/1","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.408Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-tq-scanner-taskqueue-0/3","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.408Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-tq-scanner-taskqueue-0","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.409Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-tq-scanner-taskqueue-0/3","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.410Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"f5f2d9e27d78:dd228140-7d8f-48a8-b012-8c5a42bcda78","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.410Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-history-scanner-taskqueue-0","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.411Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-tq-scanner-taskqueue-0/2","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.412Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-tq-scanner-taskqueue-0","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.412Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"temporal-sys-history-scanner-taskqueue-0","WorkerID":"1@f5f2d9e27d78@","logging-call-at":"scanner.go:239"}
{"level":"info","ts":"2025-09-30T09:29:24.412Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-tq-scanner-taskqueue-0/2","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.413Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-history-scanner-taskqueue-0/3","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.413Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-history-scanner-taskqueue-0/3","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.414Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-history-scanner-taskqueue-0/1","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.414Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-tq-scanner-taskqueue-0/1","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.415Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-history-scanner-taskqueue-0/2","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.415Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-history-scanner-taskqueue-0/2","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.415Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-history-scanner-taskqueue-0","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.417Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-history-scanner-taskqueue-0/1","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.433Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"f5f2d9e27d78:10d183af-caec-4fec-8e9a-48ddfc71a59d","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.433Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-processor-parent-close-policy/2","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.436Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-processor-parent-close-policy/1","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.477Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"temporal-sys-processor-parent-close-policy","WorkerID":"1@f5f2d9e27d78@","logging-call-at":"processor.go:98"}
{"level":"info","ts":"2025-09-30T09:29:24.477Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-processor-parent-close-policy/3","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.479Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-processor-parent-close-policy/1","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.480Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-processor-parent-close-policy/2","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.480Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-processor-parent-close-policy","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.480Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-processor-parent-close-policy","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.481Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"f5f2d9e27d78:21fa2d53-56c8-4a98-83e8-1bbabb8c7151","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.481Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-batcher-taskqueue","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.483Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-batcher-taskqueue/3","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.493Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"temporal-sys-batcher-taskqueue","WorkerID":"1@f5f2d9e27d78@","logging-call-at":"batcher.go:96"}
{"level":"info","ts":"2025-09-30T09:29:24.494Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-batcher-taskqueue/2","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.494Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-batcher-taskqueue/3","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.510Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"f5f2d9e27d78:29edf619-3da6-41cd-81ea-4ebf7954c349","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.512Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"default-worker-tq","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.539Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"default-worker-tq","WorkerID":"1@f5f2d9e27d78@","logging-call-at":"worker.go:113"}
{"level":"info","ts":"2025-09-30T09:29:24.540Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"default-worker-tq","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.541Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/default-worker-tq/1","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.572Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"temporal-sys-add-search-attributes-activity-tq","WorkerID":"1@f5f2d9e27d78@","logging-call-at":"worker.go:113"}
{"level":"info","ts":"2025-09-30T09:29:24.572Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-add-search-attributes-activity-tq","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.573Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-add-search-attributes-activity-tq","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.587Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"temporal-sys-migration-activity-tq","WorkerID":"1@f5f2d9e27d78@","logging-call-at":"worker.go:113"}
{"level":"info","ts":"2025-09-30T09:29:24.588Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-migration-activity-tq","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.588Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-migration-activity-tq","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.592Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"temporal-sys-delete-namespace-activity-tq","WorkerID":"1@f5f2d9e27d78@","logging-call-at":"worker.go:113"}
{"level":"info","ts":"2025-09-30T09:29:24.592Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-delete-namespace-activity-tq","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.616Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"temporal-sys-dlq-activity-tq","WorkerID":"1@f5f2d9e27d78@","logging-call-at":"worker.go:113"}
{"level":"info","ts":"2025-09-30T09:29:24.616Z","msg":"none","component":"worker-manager","lifecycle":"Started","logging-call-at":"worker.go:118"}
{"level":"info","ts":"2025-09-30T09:29:24.616Z","msg":"none","component":"perns-worker-manager","lifecycle":"Starting","logging-call-at":"pernamespaceworker.go:173"}
{"level":"info","ts":"2025-09-30T09:29:24.616Z","msg":"none","component":"perns-worker-manager","lifecycle":"Started","logging-call-at":"pernamespaceworker.go:184"}
{"level":"info","ts":"2025-09-30T09:29:24.616Z","msg":"worker service started","service":"worker","component":"worker","address":"172.19.0.2:7239","logging-call-at":"service.go:375"}
{"level":"info","ts":"2025-09-30T09:29:24.616Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-dlq-activity-tq","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.617Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-dlq-activity-tq","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.618Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"/_sys/temporal-sys-dlq-activity-tq/1","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.620Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"f5f2d9e27d78:5a0a4a94-1b39-45a7-88ce-fa97c170b816","wf-task-queue-type":"Workflow","wf-namespace":"default","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.621Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-per-ns-tq","wf-task-queue-type":"Workflow","wf-namespace":"default","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.621Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"f5f2d9e27d78:5d70315b-3e4f-46fb-8fe6-3aa4839fce1f","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.621Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-per-ns-tq","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.623Z","msg":"Started Worker","service":"worker","Namespace":"default","TaskQueue":"temporal-sys-per-ns-tq","WorkerID":"server-worker@1@f5f2d9e27d78@default","logging-call-at":"pernamespaceworker.go:512"}
{"level":"info","ts":"2025-09-30T09:29:24.624Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-per-ns-tq","wf-task-queue-type":"Activity","wf-namespace":"default","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.624Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-batcher-taskqueue","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.624Z","msg":"Started Worker","service":"worker","Namespace":"temporal-system","TaskQueue":"temporal-sys-per-ns-tq","WorkerID":"server-worker@1@f5f2d9e27d78@temporal-system","logging-call-at":"pernamespaceworker.go:512"}
{"level":"info","ts":"2025-09-30T09:29:24.625Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-per-ns-tq","wf-task-queue-type":"Activity","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
{"level":"info","ts":"2025-09-30T09:29:24.652Z","msg":"none","service":"matching","component":"matching-engine","wf-task-queue-name":"temporal-sys-delete-namespace-activity-tq","wf-task-queue-type":"Workflow","wf-namespace":"temporal-system","lifecycle":"Started","logging-call-at":"task_queue_manager.go:330"}
Temporal server started.
Registering default namespace: default.
  NamespaceInfo.Name                    default                                           
  NamespaceInfo.Id                      45e22265-9c30-4805-8200-99a80475028f              
  NamespaceInfo.Description             Default namespace for Temporal                    
                                        Server.                                           
  NamespaceInfo.OwnerEmail                                                                
  NamespaceInfo.State                   Registered                                        
  NamespaceInfo.Data                    map[]                                             
  Config.WorkflowExecutionRetentionTtl  24h0m0s                                           
  ReplicationConfig.ActiveClusterName   active                                            
  ReplicationConfig.Clusters            [&ClusterReplicationConfig{ClusterName:active,}]  
  Config.HistoryArchivalState           Disabled                                          
  Config.VisibilityArchivalState        Disabled                                          
  IsGlobalNamespace                     false                                             
  FailoverVersion                                                                      0  
  FailoverHistory                       []                                                

Default namespace default already registered.
  ?[95m           Name           ?[0m  ?[95m   Type    ?[0m  
  BatcherNamespace            Keyword      
  BatcherUser                 Keyword      
  BinaryChecksums             KeywordList  
  BuildIds                    KeywordList  
  CloseTime                   Datetime     
  ExecutionDuration           Int          
  ExecutionStatus             Keyword      
  ExecutionTime               Datetime     
  HistoryLength               Int          
  HistorySizeBytes            Int          
  ParentRunId                 Keyword      
  ParentWorkflowId            Keyword      
  RunId                       Keyword      
  StartTime                   Datetime     
  StateTransitionCount        Int          
  TaskQueue                   Keyword      
  TemporalChangeVersion       KeywordList  
  TemporalNamespaceDivision   Keyword      
  TemporalSchedulePaused      Bool         
  TemporalScheduledById       Keyword      
  TemporalScheduledStartTime  Datetime     
  WorkflowId                  Keyword      
  WorkflowType                Keyword      
Namespace cache refreshed.
Adding Custom*Field search attributes.
{"level":"error","ts":"2025-09-30T09:29:25.198Z","msg":"Cannot add search attributes in standard visibility.","service":"frontend","pluginName":"mysql","logging-call-at":"operator_handler.go:223","stacktrace":"go.temporal.io/server/common/log.(*zapLogger).Error\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/log/zap_logger.go:156\ngo.temporal.io/server/service/frontend.(*OperatorHandlerImpl).addSearchAttributesInternal\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/service/frontend/operator_handler.go:223\ngo.temporal.io/server/service/frontend.(*OperatorHandlerImpl).AddSearchAttributes\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/service/frontend/operator_handler.go:203\ngo.temporal.io/api/operatorservice/v1._OperatorService_AddSearchAttributes_Handler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/go.temporal.io/api@v1.29.2/operatorservice/v1/service_grpc.pb.go:317\ngo.temporal.io/server/common/rpc/interceptor.(*RetryableInterceptor).Intercept.func1\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/retry.go:63\ngo.temporal.io/server/common/backoff.ThrottleRetryContext\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/backoff/retry.go:143\ngo.temporal.io/server/common/rpc/interceptor.(*RetryableInterceptor).Intercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/retry.go:67\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc/interceptor.(*CallerInfoInterceptor).Intercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/caller_info.go:80\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc/interceptor.(*SDKVersionInterceptor).Intercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/sdk_version.go:69\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc/interceptor.(*RateLimitInterceptor).Intercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/rate_limit.go:88\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc/interceptor.(*NamespaceRateLimitInterceptor).Intercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/namespace_rate_limit.go:93\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc/interceptor.(*ConcurrentRequestLimitInterceptor).Intercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/concurrent_request_limit.go:121\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc/interceptor.(*NamespaceValidatorInterceptor).StateValidationIntercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/namespace_validator.go:196\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc/interceptor.(*TelemetryInterceptor).UnaryIntercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/telemetry.go:165\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/service/frontend.(*RedirectionInterceptor).Intercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/service/frontend/redirection_interceptor.go:184\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/authorization.(*interceptor).Interceptor\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/authorization/interceptor.go:162\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/service/frontend.GrpcServerOptionsProvider.NewServerMetricsContextInjectorInterceptor.func1\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/metrics/grpc.go:66\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc.UnaryServerInterceptor.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc@v0.51.0/interceptor.go:315\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc/interceptor.(*NamespaceLogInterceptor).Intercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/namespace_logger.go:84\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc/interceptor.(*NamespaceValidatorInterceptor).NamespaceValidateIntercept\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/interceptor/namespace_validator.go:113\ngoogle.golang.org/grpc.getChainUnaryHandler.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1196\ngo.temporal.io/server/common/rpc.ServiceErrorInterceptor\n\t/mnt/d/work_msy/project/temporal/docker-builds/temporal/common/rpc/grpc.go:145\ngoogle.golang.org/grpc.NewServer.chainUnaryServerInterceptors.chainUnaryInterceptors.func1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1187\ngo.temporal.io/api/operatorservice/v1._OperatorService_AddSearchAttributes_Handler\n\t/mnt/d/work_msy/project/go/pkg/mod/go.temporal.io/api@v1.29.2/operatorservice/v1/service_grpc.pb.go:319\ngoogle.golang.org/grpc.(*Server).processUnaryRPC\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1379\ngoogle.golang.org/grpc.(*Server).handleStream\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1790\ngoogle.golang.org/grpc.(*Server).serveStreams.func2.1\n\t/mnt/d/work_msy/project/go/pkg/mod/google.golang.org/grpc@v1.64.0/server.go:1029"}
Search attributes have been added
```

#### **3. 启动后台**

打开页面 http://localhost:8080/namespaces/default/workflows，如下，

![Desktop View](/assets/img/20250930/temporal_server_run.png){: width="500" height="300" }
_temporal 首页效果图_

#### **4. 写测试用例，或者加断点Debug代码。**

单元测试和调试步骤可以参考文档，[Testing - Java SDK](https://docs.temporal.io/develop/java/testing-suite)、[Debugging - Java SDK](https://docs.temporal.io/develop/java/debugging)

如果要加断点，需要注意环境变量加上`TEMPORAL_DEBUG=true`， 否则会报`PotentialDeadlockException`。

## **外围——Temporal SDK**
`Temporal`支持多种语言的`SDK`，比如`go`、`java`、`python`、`ruby`、`php`、`typescript`等，本文主要以`go`和`java` `sdk`为例，来看下使用姿势。

![Desktop View](/assets/img/20250930/temporal_sdk.png){: width="500" height="300" }
_temporal sdk_

### **Temporal Java SDK**

[Temporal Java SDK Github](https://github.com/temporalio/sdk-java)

|tag|commit_id|remark|
| :----------------| :----------------------------- |:----------------------------- |
|[v1.31.0](https://github.com/temporalio/sdk-java/releases/tag/v1.31.0)|94007cccaa6bee1ada3d47df66f31ddd3d3f498a|Aug 22, 2025 1:23 AM GMT +8|
|~~[v1.3.1](https://github.com/temporalio/sdk-java/releases/tag/v1.3.1)~~|~~01a1b9fe189fc373ded9692c47c108d0db5ad54c~~|~~Sep 14, 2021 1:40 AM GMT +8~~|

### **Java SDK与SpringBoot/Dubbo集成**

**问题**

服务使用的`Dubbo`框架依赖`guava 19.0`，`temporal`用的是`guava 30+`（比如`30.1.1-jre`），如果把`temporal`集成到`dubbo`框架中，会产生`guava`包冲突。

**解决方案**

使用`maven-shade-plugin`解决依赖版本冲突，对`temporal`依赖的`guava 30+`，做全面的`shade`，`com.google.common` -> `shaded.v30_1_1_jre.com.google.common`，使得`temporal`只使用`shaded guava`，而`dubbo`依旧使用`guava 19.0`。

**shade后的sdk列表**

|相关的依赖包|
|shaded-guava-0.0.1-SNAPSHOT|
|shaded-grpc-core-0.0.1-SNAPSHOT|
|shaded-grpc-netty-0.0.1-SNAPSHOT|
|shaded-grpc-netty-shaded-0.0.1-SNAPSHOT|
|shaded-grpc-protobuf-0.0.1-SNAPSHOT|
|shaded-grpc-services-0.0.1-SNAPSHOT|
|shaded-grpc-stub-0.0.1-SNAPSHOT|
|temporal-sdk-0.0.1-SNAPSHOT|
|temporal-serviceclient-0.0.1-SNAPSHOT|
|temporal-testing-0.0.1-SNAPSHOT|
|temporal-test-server-0.0.1-SNAPSHOT|

### **Temporal Go SDK**
#### **Go Case**

1. 部署好`TS`后，找`helloworld sample`测试，看下效果

    ```markdown
    https://github.com/temporalio/samples-go/tree/main/helloworld
    ```

2. 启动`worker`（用户实际业务的执行（类比消费者，实际执行））

    ```terminal
    # go run helloworld/worker/main.go
    2025/08/04 17:24:39 INFO  No logger configured for temporal client. Created default one.
    2025/08/04 17:24:39 INFO  Started Worker Namespace default TaskQueue hello-world WorkerID 4861@shouyuanman-nb1@

    2025/08/04 17:28:52 INFO  HelloWorld workflow started Namespace default TaskQueue hello-world WorkerID 4861@shouyuanman-nb1@ WorkflowType Workflow WorkflowID hello_world_workflowID RunID 01987469-9325-7800-9822-7dab474b00c3 Attempt 1 name Temporal
    2025/08/04 17:28:52 DEBUG ExecuteActivity Namespace default TaskQueue hello-world WorkerID 4861@shouyuanman-nb1@ WorkflowType Workflow WorkflowID hello_world_workflowID RunID 01987469-9325-7800-9822-7dab474b00c3 Attempt 1 ActivityID 5 ActivityType Activity
    2025/08/04 17:28:52 INFO  Activity Namespace default TaskQueue hello-world WorkerID 4861@shouyuanman-nb1@ ActivityID 5 ActivityType Activity Attempt 1 WorkflowType Workflow WorkflowID hello_world_workflowID RunID 01987469-9325-7800-9822-7dab474b00c3 name Temporal
    2025/08/04 17:28:52 INFO  HelloWorld workflow completed. Namespace default TaskQueue hello-world WorkerID 4861@shouyuanman-nb1@ WorkflowType Workflow WorkflowID hello_world_workflowID RunID 01987469-9325-7800-9822-7dab474b00c3 Attempt 1 result Hello Temporal!
    ```

3. 启动`starter`（用于服务的触发（类比生产者，业务触发入口）），触发开启这个`Case`

    ```terminal
    # go run helloworld/starter/main.go
    2025/08/04 17:28:52 INFO  No logger configured for temporal client. Created default one.
    2025/08/04 17:28:52 Started workflow WorkflowID hello_world_workflowID RunID 01987469-9325-7800-9822-7dab474b00c3
    2025/08/04 17:28:52 Workflow result: Hello Temporal!
    ```

4. 执行完成

    ![Desktop View](/assets/img/20250930/temporal_go_sdk_result.png){: width="500" height="300" }
    _temporal go sample 运行效果图_

5. 查看执行历史

    [view history link](http://localhost:8080/namespaces/default/workflows/hello_world_workflowID/01987469-9325-7800-9822-7dab474b00c3/history)

    ![Desktop View](/assets/img/20250930/temporal_go_sdk_history_all.png){: width="500" height="300" }
    _sample all history_
    ![Desktop View](/assets/img/20250930/temporal_go_sdk_history_compact.png){: width="500" height="300" }
    _sample compact history_

6. 查看执行时的`workers`——`hello_world_workflowID`

    [view workers link](http://localhost:8080/namespaces/default/workflows/hello_world_workflowID/01987469-9325-7800-9822-7dab474b00c3/workers)

    ![Desktop View](/assets/img/20250930/temporal_go_sdk_worker.png){: width="500" height="300" }
    _sample worker_

7. 查看`Task Queue`——`hello-world`

    [view task queue link](http://localhost:8080/namespaces/default/task-queues/hello-world)

    ![Desktop View](/assets/img/20250930/temporal_go_sdk_task_queue.png){: width="500" height="300" }
    _sample task queue_

## **欣赏——Temporal亮点和局限性**
### **亮点**
1. 客户端提供了完善的多语言`SDK`和样例、单元测试：`Go/Java/TS/Python/PHP/C#/Ruby`，工作流的编程方式非常友好；
2. 背后是从`Uber`出来创业的商业公司，高性能、稳定性好，可大规模用于产线；
3. 自带`WebUI`对`Workflow`和`Activity`进行查询很方便；
4. 对应用层的灵活性非常高，应用场景广泛；
5. 在中心化状态机+事件溯源的模式下开发运行的代码非常健壮，`Bug`率低；
6. 对于长时间跨度的业务处理很方便，无需引入一套分布式`CronJob/Timer`方案，一行代码可以实现分布式`Timer/CronJob`。

### **局限性**
1. 偏底层，没有提供`Workflow as DSL/Yaml`，需要自己实现`DSL`和对应的可视化`UI`；
2. `SDK`的多语言和多框架的支持还没有特别完善，比如官方对`SpringBoot`自动配置的支持在开发中，更多编程语言的`SDK`还在陆续发布；
3. `Temporal`的`Cron Job`目前支持到分钟级别，不能设置秒级的`Cron Job`。

## **扩展——DSL + UI**
`Temporal`偏底层，没有提供`Workflow as DSL/Yaml`，需要自己实现`DSL`和对应的可视化`UI`。
### **Temporal DSL**
#### **swtemporal**
`Temporal Technologies`公司开源了工作流编排引擎`Temporal`。`Tihomir Surdilovic`作为`CNCF Serverless Workflow`社区的领导人（`CNCF Serverless Workflow`社区致力于无服务器计算领域的标准化工作），开源了`swtemporal`项目。该项目是`Serverless Workflow`（`SW`）与`Temporal`技术的集成`demo`，通过标准化函数执行和可视化编排能力，为分布式系统开发提供实验性演示环境。
- [swtemporal Github](https://github.com/tsurdilo/swtemporal)
- 开发语言：`Java`
- 依赖`SDK`：`Temporal Java SDK`

##### **关于 swtemporal**
`swtemporal`的开源定位是一个演示`demo`，它使用`Temporal Java SDK`定义了一个动态工作流，该工作流能够执行`Serverless Workflow DSL`定义的指令。为此，它使用`Serverless Workflow Java SDK`在解释之前验证和解析`Json DSL`（也可使用`yaml`），将其转换为对象模型。
1. 展示了`Serverless Workflow`项目的体验，可以在其中轻松构建在线编辑器、可视化工具、基于表单的工作流数据输入，并能启动工作流执行并等待结果。
2. `Serverless Workflow`规范支持基于标准的函数执行，即它支持如`OpenApi`、`AsyncApi`、`GraphQL`等标准。
3. 对于集成，`Serverless Workflow`（`SW`）函数与`Temporal Activities`的调用相匹配。`Temporal Activities`是可以自己添加代码的函数，`demo`并没有实现第三方服务的实际调用，代码中的`activities`仅仅做了模拟，实际应用中可以使用例如`OpenApi`服务定义的生成器自动生成代码。

`swtemporal`方案体现了`CNCF Serverless Workflow`社区推动的开放标准与`Temporal`技术栈的互补性，为分布式系统开发提供了新的工具链整合思路。

>注：演示`demo`没有支持`Serverless Workflow`规范的全部内容，虽然目标是尽快实现全面支持。如果要应用在实际生产环境项目中，需要针对特定复杂需求进行二次开发。
{: .prompt-tip }

#### **DSL示例**

```json
{
	"id": "customerapplication",
	"name": "Customer Application Workflow",
	"version": "1.0",
	"specVersion": "0.7",
	"timeouts": {
		"workflowExecTimeout": {
			"duration": "PT1M"
		},
		"actionExecTimeout": "PT10S"
	},
	"retries": [{
		"name": "WorkflowRetries",
		"delay": "PT3S",
		"maxAttempts": 10
	}],
	"start": "NewCustomerApplication",
	"states": [{
			"name": "NewCustomerApplication",
			"type": "event",
			"onEvents": [{
				"eventRefs": [
					"NewApplicationEvent"
				],
				"actionMode": "parallel",
				"actions": [{
						"name": "Invoke Check Customer Info Function",
						"functionRef": "CheckCustomerInfo"
					},
					{
						"name": "Invoke Update Application Info Function",
						"functionRef": "UpdateApplicationInfo"
					}
				]
			}],
			"transition": "GetJobStatus"
		},
		{
			"name": "GetJobStatus",
			"type": "operation",
			"actionMode": "sequential",
			"actions": [{
				"functionRef": {
					"refName": "checkJobStatus",
					"arguments": {
						"name": "${ .jobuid }"
					}
				},
				"name": "Invoke checkJobStatus Function",
				"actionDataFilter": {
					"results": "${ .jobstatus }"
				}
			}],
			"stateDataFilter": {
				"output": "${ .jobstatus }"
			},
			"transition": "MakeApplicationDecision"
		},
		{
			"name": "MakeApplicationDecision",
			"type": "switch",
			"dataConditions": [{
					"condition": "$..[?(@.jobstatus == \"SUCCEEDED\")]",
					"transition": "ApproveApplication"
				},
				{
					"condition": "$..[?(@.jobstatus == \"FAILED\")]",
					"transition": "RejectApplication"
				}
			],
			"defaultCondition": {
				"transition": "RejectApplication"
			}
		},
		{
			"name": "ApproveApplication",
			"type": "operation",
			"actions": [{
				"name": "Invoke Approve Application Function",
				"functionRef": "ApproveApplication",
				"sleep": {
					"before": "PT1S"
				}
			}],
			"end": true
		},
		{
			"name": "RejectApplication",
			"type": "operation",
			"actions": [{
				"name": "Invoke Reject Application Function",
				"functionRef": "RejectApplication",
				"sleep": {
					"before": "PT1S"
				}
			}],
			"end": true
		}
	],
	"functions": [{
			"name": "CheckCustomerInfo",
			"type": "rest"
		},
		{
			"name": "UpdateApplicationInfo",
			"type": "rest"
		},
		{
			"name": "ApproveApplication",
			"type": "rest"
		},
		{
			"name": "RejectApplication",
			"type": "rest"
		},
		{
			"name": "checkJobStatus",
			"type": "rest"
		}
	],
	"events": [{
		"name": "NewApplicationEvent",
		"type": "com.fasterxml.jackson.databind.JsonNode",
		"source": "applicationsSource"
	}]
}
```
{: file='DSL示例' .nolineno }

#### **看看效果**

##### **启动 Temporal Server**
##### **部署swtemporal，RunWorkFlow**

1. 访问`localhost:8081`，打开`Temporal DSL`可视化的界面

    ![Desktop View](/assets/img/20250930/swtemporal_workflow_define.png){: width="500" height="300" }
    ![Desktop View](/assets/img/20250930/swtemporal_workflow_run.png){: width="500" height="300" }

2. 查看`Temporal Server`中对应的结果，`Temporalio UI 2.26.2`版本的效果图如下，

    ![Desktop View](/assets/img/20250930/swtemporal_workflow_run_result_01.png){: width="500" height="300" }
    ![Desktop View](/assets/img/20250930/swtemporal_workflow_run_result_02.png){: width="500" height="300" }
    ![Desktop View](/assets/img/20250930/swtemporal_workflow_run_result_03.png){: width="500" height="300" }
    ![Desktop View](/assets/img/20250930/swtemporal_workflow_run_result_04.png){: width="500" height="300" }

### **可视化流程设计——用 ReactFlow UI 生成 Temporal DSL**
前端可视化主流方案包括`ReactFlow`、`VueFlow`等，`UI`这里可以选型`ReactFlow`通过拖拉拽控件的方式生成上述`DSL`。效果图如下，

![Desktop View](/assets/img/20250930/reactflow_example.png){: width="500" height="300" }

## **升华——一站式应用示例**
至此，通过选型`ReactFlow UI + DSL + Temporal Service`，把这套架构融合到自己的系统中，就形成了一站式分布式工作流编排解决方案。

![Desktop View](/assets/img/20250930/temporal_task_arch.png){: width="500" height="300" }

## **总结**
1. 控制模式‌
- 采用集中式编排（`Orchestration`），通过中央控制器协调服务执行顺序，确保流程强一致性。
2. 核心组件‌
- `Workflow`：定义业务流程逻辑，开发者需编写完整代码实现执行顺序。
- `Activity`：封装具体任务单元（如`API`调用、数据库操作），由`Workflow`调用。
3. 状态管理‌
- 基于事件溯源（`Event Sourcing`）持久化工作流状态，自动实现故障恢复与任务重试。
4. 复杂度与适用场景‌
- 优势‌：适用于严格顺序执行的长时流程（如订单处理、金融结算）及需高可靠性的分布式任务。
- 挑战‌：需处理分布式协调问题，开发成本较高，且需完整实现`Workflow`代码。
5. 扩展性与工具支持‌
- 支持子工作流扩展复杂流程；
- 提供可视化`UI`专注工作流监控，但缺乏低代码流程设计能力，需额外设计结合`UI`（比如`ReactFlow`）和自定义`DSL`来实现低代码流程设计。
6. 典型定位‌
- 作为独立平台，适合对一致性要求严苛的场景，但需权衡开发效率与复杂度。

## **参考资料**

### **Temporal**
[Temporal Github](https://github.com/temporalio)

[Temporal 官方文档](https://docs.temporal.io/)

[创始人Maxim讲解的Temporal详细原理](https://www.youtube.com/watch?v=t524U9CixZ0)

[Designing a Workflow engine from first principles](https://temporal.io/blog/workflow-engine-principles)

[7分钟快速入门案例](https://www.youtube.com/watch?v=2HjnQlnA5eY&ab_channel=Temporal)

[3种使用场景详细介绍](https://www.youtube.com/watch?v=eMf1fk9RmhY&ab_channel=Temporal)

[工作流引擎Temporal学习笔记](https://code2life.top/blog/0070-temporal-notes)

### **serverlessworkflow**
[serverlessworkflow工作流](https://github.com/serverlessworkflow)

[swtemporal](https://github.com/tsurdilo/swtemporal)

[Mapping DSL to Temporal](https://serverlessworkflow.io/)

[Temporal at Netflix, Serverless Workflow DSL, and TypeScript SDK Demo](https://www.youtube.com/watch?v=JQ6FRTnQWFI)

### **项目链接**
[temporalio/temporal](https://github.com/temporalio/temporal)

[temporalio/docker-builds](https://github.com/temporalio/docker-builds.git)

[temporalio/docker-compose](https://github.com/temporalio/docker-compose)

[temporalio/tctl](https://github.com/temporalio/tctl)

[temporalio/cli](https://github.com/temporalio/cli)

[temporalio/temporalite-archived](https://github.com/temporalio/temporalite-archived)

[temporalio/ui](https://github.com/temporalio/ui)

[temporalio/ui-server](https://github.com/temporalio/ui-server)

[samples-go](https://github.com/temporalio/samples-go.git)

[samples-java](https://github.com/temporalio/samples-java.git)

[sdk-go](https://github.com/temporalio/sdk-go.git)

[sdk-java](https://github.com/temporalio/sdk-java.git)

[money-transfer-project-template-go](https://github.com/temporalio/money-transfer-project-template-go.git)

[money-transfer-project-java](https://github.com/temporalio/money-transfer-project-java.git)

[hello-world-project-template-java](https://github.com/temporalio/hello-world-project-template-java.git)

[hello-world-project-template-go](https://github.com/temporalio/hello-world-project-template-go.git)

[spring-boot-demo](https://github.com/temporalio/spring-boot-demo.git)

[demo-go](https://github.com/temporalio/demo-go.git)

[edu-101-java-code](https://github.com/temporalio/edu-101-java-code.git)

[edu-102-java-code](https://github.com/temporalio/edu-102-java-code.git)

[edu-101-go-code](https://github.com/temporalio/edu-101-go-code.git)

[serverlessworkflow/specification](https://github.com/serverlessworkflow/specification.git)

[serverlessworkflow/sdk-go](https://github.com/serverlessworkflow/sdk-go)

[serverlessworkflow/sdk-java](https://github.com/serverlessworkflow/sdk-java)

[serverlessworkflow/sdk-typescript](https://github.com/serverlessworkflow/sdk-typescript)

[serverlessworkflow/synapse](https://github.com/serverlessworkflow/synapse)

[serverlessworkflow/workflow-diagram-service](https://github.com/serverlessworkflow/workflow-diagram-service)
