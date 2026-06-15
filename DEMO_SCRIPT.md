# OpenTelemetry Operator 10 分钟内演讲稿

## 版本：8～10 分钟

大家好，今天我想讲清楚一件事：

**OpenTelemetry Operator 在整套观测架构里，到底站在哪个位置？**

很多人第一次看 OpenTelemetry，会先看到 Collector。
于是很容易形成一个印象：
好像只要把 Collector 跑起来，观测体系就差不多了。

但真正进入 Kubernetes 之后，你会很快发现，难点不是 Collector 能不能跑，
而是下面这些事情谁来统一完成：

- 谁来把 Collector 以一致方式部署到各个集群？
- 谁来在不同服务里统一接入探针？
- 谁来在配置变化之后持续校正状态？
- 谁来让这件事从“手工接入”变成“平台能力”？

这时候，OpenTelemetry Operator 才真正出现它的意义。

所以今天这 10 分钟，我只讲三件事：

1. Operator 在整个架构中的位置
2. 数据和请求是怎么流动的
3. 为什么它本质上是控制面，而不是数据面

---

## 第一部分：先看位置

请先看第一页大图。

这张图如果只记一句话，我希望大家记住的是：

**OpenTelemetry Operator 站在中间，它不直接承载业务请求，也不直接充当最终后端，它负责控制整条观测接入链路。**

左边是应用。
也就是你的业务服务、API、Job、Worker。
这些应用每天都在处理真正的业务请求，比如 HTTP、gRPC、数据库调用。

这些应用同时会产生三类观测信号：

- traces，也就是链路追踪
- metrics，也就是指标
- logs，也就是日志

但这些信号刚产生的时候，通常并不会自己自动、稳定、统一地进入你的观测平台。

中间这个位置，就是 Operator。

大家注意，Operator 不负责“消费用户流量”。
真正的业务流量依然在应用之间流动。
Operator 也不负责做最终展示。
最终展示发生在右边这些观测后端里，比如 Jaeger、Tempo、Prometheus，或者其他 APM 系统。

Operator 站在中间，做的是控制动作：

- watch 资源变化
- reconcile 期望状态和真实状态
- mutate 创建请求
- 触发自动注入
- 管理 Collector 的部署和配置

所以如果用 Kubernetes 的语言来说：

**Operator 是控制面。**

而下方这个 Collector，才是数据面。

Collector 真正接收 traces、metrics、logs，做处理、聚合、批量发送、转发到后端。

也就是说，这里有一个非常重要的分工：

- 应用：产生日志、指标、链路
- Operator：决定如何接入、如何部署、如何维护
- Collector：真正搬运和处理观测数据
- 后端：展示、查询、告警、分析

这就是整套架构中最重要的一层定位。

---

## 第二部分：数据和请求是怎么流动的

接下来还是看这张大图，但换一个角度。

这一次不要只看组件，要看“流动”。

### 1. 业务请求的流动

最左边是业务服务。
用户请求首先进入应用。
应用之间会互相调用，比如 Service A 调 Service B，Service B 再访问数据库或其他服务。

这些业务请求本身，不经过 Operator。

这一点非常重要。
因为很多人会误以为 Operator 像一个代理，或者像网关一样夹在业务流量里。
其实不是。

**Operator 不在主业务流量路径上。**

它不转发用户请求，不处理 HTTP 包，不承接线上调用。

### 2. 观测数据的流动

虽然业务请求不经过 Operator，
但是业务请求会触发观测数据的产生。

比如一次 HTTP 请求经过应用后，会产生：

- 一个 trace span
- 几个 latency / error / throughput 指标
- 一些结构化日志

这些数据不会直接飞到 Jaeger 或 Prometheus。
通常会先进入 Collector。

Collector 在这里承担的是统一的数据入口和处理节点。
它负责：

- 接收 OTLP 数据
- 做 processors 处理
- 批量发送
- 转换协议
- 再导出到后端

所以从流向上看是：

**应用 → Collector → 后端**

### 3. 控制请求的流动

真正体现 Operator 价值的，是另外一条线：控制请求。

当你在 Kubernetes 里声明一个 OpenTelemetryCollector，
或者给某个工作负载加上 instrumentation 注解，
或者更新 Instrumentation 配置时，
Operator 会感知这些变化。

它看到的是“声明”，不是“数据包”。

然后它会根据这些声明做动作：

- 创建或更新 Collector 的 Deployment、Service、ConfigMap
- 在 Pod 创建阶段调用 admission webhook
- 改写 Pod 模板，注入环境变量、volume、initContainer，必要时也可以加 sidecar

这就是为什么图里 Instrumentation 在上方往下打到 Operator，
Operator 再往下影响 Collector，或者往左改写业务应用的 Pod 生命周期。

所以这张图里其实有三条完全不同的流：

- 左边应用之间的业务请求流
- 应用到 Collector 再到后端的观测数据流
- 围绕 Operator 展开的控制请求流

而 Operator 管的是第三条。

---

## 第三部分：为什么它是控制面

如果要再往前讲一步，
我们就要回答：为什么 Kubernetes 世界里会需要这样一个 Operator？

答案很简单。

因为只靠手工接入，不可能支撑大规模、长期、多人协作的观测建设。

一开始服务少的时候，手工做还可以：

- 手工部署一个 Collector
- 手工给服务加环境变量
- 手工改启动参数
- 手工配 exporter 地址

但服务一多，就会出问题：

- 每个团队接入方式不一样
- 每个环境的配置不一致
- 升级 collector 或探针会很痛苦
- 新服务上线时容易漏接入
- 多语言场景下更难统一

所以平台团队会想要一个机制：

**我不要每次都手工改，我要声明目标，然后系统自动完成。**

这就是 Operator 模式的价值。

你不再直接维护一堆底层 Deployment 和杂散脚本，
而是写更高层的意图。

比如：

- 我要一个 Collector，用 deployment 模式，跑 2 个副本
- 我要 Java 服务自动注入探针
- 我要 OTLP 数据统一发到某个 collector endpoint

你写完这些声明之后，剩下的协调工作由 Operator 负责。

这就是 Kubernetes 里很典型的控制面思想：

**用户声明意图，控制器持续把现实拉向这个意图。**

---

## 第四部分：第二张图——它到底反复做什么

接下来请看第二张图。

这张图把 Operator 的动作压缩成四个词：

- 监听
- 改写
- 生成
- 校正

这四个词，就是它反复在做的事情。

### 1. 监听

Operator 首先 watch 集群里的对象变化。

比如：

- CRD 被新建了
- Instrumentation 改了
- 某个 Collector spec 更新了
- 某个相关对象状态漂移了

监听是入口。

### 2. 改写

第二步是改写，也就是 mutate。

最典型的就是 admission webhook。

在 Pod 被真正创建出来之前，
Webhook 会拦截这次创建请求，
检查有没有命中自动注入规则。

如果命中，它会直接改写 Pod 模板。

所以自动注入不是“服务启动后再补救”，
而是在“创建时就改好”。

### 3. 生成

第三步是生成底层资源。

你声明的是 OpenTelemetryCollector，
但真正落地时，集群里需要的是：

- Deployment
- Service
- ConfigMap
- RBAC 之类的对象

这些都由 Operator 来生成或维护。

### 4. 校正

最后一步是校正。

这一点特别关键。

因为 Operator 的价值不只是“创建一次”。
而是：

- 配置变了，它会重新协调
- Pod 丢了，它会继续拉回
- 期望状态和实际状态不一致，它会继续修正

所以它不是一次性脚本，
而是一个持续运行的协调器。

这就是我们常说的 reconcile loop。

一句话总结这张图：

**你声明目标，Operator 就不停地把集群拉回这个目标。**

---

## 第五部分：把两张图合在一起理解

现在把前后两张图放在一起，就很容易了。

第一张图回答的是：

**它站在哪。**

第二张图回答的是：

**它一直在干什么。**

于是我们就得到一个非常清晰的理解框架：

- Operator 不在主业务请求路径上
- Operator 也不是最终的观测后端
- Operator 管的是观测接入过程本身
- 它通过 webhook 和 reconcile，把“接入动作”自动化
- 它通过 Collector，把“数据出口”标准化

所以真正的结论不是“这里又多了一个组件”，
而是：

**OpenTelemetry Operator 把观测接入这件事，从手工活，变成了平台能力。**

---

## 结束语

最后我想用三句话收尾。

第一句：
**业务请求不经过 Operator，但观测接入离不开 Operator。**

第二句：
**Collector 搬运数据，Operator 管理接入。**

第三句：
**当服务规模越来越大时，Operator 的价值不是可选增强，而是平台化的基础设施。**

如果大家今天只带走一个印象，
那我希望是这个：

**OpenTelemetry Operator 的位置，不在最前面，也不在最后面；它站在整条观测链路的中间，负责让这条链路持续、有序、自动地运转起来。**

谢谢大家。

---

## 每一屏的极简提示词

### 第 1 屏：位置
- 左边应用
- 中间 Operator
- 下方 Collector
- 右边后端
- 三条流：业务流、观测流、控制流

### 第 2 屏：控制循环
- 监听
- 改写
- 生成
- 校正
- 一直循环

### 可选 Q&A 一句话回答
- 为什么不是直接用 Collector？
  - 因为 Collector 解决的是数据处理，Operator 解决的是平台化接入与持续维护。
- Operator 会不会经过业务流量？
  - 不会，它不在主业务请求路径上。
- 自动注入发生在什么时候？
  - 在 Pod 创建时，由 admission webhook 改写模板。
