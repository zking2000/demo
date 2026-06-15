# OpenTelemetry Operator Demo 演说词

## 版本一：3 分钟短讲

大家好，今天我想用这一个页面，快速讲清楚 **OpenTelemetry Operator 到底是什么**。

很多人第一次接触 OpenTelemetry 的时候，会先看到 Collector，于是会以为观测系统的重点就是“把 Collector 跑起来”。
但当系统真的进入 Kubernetes、进入多团队、多环境、多服务的场景之后，问题就不再只是“Collector 能不能跑”，而是：
**谁来统一部署它？谁来统一升级它？谁来统一把探针接进应用里？**

这时候，OpenTelemetry Operator 就出现了。

你可以把它理解成：
**它不是干活的数据采集员，它是观测系统的控制面。**
Collector 负责接收、处理、转发数据；而 Operator 负责管理整套观测接入过程。

大家先看页面第一张大图。
左边是业务应用和自动注入配置，
中间是 Operator，
下面是 Collector，
右边是 Jaeger、Tempo、Prometheus 这些后端。

整个链路其实非常清楚：
应用产生日志、指标和链路；
Operator 根据你的声明去创建和维护 Collector；
如果需要，它还会在 Pod 创建的时候自动把探针注入进去；
最后数据再由 Collector 发往后端系统。

接下来第二张图，是 **Operator 管理的四组核心资源**。

第一组是 **OpenTelemetryCollector**。
这个最核心。你可以用它声明 Collector 应该跑成 Deployment、DaemonSet、StatefulSet 还是 Sidecar，也可以声明 pipeline 怎么配置。

第二组是 **Instrumentation**。
这个资源决定自动注入怎么做，比如用什么语言探针、发往哪个 exporter、采样策略是什么。

第三组是 **TargetAllocator**。
这个主要面向 Prometheus 抓取类场景，它可以把采集目标更均衡地分给多个 Collector。

第四组是 **OpAMPBridge**。
它的作用是把集群里的 Collector 和远端 OpAMP 管理端连接起来，适合更大规模、更集中式的管理。

再往下看，我们会发现 Operator 的实现原理其实就两件事。

第一件事，是 **reconcile loop**。
它会不停地监听资源变化，读取你的 spec，计算应该生成哪些 Kubernetes 对象，比如 Deployment、Service、ConfigMap，然后持续校正，让真实状态回到你期望的状态。

第二件事，是 **admission webhook**。
自动注入不是改你的应用代码，而是在 Pod 创建的时候拦截请求，改写 Pod 模板，把需要的环境变量、volume、initContainer，甚至 sidecar 注入进去。所以应用一启动，就已经“带着探针”了。

所以总结一句话：
**OpenTelemetry Operator 的价值，不是让 OpenTelemetry 多一个组件，而是把“观测接入”这件事，从手工活变成平台能力。**

如果你的环境是 Kubernetes，而且你的服务数量正在增长，那它的价值会非常明显。

谢谢大家。

---

## 版本二：6～8 分钟完整版

大家好，今天我想介绍的是 **OpenTelemetry Operator**。

很多团队在做可观测性建设的时候，第一步通常是先把 OpenTelemetry Collector 跑起来。
这一步不难，真正难的是后面：
- 怎么让不同团队用同样的方式接入？
- 怎么统一升级？
- 怎么减少重复 YAML？
- 怎么让探针自动进入应用，而不是每个服务自己改配置？

所以今天这页内容，核心是想说明：
**OpenTelemetry Operator 解决的不是“如何采数据”，而是“如何把采数据这件事平台化、自动化”。**

### 第一部分：先看整体架构

请先看第一页的大图。

这里其实分成四层：
- 左边是应用工作负载，也就是业务 Pod
- 左下是 Instrumentation，也就是自动注入配置
- 中间是 Operator，本质上是控制面
- 下方是 OpenTelemetry Collector，也就是数据平面
- 右边是最终的可观测性后端，例如 Jaeger、Tempo、Prometheus 或其他 APM

所以可以这么理解：
**应用负责产生日志、指标和链路；Collector 负责处理这些数据；Operator 负责把这套东西自动部署好、维护好、注入好。**

也就是说，Operator 不是替代 Collector，而是管理 Collector，管理接入流程。

### 第二部分：Operator 管理哪些资源

接下来，请看资源组大图。

这里我建议大家只先记住四个名字。

#### 1. OpenTelemetryCollector
这是最核心的资源。
如果你要告诉 Kubernetes：“我要一个 Collector，跑两个副本，用 deployment 模式，开放 OTLP 端口，并把数据发往某个后端。”
那你写的不是 Deployment，而是 **OpenTelemetryCollector**。

也就是说，这个资源代表的是“更高层的意图”。
你写的是目标，Operator 帮你翻译成真正的 Deployment、Service、ConfigMap 等资源。

#### 2. Instrumentation
第二个是 **Instrumentation**。
这个资源主要管自动注入。
比如：
- 这套 Java 应用要不要注入？
- 用哪个 exporter 地址？
- 用哪种采样方式？
- 要不要补充资源属性？

这些都可以通过 Instrumentation 来声明。

#### 3. TargetAllocator
第三个是 **TargetAllocator**。
这个资源更偏 Prometheus 抓取场景。
如果你有多个 Collector，都需要从不同目标抓取指标，那么目标怎么分、如何避免重复抓取、如何做更均衡的分配，就由它来解决。

#### 4. OpAMPBridge
第四个是 **OpAMPBridge**。
它适合更大规模的场景。
简单说，就是把集群内的 Collector 和远端管理端桥接起来，方便做集中式控制。

所以这四个资源分别对应四件事：
- Collector 怎么跑
- 探针怎么注入
- targets 怎么分
- 远端怎么管

### 第三部分：组件怎么分工

接着看组件说明。

其实组件并不多，但是分工很清楚。

#### Controller Manager
它会 watch 这些 CRD 的变化。
一旦你创建、修改、删除资源，它就会开始新一轮协调，也就是 reconcile。

#### Admission Webhook
这个负责自动注入。
当一个 Pod 或 Deployment 被创建时，Webhook 可以拦截这个请求，然后改写 Pod 模板。
所以它不是在运行后再补东西，而是在“创建那一刻”就已经改好了。

#### OpenTelemetry Collector
这个大家最熟悉。
它是真正干活的那一个。
负责接收、处理、转发 traces、metrics、logs。

所以最重要的一句话是：
**Operator 是控制面，Collector 是数据面。**

### 第四部分：实现原理

实现原理我建议只记住两条线。

#### 第一条线：reconcile loop
请看 reconcile 大图。

这个闭环就是 Operator 的核心逻辑。
它大概分成四步：
1. 监听变化
2. 读取 spec
3. 生成底层资源
4. 持续校正状态

为什么要“持续校正”？
因为 Kubernetes 里真实世界是会漂移的。
Pod 可能挂掉，配置可能变化，版本可能升级。
Operator 的价值就在这里：
它不是一次性生成 YAML 就结束，而是会一直盯着现实状态，让它尽量回到你声明的目标状态。

#### 第二条线：admission webhook
再看自动注入大图。

自动注入的关键不在代码，而在 Pod 创建流程。
流程是这样的：
- 你先提交一个 Deployment
- 上面带着某个 instrumentation 注解
- Webhook 拦截这个请求
- 它去找对应的 Instrumentation 配置
- 然后改写 Pod 模板，加上环境变量、volume、initContainer，必要时也可能加 sidecar
- 最后这个被改写过的 Pod 才真正创建出来

所以你看到的最终效果是：
**应用一启动，就已经带着 OpenTelemetry 探针了。**

这也是为什么它非常适合平台化接入。
因为业务团队不需要每次自己手工改一堆启动参数。

### 第五部分：为什么这件事很重要

如果只有一两个服务，手工写 YAML 也能用。
但一旦到了几十个、几百个服务，手工方式会迅速变得脆弱：
- 配置不一致
- 升级容易漏
- 接入方式五花八门
- 自动注入难以统一

而 Operator 的价值，就是把这些本来分散在人身上的动作，收拢到平台控制面。

所以最后我想用一句话收尾：

**OpenTelemetry Operator 的真正价值，不是多了一个组件，而是让“观测接入”这件事，变成一个可重复、可维护、可规模化的基础设施能力。**

如果你的环境是 Kubernetes，而且你的服务数量、环境复杂度、团队协作复杂度都在上升，那么 Operator 基本就不是“可选项”，而会越来越接近“标准答案”。

谢谢大家。

---

## 一页一页讲解提示词（讲页面时可看）

### 第 1 屏：总览
- 它不是 Collector
- 它是观测接入控制面
- 应用、Operator、Collector、后端四层关系

### 第 2 屏：4 组资源
- Collector：怎么跑
- Instrumentation：怎么注入
- TargetAllocator：targets 怎么分
- OpAMPBridge：远端怎么管

### 第 3 屏：组件分工
- Controller Manager：协调
- Webhook：注入
- Collector：干活

### 第 4 屏：Reconcile
- 监听
- 计算
- 生成
- 校正

### 第 5 屏：自动注入
- 注解触发
- Webhook 改写 Pod
- 应用启动时已带探针
