---
title: APISIX——南北向流量调度那些事儿
date: 2026-07-25 19:00:00 +0800
categories: [微服务, 网关]
tags: [后端, 微服务, 网关, APISIX]
music-id: 2111831690
---

## **引言**

`Apache APISIX`是一个基于`OpenResty`（`Nginx + LuaJIT`）的云原生`API`网关。它把动态路由、限流熔断、鉴权、可观测性等能力都做成了插件，配置存储可以选择依赖`etcd`或者`yaml`，支持热更新、无需`reload`。

>参考`APISIX`的官方博客，聊聊为什么`Apache APISIX`选择`NGINX+LuaJIT`技术栈？
{: .prompt-tip }

从`Apache APISIX`的`Nginx + LuaJIT`的技术栈出发，`Nginx + LuaJIT`的技术栈给我们带来的，不仅仅是高性能。

思考一个问题，既然`APISIX`是基于`Nginx`开源版本，而`Nginx`并不支持动态配置，为什么`Apache APISIX`声称自己可以实现动态配置？是不是改了点东西？

是的，`APISIX`确实有在维护自己的`Nginx`发行版，不过`Apache APISIX`的大部分功能在官方的`Nginx`上就能使用。之所以能做到动态配置，全靠把配置放到`Lua`代码里面来实现。

以路由系统为例，`Nginx`的路由需要在配置文件里面进行配置，每次更改路由，都需要`reload`之后才能生效。这是因为`Nginx`的路由分发只支持静态配置，不能动态增减路由。

为了实现路由动态配置，`Apache APISIX`做了两件事，
- 在`Nginx`配置文件里面配置单个`server`，这个`server`里面只有一个`location`。把这个`location`作为主入口，这样所有的请求都会走到这个地方上来。
- 用`Lua`完成路由分发的工作。`Apache APISIX`的路由分发模块，支持在运行时增减路由，这样就能动态配置路由了。

>你可能会问，在`Lua`里面做路由分发，会比`Nginx`的实现慢吗？
{: .prompt-tip }

凡是对性能要求比较高的，`APISIX`会把核心代码用`C`改写。路由分发模块就是这么干的。路由分发模块在匹配路由时，会采用一个前缀树来匹配。而这个前缀树是用`C`实现的。感兴趣的读者可以看下代码：[lua-resty-radixtree](https://github.com/api7/lua-resty-radixtree/)。

完成了`C`层面上的前缀树匹配，接下来就该`Lua`发挥灵活性的时刻了。对于匹配同一前缀的各个路由，`APISIX`支持通过许多别的方式来进行下一级的匹配，其中就包含通过一个特定的表达式来匹配。尽管硬着头皮，也能在`C`层面上接入一个表达式引擎，但是纯`C`实现做不了非常灵活地自定义表达式里面的变量。

举个例子，下面是`Apache APISIX`用来匹配`GraphQL`请求的`route`配置，

```json
{
        "methods": ["POST"],
        "upstream": {
            "nodes": {
                "127.0.0.1:1980": 1
            },
            "type": "roundrobin"
        },
        "uri": "/hello",
        "vars": [["graphql_name", "==", "repo"]]
}
```

它会匹配这样的`GraphQL`请求，

```
query repo {
    owner {
        name
    }
}
```

这里的`graphql_name`并非`Nginx`内置变量，而是通过`Lua`代码定义的。`Apache APISIX`一共定义了三个`GraphQL`相关的变量，连同解析`GraphQL body`在内不过`62`行`Lua`代码。如果要通过`Nginx C`模块来定义变量，`62`行可能只不过是把相关方法的样板代码搭建起来，都还没有到真正的解析`GraphQL`的逻辑呢。

>采用`Lua`代码来做路由还有一个好处：它降低了二次开发的门槛。
{: .prompt-tip }

如果在路由过程中需要有特殊的逻辑，用户可以实现成自定义的变量和运算符，比如通过`IP`库匹配到的地理位置来决定采用哪条路由。用户只需要写一些`Lua`代码，这要比修改`Nginx C module`的难度小多了。

在`Apache APISIX`里面，不仅仅路由是动态的，`TLS`服务端证书和上游节点配置都是动态的，而且无需修改`Nginx`，功能可以跑在官方的`Nginx + LuaJIT`技术栈上。当然通过修改`Nginx`，还实现了更多的高级功能，比如动态的`gzip`配置和动态的客户端请求大小限制。后续`APISIX`也会推行自己的`Nginx`发行版，这样开源用户也能轻松用上这些高级功能。

>一句话理解它的定位：**它本质上是一个增强版的`OpenResty`**。`APISIX`没有独立的常驻后台进程，流量由`Nginx`的`master/worker`多进程模型处理，所有网关能力都是挂在请求生命周期上的`Lua`插件。
{: .prompt-tip }

`Nginx`请求生命周期的`11`个阶段，如下，

![Desktop View](/assets/images/20260724/nginx_11_phase.jpg){: width="800" height="400"}
_Nginx请求生命周期_

## **APISIX的数据面和控制面**

![Desktop View](/assets/images/20260724/apisix_function_arch.png){: width="800" height="400"}
_数据面和控制面_

如上图所示，左右分别是`APISIX`的数据面（`Data Plane`）和控制面（`Control Plane`），
- 数据面：以`NGINX`的网络库为基础（未使用`NGINX`的路由匹配、静态配置和`C`模块），使用`Lua`和`NGINX`动态控制请求流量；
- 控制面：使用`etcd`来存储和同步网关的配置数据，管理员通过`Admin API`或者`Dashboard`可以在毫秒级别内通知到所有数据面节点。

## **APISIX有什么？**

看下`APISIX`控制面配置的元数据到数据面的数据流向，

![Desktop View](/assets/images/20260724/apisix_deploy_arch.png){: width="800" height="400"}
_元数据流向图_

在数据面，微观上看进程列表（进入`APISIX`容器执行`ps`），一般情况下，只会有两类进程，分别是`Master`、`Worker`，而特权进程可选，不是网关必备主流程，这里不对特权进程做发散，感兴趣的话可以单独看下附录里边特权进程的简单介绍。

```markdown
Master（Nginx原生）
├─ Worker #1（业务流量、处理请求、ngx.timer）
├─ Worker #2
└─ Privileged Agent 特权进程（OpenResty扩展）
```

从进程的视角观察数据面，流程如下，

![Desktop View](/assets/images/20260724/apisix_ps.png){: width="800" height="400"}
_从进程的视角观察控制面_

```markdown
Nginx Master（管理进程）
├─ Nginx Worker #1
└─ Nginx Worker #2
     ↓
     请求进来 → 执行内嵌Lua代码（APISIX核心逻辑、插件、路由、限流、鉴权）
```

`Lua`代码不是独立程序，是寄生在`Nginx Worker`请求回调钩子中的脚本。

>所有网关能力挂在请求生命周期上，是什么意思？
{: .prompt-tip }

`Nginx`在请求生命周期的`11`个阶段，提供一系列请求阶段钩子（`rewrite`、`access`、`filter`、`log`等），
1. 客户端发起`HTTP`请求到达`Worker`
2. `Worker`按顺序触发各个阶段
3. `APISIX`在这些阶段插入`Lua`逻辑，
   - `access`阶段：路由匹配、`JWT`鉴权、`CORS`、限流插件
   - `upstream`阶段：负载均衡、改写上游地址
   - `log`阶段：访问日志、监控上报

只要没有请求进来，`Lua`插件代码不会主动运行。

>那定时任务、`etcd watch`配置监听是谁在跑？<br/>
很多人会疑惑，`APISIX`持续监听`etcd`变更、健康检查、定时清理缓存，这些后台任务难道不需要进程？
{: .prompt-tip }

关键点来了，`OpenResty`提供`ngx.timer`定时器，定时器依然运行在现有`Nginx Worker`内部，不会新建独立进程。
- 定时器 = `Worker`空闲间隙执行的一段`Lua`任务
- 没有`fork`出新程序、没有独立后台进程
- `Worker`退出，所有定时器、`etcd`长连接全部跟着消失

形象比喻，
- `Nginx Worker`是一间办公室；
- `Lua`插件、`etcd`监听、定时任务，都是办公室里员工顺带完成的工作；
- 没有另外单独聘请一个专职后台人员（独立进程）。

反面例子帮助理解，
- 如果有独立常驻进程：会多出一条进程，脱离`Nginx`生命周期，可以单独启停。
- `APISIX`现状：关掉`Nginx`，整个网关直接消失；无法单独启停`apisix`业务模块。

底层载体永远是`Nginx`进程；网关只是运行在`Nginx`内部的`Lua`应用。对比`Nginx`原生，
- 原生`Nginx`只有`C`模块；`OpenResty`给它增加运行`Lua`的能力；
- `APISIX` = 一套极其庞大、封装好的`Lua`业务套件，跑在`OpenResty`之上。

> `Apache APISIX`本质是运行于`OpenResty`之上的`Lua`应用，不存在脱离`Nginx`的独立常驻后台进程。<br/>
网关所有路由、鉴权、限流等能力，依托`Nginx`请求生命周期钩子执行；<br/>
配置监听、健康检查等后台任务依靠Worker内`ngx.timer`实现，复用现有Worker资源，不产生额外独立进程；<br/>
整个网关生命周期完全依附`Nginx Master/Worker`进程模型。
{: .prompt-tip }

## **搭建APISIX测试环境**

为了适配特定的需求，在实际生产中往往需要对`APISIX`进行二次开发，这里搭建的`APISIX`集群仅用于开发测试，包含了`Consul`（单机）、`Etcd`、`APISIX`和两个后端`web`实例。

### **准备docker-compose**

![Desktop View](/assets/images/20260724/apisix_docker_compose.png){: width="800" height="400"}
_apisix docker compose示例_

### **config.yaml 关键配置**

`APISIX 3.x`的配置容易踩两个概念混淆的坑，先把它们区分清楚，
1. 网关配置存储（路由/上游/消费者）：由`config_center`决定，`yaml` 或 `etcd`；
2. 服务发现（这里选型用的`Consul` `HTTP`模式）：由`discovery`模块决定，只负责动态拉取后端节点，不存储网关配置。

#### **网关配置存储**

原生`APISIX`关于网关配置存储的日常用法有两种，
- `config_center: etcd`，路由持久化到`etcd`，配合内置`UI`才能保存生效；
- 若用`config_center: yaml`，即`DB-less`无`etcd`模式（也叫`yaml standalone`独立模式），改路由刷新即丢失。

这两条路线面向两种完全不同的部署场景，一个定位分布式集群网关，一个定位轻量单机/边缘极简部署。`Apache APISIX`在诞生之初，目标就是云原生集群网关，所以`etcd`是原生标准模式；而`DB-less`是后续补充出来的轻量化方案，解决不想部署`etcd`的痛点。
- `etcd`模式：依靠分布式`KV`实现多节点配置共享、`Admin API`动态管理、增量同步；
- `DB-less`模式：零外部依赖、部署简单。

原生的两条路线的痛点也很明显，
- `etcd`模式：网关强依赖`etcd`，增加运维复杂度，`etcd`不可用时网关无法启动；需要运维`etcd`集群；
- `DB-less`模式：关闭`Admin API`，无法使用`Dashboard`可视化管理，无法原生支持动态`API`管理；缺乏集群同步能力，只能手工改文件，多节点配置难以统一；配置刷新为全量重载。

官方的落地建议
1. 生产集群、多节点`APISIX`、需要动态调整路由，使用`APISIX Dashboard`可视化管理路由，选`etcd`（官方主推）；
2. 边缘设备、单机测试、离线环境、配置极少改动、不想运维`etcd`，选用`DB-less`，放弃`UI`动态保存路由的用法，全部配置在`yaml`文件维护；
3. 禁忌场景
   - 多节点`APISIX`集群不推荐直接使用原生`yaml`模式，扩容多个`APISIX`实例时，各个节点内存路由不互通，极易出现集群严重不一致，各实例路由不统一；
   - 拥有数千条路由并且经常动态更新，不推荐`yaml`模式，全量加载会持续消耗`CPU`。

>不要混淆网关配置`config.yaml`和业务路由`apisix.yaml`
- `conf/config.yaml`：`APISIX`自身启动参数（`etcd`地址、`worker`数量、`admin`密钥等）
- `apisix.yaml`：`DB-less`模式下存放路由、上游、插件规则
{: .prompt-tip }

聚焦下原生`DB-less`模式，

原生的`DB-less`模式正确使用方式：所有路由、上游提前写死在`apisix.yaml`，修改路由必须改动`yaml`文件，再执行`apisix reload`。不能依赖`UI`临时新增。

直观看起来，这种模式在运维范式上退化至类似`Nginx`：持久规则依赖静态文件，无法通过`UI/API`永久保存路由。但它不属于完全退回`Nginx`，相较于`Nginx`，它重载代价较低，依然保留`APISIX`丰富的插件体系、更友好的声明式配置、支持运行时临时调试路由，只是舍弃了`etcd`提供的分布式动态配置能力。详细比对参见附录`APISIX DB-less vs Nginx`。

>`DB-less`无`etcd`模式的启动逻辑<br/>
`APISIX`启动时，一次性读取本地`apisix.yaml`文件，加载路由到内存。<br/>
这里有两条硬性约束，
1. 启动后，在`Dashboard/Admin API`新增路由，仅保存在当前节点内存，不会写入任何持久化介质；
2. 执行`apisix reload`/容器重启，`APISIX`重新加载本地`yaml`，内存里`API`新增的路由全部清空，只会保留`yaml`内写死的规则。
```markdown
apisix.yaml = 唯一静态数据源
任何通过Admin API动态添加的配置 ≠ 写入yaml
APISIX不会自动把内存配置回写到yaml文件
```
{: .prompt-tip }

>为什么`DB-less`模式设计成不自动回写`yaml`？
- 官方设计初衷：`DB-less`面向`GitOps`声明式部署。
- 期望流程：代码仓库维护`apisix.yaml` → `CI`下发文件到网关 → `reload`生效。
- 不设计自动回写，防止多人通过`UI`随意改动，破坏`yaml`单一可信源。
{: .prompt-tip }

了解了两种路线的背景后，再聚焦到痛点，

原生`etcd`模式痛点，
- `APISIX`网关启动强依赖`etcd`连通性；`etcd`宕机/网络不通，`APISIX`进程直接启动失败；
- 所有数据平面节点运行时持续和`etcd`建立`watch`长连接，大规模集群场景`etcd`连接压力大；
- 离线环境、边缘节点无法部署`etcd`时无法落地；
- 缺少静态配置文件载体，难以直接对接`GitOps`配置入库、离线备份。

原生`DB-less`模式痛点，
- 原生禁用`Admin API`，无法使用`APISIX Dashboard`，只能手工编辑`yaml`，运维效率极低；
- 缺少统一配置管控中心，多`APISIX`节点需要人工同步`yaml`，极易出现节点配置不一致，无单一可信源；
- 变更入口混乱：任何人可直接修改本地`yaml`，线上配置溯源困难，存在人为误操作风险；
- 没有集中存储，配置分散在各个网关服务器，统一备份、审计困难。

>思考一个问题，有没有办法通过二次开发，补上原生缺失的`etcd`→`yaml`的回写通道，且不会破坏单一可信源，实现`DB-less+Dashboard`方式长期管理路由，解决上面所述的痛点呢?
{: .prompt-tip }

先说答案，技术方案上是可以实现的。

运行时路由表仍然来自`apisix.yaml`（`config_center=yaml`），只是`yaml`不再手工维护，而是由`etcd`自动`dump`生成（这时的`yaml`从**可信源**降级成**导出产物**），同时做好快照版本。`etcd`从**数据面运行依赖**变成**控制面同步总线**，将会是一个合适的想法。

>先厘清单一可信源到底约束什么？
{: .prompt-tip }

`APISIX standalone(yaml)`模式的官方初衷，核心不是`yaml`这个文件很重要，而是只允许一个写入方/一个权威源。声明式配置的价值在于，
- 配置是`GitOps/CI`管出来的，可`review`、可回滚、可审计；
- `APISIX`只读、不回写，所以运行态永远等于你`commit`的那份`yaml`；
- 不存在有人在运行时偷偷改了，和仓库里的不一致的问题。

所以真正要防的是双写（两个都能改、又互相覆盖的入口），而不是`yaml`不能被程序生成。

现在技术方案中，`yaml`不再依赖`GitOps/CI`里的配置，而是改由`etcd`控制面总线同步。每个节点的特权进程各自`watch`同一个`etcd version key`，各自把全量`dump`成自己本地的`apisix.yaml`，每个普通`worker`每秒轮询一次文件（`pull`模式）。由于是`pull`模式，这件事根本没有中心分发，自然也做到了集群多节点的数据一致同步。`Dashboard`只负责写`etcd`，和版本`+1`这一个触发信号。

>原生`yaml`模式是只读静态文件，原生`etcd`模式是改即生效无版本。把两者拼起来，`etcd`当控制面同步总线+提交点(`version key`)，定制`APISIX`把`etcd`自动`dump`回`yaml`作为真正的运行时数据源，从而在保留`yaml`模式容灾优势的同时，获得了`UI`动态改配、批量原子发布、全局快照回滚等机制。
- 提交发布：原生`etcd`模式是每条`key`变更各自`watch`、各自生效，批量发布时会经历中间不一致态。考虑在写完多条`etcd`后只提交一次，`APISIX`只在`version`变化时整体重生成`yaml` → `reload`。批量发布具备原子提交语义。
- 快照回滚：把整棵`/apisix/*`存成一条`JSON`记录（全量`etcd`快照 ），也能把任意历史快照整体写回`etcd`再`dump`版本，达到回滚的效果，原生`etcd`模式没有全局时间点回滚。
- 容灾：同时考虑到容灾/可审查性，配置以纯文本`yaml`落地，且每个版本一个`apisix_<ver>.yaml`。即使`etcd`挂了，数据面仍能靠`yaml`独立启动运行，`etcd`不再是网关转发的硬依赖。
{: .prompt-tip }

可以看到，针对`etcd`的痛点，能有以下收益，
1. 解除网关运行时对`etcd`强依赖（`DB-less`核心收益）
- `APISIX`只读取本地`yaml`启动，`etcd`故障不影响现有网关运行、不阻止网关启动。网关启动只依赖磁盘`yaml`文件，流量持续可用；`etcd`仅作为控制面存储，控制面故障只会阻断新增配置下发，不会断流量。
2. 减轻`etcd`连接压力
- 不再是`N`个`APISIX`节点同时`watch etcd`；仅`1`套同步程序`watch etcd`，同步程序负责生成`yaml`分发；数据平面网关不再直连`etcd`，大幅降低`etcd`长连接数量。
3. 保留`Dashboard`可视化运维能力
- 原生`etcd`模式才有可用`Admin API`与`Dashboard`；这套架构继承该能力，不会因为改用文件加载而丢掉图形化管理。
4. 天然产出声明式`yaml`文件
- 配置可以持续落盘为标准`apisix.yaml`，支持`Git`归档、版本对比、离线灾备、环境复制。

针对`DB-less`的痛点，能有以下收益，
1. 补齐可视化管理能力
- 配置统一在`Dashboard`操作，复用官方成熟的路由编辑、测试、导入导出、权限能力，不再手写`YAML`；
2. 建立严格单一可信源，消除配置不一致风险
- 规定`etcd`作为唯一可信源，所有`yaml`只是镜像；禁止直接修改节点本地`yaml`。所有变更收敛到`Admin API`，具备完整操作日志、审计；多节点`yaml`由同步程序统一分发，天然保证所有网关配置一致。
3. 统一配置集中存储
- `etcd`作为中央配置仓库，集中保存全量路由，便于全局检索、备份、灰度规划；`yaml`只是下发至数据平面的载体。

>这里的`etcd`全量落回`yaml`实现方式，和不设计自动回写，防止多人通过`UI`随意改动，破坏`yaml`单一可信源的官方初衷冲突吗？
{: .prompt-tip }

冲突不冲突，看权威方向，分下面两种情况，
1. `yaml`是权威，`etcd`只是运行态缓存（`yaml` → `etcd`）
- 启动时把`yaml`灌进`etcd`给`APISIX`用，`UI`只读展示。这不冲突——`yaml`仍是唯一可信源，`etcd`是派生物。
2. `etcd/UI`是权威，`yaml`是它的快照（`etcd` → `yaml`，即全量落回）
- 这时候`yaml`从可信源降级成了导出产物/备份。这确实和官方初衷冲突。

冲突点在于，
- 谁要是直接改了yaml（比如运维手动改、`Git`里改），下一次全量落回就会被`etcd`覆盖，改动静默丢失；
- 可信源事实上变成了`etcd`，`yaml`只是它的投影。你没消灭双写，只是把权威从`yaml`挪到了`etcd`。

所以，不做自动回写和全量落回是两回事。不设计自动回写来防止多人随意改动，本质是想保住`etcd/UI`不能成为权威。而全量落回`yaml`如果是`etcd`→`yaml`方向，恰恰是让`etcd`成了权威。两种设计方案是矛盾的。

要既落回`yaml`、又不破坏单一可信源，通常得满足其一，
- 落回的`yaml`重新进入`Git`/审批流程：`UI`只是生成配置的编辑器，产物必须走`review`才生效。这样权威仍在`Git`，`UI`不是运行时热更新入口；
- 明确`yaml`单向、只读方向：运行态永远从`yaml`生成，禁止人工直接改`etcd`之外的东西，UI关掉或只读；
- 接受`etcd`为权威，把`yaml`明确定义为导出快照/灾备，并在文档和流程上禁止直接编辑`yaml`（靠约定而非机制，风险较高）。

最糟的组合就是：`UI`能改etcd、程序又全量落回`yaml`、同时还允许人改`yaml`——三个写入方，没有任何一个是明确权威，这才是真正破坏初衷的场景。

这时`yaml`从可信源降级成了导出产物/备份，谁要是直接改了`yaml`（比如运维手动改、`Git`里改），下一次全量落回就会被`etcd`覆盖，改动静默丢失。这在单向同步+`etcd`为唯一可信源的架构里是结构性风险，不是偶发`bug`。只要满足两个条件就必然发生，
1. 同步是单向的：`etcd`→`yaml`（导出），而`yaml`没有回写`etcd`的通道。
2. 全量落回是覆盖式而非合并式：重新生成`yaml`时是按`etcd`当前状态整份重写，不比对、不保留本地差异。

在这两个前提下，任何绕过`etcd`的`yaml`改动（运维手改文件、`Git`里改后部署）都是游离态的。它没进`etcd`，而`etcd`也不知道它的存在。下一次触发全量落回（重启、定时快照、手动`dump`、`watcher`全量`resync`）时，`etcd`的状态会原样盖回`yaml`，改动静默丢失，而且没有任何冲突提示，因为系统根本不认为这是冲突，在它的世界观里`etcd`永远是对的。

关键在于把`yaml`从可信源降级成导出产物这个动作本身，同时也取消了它作为输入的合法性。降级后，
- `yaml`改动有时生效，只是因为还没触发落回，这是最危险的。它给运维一种改`yaml`管用的错觉，埋到下一次`resync`才爆。
- 时间窗口不可预测：如果落回是事件驱动(`watcher resync`)而非定时，丢失可能在任意时刻发生。

要不要接受这个风险，取决于你的定位，
- 如果`yaml`确实只是备份/审计快照，这就是设计意图，不是缺陷。但需要在流程和工具上封死`yaml`的写入路径，比如文件设为只读、加显式头注释`# GENERATED, DO NOT EDIT`、`CI`里拒绝对该文件的手工`diff`、运维文档明确改配置只能走`etcd/dashboard`。风险来自人以为能改。把这条路堵死，风险就消失了。
- 如果现实中运维确实需要能改`yaml`(`GitOps`场景)：那`yaml`就不该被降级成纯导出产物，单向同步就是错的方向。需要的是双向或以`yaml`/`Git`为源的模型(`yaml`→`etcd reconcile`)，或至少在落回前做三方`diff`，发现`etcd`与`yaml`不一致时告警而非静默覆盖。

>然而，这个思路并没有完全消除两种原生模式所有短板，只是互相取长补短，同时也引入新代价。
{: .prompt-tip }

仍然遗留`DB-less`本身固有的机制短板（无法规避）
- `APISIX`加载机制不变：`yaml`文件更新触发全量重载整套配置，不存在`etcd`原生的增量内存更新；上万条路由频繁变更依然存在`CPU`抖动；
- 配置同步存在两级延迟：`Dashboard`写入`etcd`（毫秒）→同步程序导出`yaml`→文件分发到节点→`APISIX` `1s`轮询检测文件变更；整体最大延迟高于原生`etcd watch`直连方案；
- `yaml`存在文件半写风险，必须严格遵守`#END`规范、原子写文件。

新增二次开发带来的工程负担
- 需要自行维护`etcd watch` → `yaml`序列化 → 文件分发同步组件，属于自研代码，要处理，
  - `etcd`断连重连、版本对齐；
  - `yaml`序列化格式严格兼容`APISIX standalone`语法；
  - 文件原子下发，避免网关读到半截`yaml`；
  - 一致性校验：定时对比`etcd`快照与各节点`yaml`，发现漂移自动修复；
  - 生产落地风险清单，以及对应的配套解决方案，详细参考附录网关存储优化方案：生产落地风险清单+配套解决方案；
- 架构链路变长：用户→`Dashboard`→`etcd`→自研同步器→`yaml`→`APISIX`，故障排查链路比原生架构更长。

>总结下，该方案本质是控制面沿用`etcd`生态的管理优势，数据面采用文件无中心运行模式，牺牲一部分配置更新实时性与增量更新能力，换取网关更高的运行独立性。
{: .prompt-tip }

原生`etcd`模式优势是动态`API`、集群一致性、增量同步，但网关强依赖`etcd`；原生`yaml`模式零外部依赖、网关不依赖中心组件，但缺失`Dashboard`、缺乏统一管控、变更入口散乱。通过搭建`etcd`做唯一可信源 + `Dashboard`管理 + 自研`etcd`->`yaml`单向同步通道 + `APISIX`以`DB-less`方式运行，
- 解决`etcd`模式短板：网关启动和运行不再强依赖`etcd`，边缘/离线场景可落地，降低`etcd`连接压力；同时保留集中配置存储；
- 解决原生`yaml`模式短板：打通`Dashboard`可视化管理；收敛配置变更入口，确立单一可信源，解决多节点`yaml`同步混乱、无审计、随意修改文件的问题。

数据流单向约束：唯一可信源 = `etcd`，禁止反向同步。另外需要补充边界红线（强制规范），
- 禁止任何人直接登录网关节点修改`apisix.yaml`；
- 禁止任何程序把`yaml`配置反向回写到`etcd`；
- 所有配置变更入口收敛：`人员 → Dashboard → etcd`；
- `yaml`只是`etcd`配置的镜像副本，不具备独立权威。

#### **服务注册发现（APISIX与Consul）**

`Consul`是用`Go`语言开发的，是一个支持多数据中心、分布式、高可用的微服务基础组件，支持健康检测与基于`HTTP`和`DNS`协议的查询调用，服务注册发现与配置共享是`Consul`的重要功能。`Consul`内部采用`Raft`一致性算法来保证服务的高可用性，使用`gossip`协议来管理成员和广播消息，并且支持`ACL`访问控制，提供了一个现代的、灵活的、强大的基础架构，是作为微服务注册中心的最佳选择。

`Consul`的特性如下，
- 服务发现：`Consul`的客户端提供对可用服务的查询功能，我们可以使用服务的标识去发现指定服务的所有提供者，在`Consul`内部可以通过`DNS`或者`HTTP`协议找到外部服务所依赖的服务；
- 健康检测：`Consul`客户端提供健康检测功能，我们可以通过指定健康检测的地址和指定频率，及时得知一个服务是否处于健康状态，以避免将流量发送到不健康的服务节点；
- 键/值存储：在应用程序中，用户可以根据自己的需要使用`Consul`提供的键/值存储。它用于实现动态配置、功能标记和协调等，通过简单易用的`HTTP`接口即可调用；
- 多数据中心：`Consul`支持开箱即用的多数据中心机制。

对比`etcd`、`ZooKeeper`和`Consul`三种服务发现工具，如下，
- `Consul`支持分布式健康检测功能，可以指定任意节点进行检测，`etcd`不提供此功能；
- `Consul`提供内置的`Web`界面管理功能，`etcd`不提供此功能；
- `Consul`全面支持服务网格解决方案；
- `Consul`内部使用`Raft`算法来保证一致性，比`ZooKeeper`的`Paxos`算法更为简单、直接、有效；
- `Consul`支持`HTTP`和`DNS`协议接口。`ZooKeeper`的集成较为复杂，`etcd`只支持`HTTP`协议；
- `ZooKeeper`临时节点在客户端断开连接时删除键/值项，相比心跳机制更复杂；另外，所有客户端必须保持到`ZooKeeper`服务器的活动连接，客户端调试较为困难；
- `Consul`支持跨数据中心，采用不同的端口监听内外网的服务；`ZooKeeper`和`etcd`均不提供多数据中心功能。

可以发现它们各有优劣。总体来说，选择`Consul`更为适合。`Apache APISIX`对接`Consul`有两种模式，分别是`HTTP`模式和`DNS`模式，这里用的是`HTTP`模式部署，它们之间的区别详情参见附录`Kong.vs.APISIX`对接`Consul`底层实现对比。

`APISIX`是`Consul`的消费者，不会把自己注册进`Consul`。数据流是单向的，

```markdown
web1/web2  ──(注册)──►  Consul  ◄──(拉取/查询)──  APISIX
              ▲                                    │
       注册脚本 / sidecar                     discovery.consul
                                             （只读服务目录）
```

| 组件 | 对`Consul`的行为 |
|------|------------------|
| `web1/web2` | 被注册进去（被动） |
| 注册脚本/`sidecar` | 主动写入注册 |
| `APISIX` | 只读拉取，不注册自己 |

`APISIX`作为流量入口，客户端直接访问它，它无需被服务发现，所以`APISIX`注册到`Consul`的这个方向不存在、也不需要。

这里配置的是`discovery`（服务发现）。`APISIX`会周期性地去`Consul`拉取服务目录（`fetch_interval: 3`，表示每`3`秒拉一次），把结果缓存到本地`discovery`共享字典（`discovery: 1m`），供上游`discovery_type=consul`的路由使用。

```yaml
discovery:
   consul:
      servers:
         - http://consul:8500
      fetch_interval: 3
      refresh_interval: 15
```

服务发现端（`APISIX`，已在`config.yaml`配好），`APISIX`启动后连到`consul:8500`，周期性拉取`Consul`的健康服务列表并缓存到共享内存。路由/上游里用`discovery_type: consul` + `service_name` 引用即可。

服务注册端（需要自己做），`Consul`不知道`web1/web2`的存在，得由服务方或脚本调用`Consul`注册`API`。

正常服务都是服务自注册，本文例子中用的`sidecar`自动注册，最简单的`HTTP API`手动注册如下，

```terminal
# 注册 web1
curl -X PUT http://127.0.0.1:8500/v1/agent/service/register -d '{
  "ID": "web1",
  "Name": "my-web",
  "Address": "172.19.5.12",
  "Port": 8000,
  "Check": { "HTTP": "http://172.19.5.12:8000/web1.txt", "Interval": "10s" }
}'

# 注册 web2：同一个 Name 才能被负载均衡到一起
curl -X PUT http://127.0.0.1:8500/v1/agent/service/register -d '{
  "ID": "web2",
  "Name": "my-web",
  "Address": "172.19.5.13",
  "Port": 8000,
  "Check": { "HTTP": "http://172.19.5.13:8000/web2.txt", "Interval": "10s" }
}'

# 校验
curl http://127.0.0.1:8500/v1/catalog/service/my-web
```

注册要点
- `service_name`必须匹配：路由里的`service_name`=`Consul`注册时的`Name`（示例是`my-web`）；
- 只发现健康实例：`APISIX`只会发现通过健康检查的节点；
- 区分`consul`与`consul_kv`：前者基于`Consul`原生`catalog/health`接口，后者把节点写进`KV`，两者配置与注册方式不同，别混用；
- 生产环境更推荐**服务自注册**（应用启动注册、退出反注册）或用**sidecar**自动注册，避免手动脚本。

>`APISIX`怎么用这些数据？<br/>
在`APISIX`里建`upstream`时用，`APISIX`就会从`Consul`拿到`my-web`下的两个实例做负载均衡。
{: .prompt-tip }

创建路由引用`Consul`服务，也可以直接在`DashBoard`管理端配置，

```bash
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
  -H "X-API-KEY: nWWPdkrEQGdtKamXuJkJgdkARUvmSmJI" \
  -X PUT -d '{
    "uri": "/*",
    "upstream": {
      "service_name": "my-web",
      "discovery_type": "consul",
      "type": "roundrobin"
    }
  }'
```

多打几次代理入口，就能看到在`web1`、`web2`之间轮询，

![Desktop View](/assets/images/20260724/request_route_diff_upstream.png){: width="800" height="400"}
_负载均衡_

### **启动apisix集群**

`docker-compose`和其他配置文件准备好后，就可以运行下面的命令，启动`APISIX`服务集群了，

```terminal
mkdir -p ./apisix_log ./apisix_conf ./etcd_data ./consul_data
docker compose down -v      # 清理旧环境，避免旧数据冲突
docker compose up -d
docker compose logs -f apisix
```

下面是一些常用的入口，

| 服务 | 地址 | 备注 |
|------|------|------|
| 网关代理入口 | `http://127.0.0.1:9080` | 业务流量 |
| `APISIX`内置`UI` | `http://127.0.0.1:9180/ui/` | 输入`admin key` |
| `APISIX Admin API` | `http://127.0.0.1:9180` | 管理接口 |
| `Consul UI` | `http://127.0.0.1:8500` | 服务注册中心 |
| `web1`直连 | `http://127.0.0.1:9081/web1.txt` | 后端`1` |
| `web2`直连 | `http://127.0.0.1:9082/web2.txt` | 后端`2` |

### **简单测试流程**

打开`Apisix DashBoard`控制台，看下有哪些元素，

![Desktop View](/assets/images/20260724/apisix_dashboard_ui_new.png){: width="800" height="400"}
_Apisix DashBoard内置UI_

![Desktop View](/assets/images/20260724/apisix_dashboard_ui_old.png){: width="800" height="400"}
_Apisix DashBoard老版UI，有些选项设置不兼容_

查看`Consul`注册中心，

![Desktop View](/assets/images/20260724/consul_homepage.png){: width="800" height="400"}
_Consul主页_

![Desktop View](/assets/images/20260724/consul_register_service.png){: width="800" height="400"}
_Consul已注册服务_

![Desktop View](/assets/images/20260724/consul_register_node.png){: width="800" height="400"}
_Consul注册服务对应的节点_

查看etcd的可视化数据，

![Desktop View](/assets/images/20260724/etcdkeeper_ui.png){: width="800" height="400"}
_etcd ui_

以上都确认好了之后，简单创建一个上游和一条路由规则，测试下联通性，

![Desktop View](/assets/images/20260724/upstream_create.png){: width="800" height="400"}
_在老版本UI创建一个上游_

这里使用服务发现，老版本UI这里有个问题，不兼容`consul`这种服务发现方式，于是去新版本`UI`修正下，

![Desktop View](/assets/images/20260724/upstream_create_new.png){: width="800" height="400"}
_在新版本UI修正上游的服务发现类型_

现在上游列表里有对应一条上游数据，

![Desktop View](/assets/images/20260724/upstream_list.png){: width="800" height="400"}
_上游列表_

上游创建好后，再对应增加一条路由规则，这里域名使用`web.lvh.net`，并关联上面创建好的上游，

![Desktop View](/assets/images/20260724/route_create.png){: width="800" height="400"}
_创建路由规则_

![Desktop View](/assets/images/20260724/route_create_set_upstream.png){: width="800" height="400"}
_创建路由规则：设置上游_

现在路由规则列表里有对应一条路由数据，

![Desktop View](/assets/images/20260724/route_list.png){: width="800" height="400"}
_路由规则列表_

现在再回来看看`etcd`里边的数据，

![Desktop View](/assets/images/20260724/upstream_etcd.png){: width="800" height="400"}
_路由规则_

![Desktop View](/assets/images/20260724/route_etcd.png){: width="800" height="400"}
_上游_

现在配置好了一条能访问到上游节点的路由规则，请求多打几次代理入口，

![Desktop View](/assets/images/20260724/curl_request.png){: width="800" height="400"}
_多请求几次_

可以看到请求到后端节点，在`web1`、`web2`之间轮询，

![Desktop View](/assets/images/20260724/request_route_diff_upstream.png){: width="800" height="400"}
_轮询日志_

到这里，简单的`hello world`测试就结束了。

### **常用运维命令**

```bash
# 进容器看有哪些内置插件（有对应 .lua 就是内置）
docker exec -it apisix ls /usr/local/apisix/apisix/plugins/ | grep "\.lua$"

# 查看主配置 / 日志
docker exec apisix cat /usr/local/apisix/conf/config.yaml
docker exec apisix tail -f /usr/local/apisix/logs/error.log

# APISIX 运维
docker exec apisix /usr/local/apisix/bin/apisix version
docker exec apisix /usr/local/apisix/bin/apisix reload
docker exec apisix /usr/local/apisix/bin/apisix test
```

## **附录**

### **Nginx、Kong、Apache APISIX、Spring Cloud Gateway 横向对比**

![Desktop View](/assets/images/20260724/gateway_develop_timeline.png){: width="800" height="400"}
_网关演进时间线_

>先理清定位，
- `Nginx`：底层反向代理、`Web`服务器，不是完整`API`网关；
- `Kong`、`APISIX`：基于`OpenResty`(`Nginx+Lua`)的边缘`API`网关（流量入口、面向外网/南北向流量）；
- `Spring Cloud Gateway`：`Java`生态微服务网关（偏向集群内部、东西向流量）。
{: .prompt-tip }

#### **架构比对**

`Nginx`（`OpenResty`）
- 语言：`C + Lua`扩展（`OpenResty`）
- 配置存储：本地静态配置文件
- 变更生效：必须`reload`/重启（`reload`有短暂开销）
- 模型：无内置`Admin API`，无集群配置同步能力

> `OpenResty = Nginx + LuaJIT`，原生`Nginx`不支持`Lua`。
{: .prompt-tip }

`Kong Gateway`
- 底层：`OpenResty`(`Nginx+LuaJIT`)
- 两种模式
  1. `DB`模式（传统）：配置存`PostgreSQL`；节点定时轮询拉取配置（默认`5s`延迟）
  2. `DB-less`无库模式：静态`yaml`声明配置，更新需要重载
  3. `Hybrid`混合模式：分离控制面/数据面
- 短板：`DB`模式引入数据库单点风险；大量高级插件企业版收费

`Apache APISIX`
- 底层：`OpenResty`(`Nginx+LuaJIT`)
- 配置中心：`etcd`（`K8s`原生组件）
- 机制：`etcd watch`毫秒级推送配置，节点无状态，完全不需要`reload`
- 所有核心插件`Apache2.0`开源，无功能阉割；支持`Lua/Go/Java/Python/Wasm`多语言插件
- 原生支持`APISIX Ingress Controller`，完美适配`K8s`

`Spring Cloud Gateway`
- 语言：`Java`/ `Netty`响应式
- 定位：`Spring Cloud`微服务生态内网关
- 配置：配合`Nacos`/`Apollo`配置中心动态刷新
- 短板：`JVM`开销大，性能远低于`Nginx`系；适合内网微服务，不适合超大并发外网入口

#### **关键纬度比对**

|对比项|`Nginx(OpenResty)`|`Kong OSS`|`Apache APISIX`|`Spring Cloud Gateway`|
| ---- | ---- | ---- | ---- | ---- |
|诞生年份|`2004`|`2015`|`2019`|`2018`|
|底层|`C`语言事件驱动|`OpenResty(Nginx+Lua)`|`OpenResty(Nginx+Lua)`|`Java Netty Reactor`|
|操作系统进程模型|`Nginx Master/Worker`|依附`Nginx`，无独立网关进程|依附`Nginx`，无独立网关进程|独立`JVM`进程|
|动态配置|❌ 需`reload`|✅ `DB`模式`5s`轮询；`DB-less`需重载|✅ `etcd watch`，毫秒热更新，无需`reload`|✅ 配置中心动态刷新，无需重启网关进程|
|配置存储|本地`conf`文件|`PostgreSQL`/静态`yaml`|`etcd`（分布式`KV`）|配置中心(`Nacos`/`Apollo`)|
|配置同步方式|修改文件 + `reload`|节点`Pull`轮询（默认`5s`）|`etcd Watch Push`推送|配置中心主动拉取/事件通知|
|配置生效延迟|`reload`耗时|秒级延迟|毫秒级|视配置中心而定|
|后台定时任务|能力薄弱|`ngx.timer`（`Worker`内）|`ngx.timer`（`Worker`内）|`JVM`线程池|
|集群运维|手动同步配置，无原生同步|`DB`模式依赖数据库；扩容复杂|数据面无状态，水平扩缩容极简|依赖注册中心+配置中心|
|性能基准|最高（裸代理）|高，低于`APISIX`|很高，接近原生`Nginx`|中等，并发上限低|
|内置插件|极少，需要自行开发`lua`|基础插件免费；限流、`OIDC`等高级功能企业付费|全部核心插件开源免费，`100+`内置|基础`Filter`，自定义需`Java`开发|
|管理面板|无官方`Dashboard`|`Kong Manager`（企业版）|官方`apisix-dashboard`完全开源|无官方`UI`，需自研|
|多语言插件|仅`Lua`|`Lua`、`Go`|`Lua`、`Go`、`Java`、`Python`、`Wasm`|只能`Java`|
|协议支持|`HTTP/HTTPS/gRPC/WebSocket`|`HTTP/gRPC/WS`|`HTTP/gRPC/WS/MQTT/TCP/UDP`|`HTTP`为主|
|典型流量场景|静态资源、负载均衡、`CDN`、简单反向代理（路由长期不变）|传统企业`API`网关、对外开放`API`、存量海外业务|云原生、`K8s`、高并发边缘入口、`IoT`、频繁动态路由|`Spring Cloud`微服务内网网关|
|License|`BSD`|`Apache2.0`（高级功能闭源）|`Apache2.0`完全开源|`Apache2.0`|
|开源限制|完全开源|高级功能企业版收费|全部核心插件开源免费|完全开源|

#### **优缺点详解**

`Nginx / OpenResty`
- 优点
   - 性能天花板、极度稳定、内存占用极低；
   - 社区庞大，文档成熟；静态资源、`SSL`、缓存能力极强。
- 缺点
   - 没有原生动态路由，改路由/限流必须`reload`；
   - 缺少网关高阶能力：灰度、熔断、分布式限流、统一鉴权；
   - 集群部署需要自己做配置同步（`ansible/git/confd`）。

👉 适合：静态站点、简单负载均衡、`CDN`节点；不适合频繁变更路由的`API`网关场景。

`Kong Gateway`
- 优点
   - 诞生最早，生态成熟，海外项目使用广泛；
   - 插件体系完善，支持多种认证、流量治理；
   - `Hybrid`模式适合大规模多集群。
- 缺点
   - `DB`模式依赖`Postgres`，数据库成为故障点与扩容瓶颈；
   - `OSS`版本大量高级能力收费；
   - 配置变更存在秒级延迟；路由规模大时性能衰减明显。

👉 适合：海外业务、存量已经使用`Kong`、团队能接受商业授权；国内新项目越来越少选型。

`Apache APISIX`
- 优点
   - 无数据库依赖，仅etcd，架构简洁；
   - 配置实时推送，毫秒生效，不用`reload`；
   - 国内社区活跃、中文文档完善；`Docker/K8s`友好；
   - 开源版无功能阉割，内置`Dashboard`；支持`TCP/MQTT`物联网场景；
   - 节点无状态，容器扩缩容非常轻松（非常契合你`docker-compose`部署场景）。
- 缺点
   - 相比`Kong`起步晚，部分小众第三方插件较少；
   - 需要额外维护`etcd`集群（单机测试可以单`etcd`）。

👉 适合：云原生`K8s`、容器化部署、内外网统一网关、高并发`API`入口、国内中小/大型企业首选。

`Spring Cloud Gateway`
- 优点
   - `Java`技术栈无缝集成`Spring Cloud`；开发自定义`Filter`极其方便；
   - 支持`Spring Security`、服务发现（`Nacos/Eureka`）。
- 缺点
   - `JVM`内存开销大，延迟更高，不适合外网高并发入口；
   - 重启/刷新有波动；缺少成熟的可视化运维面板。

👉 适合：微服务集群内部网关；不要直接暴露到公网。

#### **清晰选型建议**

1. 公网边缘流量、高并发、容器/`K8s`、需要频繁调整路由
- `Apache APISIX`（当前国内最优均衡方案）
2. 已有`PostgreSQL`运维体系、海外业务、历史项目迁移
- `Kong`
3. 仅仅做静态资源、简单四层/七层负载，路由长期不变
- `Nginx/OpenResty`
4. 纯`Spring Cloud`微服务集群，只做内网流量转发，团队全`Java`
- `Spring Cloud Gateway`

> **最佳实践架构（常用分层）**<br/>
外网：`Nginx`（`SSL`、静态资源、防`CC`） → `Apache APISIX`（`API`治理、鉴权、灰度、限流） → `Spring Cloud Gateway`（微服务路由） → 业务服务
{: .prompt-tip }

>补充：`APISIX vs Kong`最容易踩坑的区别
1. 配置同步机制
- `Kong DB`模式：节点主动轮询拉取；`APISIX`：`etcd`主动推送。路由上千条时差距明显。
2. 运维成本
- `Kong`需要维护`PostgreSQL`；`APISIX`只需要`etcd`（`K8s`环境`etcd`属于基础设施，复用成本低）。
3. 开源边界
- `Kong`很多高级限流、`OIDC`、`RBAC`在开源版没有；`APISIX`全部核心功能开源。
{: .prompt-tip }

![Desktop View](/assets/images/20260724/gateway_vs_arch.png){: width="800" height="400"}
_四大网关横向对比图_

### **特权进程**

![Desktop View](/assets/images/20260724/privileged_agent_process.png){: width="800" height="400"}
_特权进程的位置_

#### **回顾标准Nginx原生进程模型**

标准`Nginx`只有两类进程，
1. `Master`主进程
    - 读取配置、管理信号、创建/回收`Worker`、监控`Worker`存活
    - 不处理业务流量，不执行`Lua`
2. `Worker`工作进程（多个）
    - 处理客户端`HTTP`请求
    - `OpenResty`下，`Worker`内部嵌入`LuaJIT VM`，执行所有请求链路`Lua`代码

> 原生`Nginx`：没有任何额外独立进程
{: .prompt-tip }

#### **OpenResty引入：Privileged Agent特权进程**

>背景限制<br/>
`Nginx Worker`有一个硬性约束：`Worker`运行在非特权用户（通常`nginx`用户），并且`Worker`内部禁止执行阻塞调用、不适合长时间阻塞任务。
{: .prompt-tip }

常见受限场景，
- 需要`root`权限执行操作
- 需要长时间阻塞的任务（外部同步、定时批量任务、独立长连接）
- 不希望被客户端请求事件抢占调度的后台任务

`Worker`里使用`ngx.timer`的短板，
1. `Timer`依附`Worker`，如果`Worker`崩溃，任务中断；
2. `Timer`受事件循环调度影响，高流量下被请求挤压；
3. `Worker`权限受限，无法执行需要高权限操作。

因此`OpenResty`新增：**特权进程`privileged agent`**

特权进程和`Master`、`Worker`的层级关系，如下，
1. 由`Master`进程`fork`出来，生命周期受`Master`管控；
2. 独立进程，不属于`Worker`；
3. 默认可以配置为`root`/高权限用户运行（`Worker`一般降权）；
4. 不监听网络端口、不处理客户端流量。

> 重点说明，
- `Worker`：处理`HTTP`请求，面向流量
- 特权进程：纯粹执行后台`Lua`任务，不接收客户端连接
{: .prompt-tip }

#### **特权进程能做什么**
1. 执行需要系统特权的操作；
2. 运行长期持续的后台任务、独立长连接；
3. 全局定时任务，不受业务流量波动影响；
4. 全局数据预热、外部资源同步、日志持久落地；
5. 注意：特权进程只有一个！全局单实例，不会启动多个。

>重要限制，
- 特权进程没有`Nginx`请求上下文；
- 不能使用大部分`ngx.xxx`请求相关`API`（`ngx.req`、`ngx.var`、`ngx.exit`等）；
- 只能使用无请求上下文的`Lua API`。
{: .prompt-tip }

#### **和Worker内ngx.timer的核心对比**

|项目|`Worker ngx.timer`|`Privileged Agent`特权进程|
|----|----------------|--------------------------|
|运行载体|`Nginx Worker`进程内部|独立`OS`进程，`Master`派生|
|数量|每个`Worker`都可创建定时器|全局仅`1`个进程|
|权限|跟随`Worker`，一般低权限|可配置高权限运行|
|是否处理流量|同时处理`HTTP`请求+任务|完全不处理客户端流量|
|调度影响|高并发请求会挤压定时器执行|不受业务流量影响|
|崩溃影响|`Worker`退出，定时器消失|独立生命周期，与`Worker`解耦|
|可用`API`|完整`Nginx Lua`请求`API`|禁用请求相关`API`|

#### **结合Apache APISIX场景理解**

>先澄清一个极易混淆的知识点：
默认情况下，`APISIX`的`etcd watch`、健康检查，早期版本全部跑在`Worker`的`ngx.timer`里，不是特权进程。
{: .prompt-tip }

`APISIX`什么时候使用特权进程？
1. 部分全局一次性初始化任务；
2. 某些需要独立后台、不占用`Worker`事件循环的批处理任务；
3. 部分需要特权操作的插件能力；

绝大多数网关流量治理逻辑依然在`Worker`内部执行。

#### **要点总结**
1. `APISIX`没有独立的`apisix`后台守护进程；
2. 绝大多数后台任务跑在`Worker` `ngx.timer`；
3. 特权进程是可选的、单实例独立`Lua`进程，仅用于特定全局后台任务，不是网关必备主流程。

### **APISIX DB-less vs Nginx**

>大体思路相似，但不完全等价；是同源，都依赖静态文件作为唯一可信源，但能力上有明显区别，`APISIX DB-less`可以理解为，介于原生`Nginx`和`etcd`集群模式中间形态。
- `DB-less(yaml)`和`Nginx`共同点：不能靠`API`动态持久变更配置，最终必须修改静态文件+重载；
- 但`APISIX DB-less`相比原生`Nginx`依然保留网关高级能力（插件、`Admin API`、路由语法更强大），只是丢掉了`etcd`分布式动态持久化能力。
{: .prompt-tip }

#### **相同点（直观上，你所能感受到的退化）**

从动态配置运维体验上看，确实退化到和`Nginx`同一套运维范式。
1. 配置单一可信源 = 本地静态文件
    - `Nginx`：唯一来源 `nginx.conf`
    - `APISIX DB-less`：唯一来源 `apisix.yaml`
2. 运行时通过`API/UI`新增的路由，只存在内存，不会落盘；重启/`reload`全部丢失
3. 想要永久生效的规则，必须修改磁盘上的静态文件，再执行重载
4. 天然不适合多节点集群动态同步：多实例需要保证所有节点`yaml`文件一致，需要外部工具同步配置（`ansible`/`git`/配置分发）

#### **关键不同点（不能完全划等号）**

重载代价不一样
- `Nginx`
   - 修改`conf` → `nginx -s reload`
   - 会重建监听、重新解析所有配置，存在一定开销；大量配置下影响明显。
- `APISIX DB-less reload`
   - `APISIX`内部机制优于原生`Nginx`：`apisix reload`不会重建监听`socket`；只是重新加载`yaml`规则更新内存路由，平滑度优于传统`Nginx reload`。

运行时临时调试能力不同
- `APISIX DB-less`模式下，你依然可以调用`Admin API`/`Dashboard`临时新增路由，测试流量；只是不能持久化，一旦`reload`就清空。
- 原生`Nginx`：运行时无法通过`API`临时添加路由，只能改文件。

> 形象区分，
- `Nginx`：运行时完全不能临时改规则
- `APISIX DB-less`：运行时可以临时改，但无法保存，重启归零
{: .prompt-tip }

配置表达能力差距巨大
- `nginx.conf`语法底层，灰度、权重、复杂限流、多鉴权、`gRPC`代理等需要大量`C`模块或`OpenResty Lua`开发；
- `apisix.yaml`内置上百种网关插件，声明式配置路由、熔断、限流、跨域、灰度，开箱即用。

集群模式差异
- `Nginx`集群：需要外部工具同步`conf`
- `APISIX DB-less`集群：同样需要外部同步`apisix.yaml`

二者运维手段一致；但`APISIX`缺失`etcd`之后，失去原生集群配置自动同步能力。

#### **分层对比表**

|维度|原生`Nginx`|`APISIX DB-less`（`yam`l）|`APISIX etcd`模式|
|----|----|----|----|
|永久配置来源|`nginx.conf`|`apisix.yaml`|`etcd`|
|想要永久变更路由|修改`conf + reload`|修改`yaml + apisix reload`|`Admin API/Dashboard`直接操作，无需改文件|
|运行时能否临时新增路由|❌ 不支持|✅ 支持，仅内存生效|✅ 支持，持久化保存|
|集群配置自动同步|❌ 无原生能力|❌ 无原生能力|✅ `etcd watch`自动同步|
|重载影响|相对较重，重建监听|较轻，只刷新路由规则|不需要`reload`|

#### **小结**

`APISIX DB-less yaml`模式，在运维范式上退化至类似`Nginx`：持久规则依赖静态文件，无法通过`UI/API`永久保存路由。

但不属于完全退回`Nginx`：它依然保留`APISIX`丰富的插件体系、更友好的声明式配置、支持运行时临时调试路由，只是舍弃了`etcd`提供的分布式动态配置能力。

最佳实践提醒
1. 多节点集群、日常需要频繁调整路由 → 禁止使用`DB-less`，必须`etcd`模式；
2. `DB-less`适合场景，
   - 简单单机、流量规则长期不变；
   - `GitOps`声明式交付，一切配置代码化，不允许任何人通过`UI`随意改动网关规则；
3. 对于原生`APISIX`，不要指望`DB-less + Dashboard`长期管理路由，`UI`只能临时测试，重启全部丢失。

### **网关存储优化方案：生产落地风险清单 + 配套解决方案**

#### **风险1：yaml文件半写问题（原子写yaml）**

##### **风险描述**

同步器直接覆盖`apisix.yaml`，若写入过程中断（进程崩溃、网络中断），产生残缺`yaml`；

`APISIX standalone`依靠 `#END` 判断配置合法性，缺失标记则拒绝加载新配置，持续使用旧路由。

##### **解决方案**

1. 原子写入范式（强制落地）
   - 先写入临时文件`apisix.yaml.tmp`
   - 完整序列化所有资源，文件末尾自动追加`#END`
   - `fsync`刷盘成功后，使用`rename()`原子替换正式文件
   - `rename`在同一文件系统下是原子操作，网关永远只会读到完整/旧版本配置
2. 禁止直接`vi/echo`覆盖生产`yaml`；
3. 同步器写入完成后主动读取临时文件自检语法。

#### **风险2：etcd ↔ 各节点yaml配置一致性漂移（一致性校验）**

##### **风险描述**

多种场景引发不一致，
- `yaml`分发网络丢包，部分网关节点未收到新版本`yaml`；
- 人为违规手动修改节点本地`yaml`；
- 同步服务重启期间漏掉部分etcd变更事件；
最终出现不同`APISIX`实例路由规则不一致，流量处理行为分裂。

##### **解决方案**

1. 同步器每次生成`yaml`时，嵌入配置版本标识（写入`yaml`注释：`etcd revision`）；
2. 定时巡检任务：
   - 拉取`etcd`当前全局`revision`
   - 远程读取所有`APISIX`节点`apisix.yaml`内记录的`revision`
   - 对比，版本不一致立即触发告警，并自动重新下发最新`yaml`进行修复
3. 增加审计告警：检测文件`mtime`非正常变更（人为篡改）；
4. 提供`API`一键触发全集群强制同步。

#### **风险3：同步服务故障、分发失败（同步失败降级策略）**

##### **风险场景**

1. 同步服务进程崩溃、重启；
2. 同步服务与`etcd`网络中断，`watch`断开；
3. 同步服务与部分`APISIX`节点网络不通，`yaml`推送失败；

##### **核心降级原则：存量流量不受影响，只阻断新配置下发**

分层降级机制
1. `Watch`断连处理
- 同步器断开`etcd`后持续重试重连；重连成功后使用`etcd revision`补齐中断期间所有变更，避免丢失配置；
2. `yaml`下发失败降级
- 单节点推送`yaml`失败不阻塞整体流程，标记异常节点、持续重试，同时上报告警；
- 绝不删除节点上已有的旧`yaml`，网关持续运行旧路由；
3. 同步服务完全宕机终极降级
- `APISIX`节点依靠本地已有`yaml`正常转发流量；业务无感知；
- 运维收到告警后恢复同步服务即可，网关不需要重启；
4. 灾备：同步服务做集群部署，防止单点故障。

#### **风险4：错误配置下发全网，引发业务故障（配置回滚机制）**

##### **风险描述**

在`Dashboard`误操作路由、错误插件参数，同步器自动将错误`yaml`推送到所有网关，全集群路由异常。
原生架构缺少一键回滚通道。

##### **完整回滚方案**

1. 配置快照持久化
- 同步器每次生成新版本`yaml`，自动归档历史快照，关联对应的`etcd revision`；保留`N`个历史版本（例如`30`份）；
2. 支持灰度下发（可选高级能力）
- 不一次性推送到所有节点，先推送灰度网关集群验证，验证通过再全量推送；
3. 一键回滚能力
- 运维选择历史`revision`，系统执行流程：
- 选择历史快照 → 将历史配置写入`etcd` → `watch`触发同步器重新生成旧版本`yaml` → 分发至所有节点；
- 关键点：回滚依旧走`etcd`（唯一可信源），不能直接推送旧`yaml`绕过控制面，保证单一可信源不被破坏；
4. 前置校验
- 同步器序列化`yaml`前执行`Schema`校验，非法配置拒绝写入`etcd`、拒绝下发。

#### **风险5：APISIX Standalone 本身固有的短板（无法彻底消除，需要预案）**

##### **风险点**

1. 每次`yaml`更新，`APISIX`执行全量重载全部路由；上万条路由频繁变更时，`CPU`抖动、`GC`压力上涨；
2. 配置同步链路变长：`Dashboard`→`etcd`→同步器→分发→`APISIX` `1s`轮询，最大延迟高于原生`etcd`直连模式；

##### **缓解手段**

1. 控制配置变更频率，避免高频大量路由更新；
2. 监控`APISIX reload`耗时、`Worker CPU`波动；
3. 尽量将大量静态路由变更安排在业务低峰；

#### **风险6：链路复杂度上升，故障定位难度增加**

链路：客户端 → `Dashboard` → `etcd` → 自研同步器 → `yaml` → `APISIX`

相比原生`etcd`直连架构，多了一层自研组件。

##### **应对**

1. 全链路埋点日志：记录每次配置变更的操作人、`revision`、下发时间、各节点生效时间；
2. 提供排查工具：快速查询某个节点当前生效配置版本，并和`etcd`基准对比；
3. 同步器完善监控指标：`watch`事件数量、`yaml`生成耗时、分发成功率、校验不一致次数。

#### **小结**

|风险类别|核心危害|核心措施|
|---|---|---|
|原子写`yaml`|产生残缺配置，网关加载失败|临时文件+`rename`原子替换，强制`#END`，写入后自检|
|一致性漂移|多网关配置分裂，流量行为不一致|携带`etcd revision`，定时全集群巡检+自动修复|
|同步服务故障|新配置无法下发|断线重试、服务集群化；失败不删除节点旧`yaml`|
|错误配置全网推送|全集群业务故障|版本快照归档、灰度下发、依托`etcd`实现标准回滚|
|全量重载性能抖动|高频更新引发网关`CPU`冲高|管控变更频率，低峰执行大批量改动|
|单一可信源破坏|直接修改`yaml`导致配置溯源混乱|权限管控、篡改告警、制度约束+程序巡检|

### **Kong vs APISIX 对接Consul底层实现对比**

`Kong`内部直接使用`lua-resty-dns`与`Consul DNS`（`8600`端口）通信，依靠`DNS`协议做服务发现。

`Apache APISIX`对接`Consul`有两条独立路线。

#### **DNS模式（和Kong思路一致）**

`APISIX`同样支持`Consul DNS`服务发现（`Consul 8600`端口）
- 底层依赖：`lua-resty-dns-client`（`Kong`开源维护的库，内部封装了`lua-resty-dns`）
- 原理：`Upstream`配置`discovery_type: dns`，把`Consul`作为`DNS`服务器，通过查询`service-name.service.consul`获取所有健康实例。
- 相似点：和`Kong`一样，基于`Consul DNS`协议，由`Consul`负责健康检测、过滤不健康节点；网关只做`DNS`解析。
- 区别：`Kong`直接裸用`lua-resty-dns`；`APISIX`封装了一层`lua-resty-dns-client`，自带缓存、重试、负载均衡逻辑。

配置示例（`DNS`模式）如下，

```yaml
discovery:
  dns:
    servers:
      - "127.0.0.1:8600" # Consul DNS端口
```

上游引用示例，

```json
{
  "discovery_type": "dns",
  "service_name": "training.rest.service.consul"
}
```

#### **原生Consul HTTP发现（APISIX独有，推荐主流方案）**

`APISIX`内置独立的`Consul`服务发现模块，不走`DNS`，直接调用`Consul HTTP API`（`8500`端口）
1. `APISIX`启动`worker`后周期性向`Consul HTTP`接口拉取服务实例列表；
2. 本地缓存健康节点列表；
3. 请求转发时直接使用本地缓存节点，不再走`DNS`解析；
4. 支持监听变更、本地缓存持久`dump`、`tag`过滤、`datacenter`指定等能力。

这是和`Kong`最大的差异，
`Kong`官方没有独立`Consul HTTP`发现模块，只能走`DNS`；`APISIX`同时支持`DNS`模式+原生`Consul HTTP`拉取模式。

配置示例（`HTTP`原生`Consul`发现）如下，

```yaml
discovery:
  consul:
    servers:
      - "http://127.0.0.1:8500"
```

上游引用示例如下，

```json
{
  "discovery_type": "consul",
  "service_name": "training.rest"
}
```

#### **核心差异汇总**

| 项目 | `Kong` | `Apache APISIX` |
|------|------------------|---------------|
| 和`Consul`通信方式 | 仅`DNS`方案，`lua-resty-dns`查询`Consul 8600` | 方案`A`：`DNS（lua-resty-dns-client）`<br>方案`B`：`HTTP API`主动拉取（`8500`） |
| 协议 | `DNS`协议 | `DNS`/`HTTP API`二选一 |
| 健康检测责任 | `Consul`执行健康检查，`DNS`只返回健康节点 | 两种模式：<br>`DNS`模式：健康检查由`Consul`负责<br>`HTTP`模式：依然依赖`Consul`健康状态 |
| 服务变更感知 | `DNS TTL`缓存被动更新 | `HTTP`模式支持定时轮询，实时性可控 |
| 底层Lua库 | `lua-resty-dns` | `lua-resty-dns-client`（封装前者） |

#### **工程选型建议**
1. 如果想完全复刻`Kong+Consul`架构：`APISIX`使用`discovery_type: dns`，行为最贴近；
2. 生产环境推荐`APISIX`使用`discovery_type: consul`（`HTTP`模式），
- 可以拿到更多`Consul`元信息（`tag`、权重、元数据）；
- 不受`DNS TTL`限制，节点更新更及时；
- 便于排查，直接抓`HTTP`接口日志，比`DNS`报文调试简单。

## **参考**

[Apache APISIX 软件架构](https://apisix.apache.org/zh/docs/apisix/architecture-design/apisix/)

[有了 NGINX 和 Kong，为什么还需要 Apache APISIX](https://www.apiseven.com/blog/why-we-need-apache-apisix)

[为什么 Apache APISIX 选择 NGINX+Lua 技术栈？](https://apisix.apache.org/zh/blog/2021/08/25/why-apache-apisix-chose-nginx-and-lua/)

[APISIX AI 网关介绍](https://apisix.incubator.apache.org/zh/blog/2025/04/08/introducing-apisix-ai-gateway/)

[Apache APISIX 博客](https://apisix.apache.org/zh/blog/)

[Apache APISIX 在 API 和微服务领域的探索](https://apisix.apache.org/zh/blog/2022/07/22/exploration-of-apisix-in-api-and-microservices/)
