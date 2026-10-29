# 架构与选型：先闭环，再扩展

## 1. 范围与前提

本方案希望回答：某个工作负载在指定时间发生了什么、日志来自哪个容器、当时资源和对象状态如何，以及观测链路本身是否健康。它不能仅凭时间上的相关性证明因果关系。

开始前先盘点已有日志入口、Prometheus 抓取任务、Grafana 数据源、身份系统、保留策略和运维责任人。若已有平台满足要求，补齐采集与查询契约即可；不要因画出一张新架构图就重复部署整套后端。下文各组件表示逻辑职责，不要求每项新增一个实例。

Kubernetes 提供容器日志访问能力，但不会替应用提供独立于节点与 Pod 生命周期的集群日志存储。集中式日志需要单独的后端；节点代理、sidecar 是不同采集路径，不应混为一种保证。[Kubernetes 日志架构](https://kubernetes.io/docs/concepts/cluster-administration/logging/)

## 2. 参考数据流

```text
应用 stdout/stderr ─→ 运行时日志 ─→ 选定的节点/API 采集路径 ─┐
应用文件日志 ──────→ LoongCollector 候选采集路径 ─────────┤
可选 Kubernetes Events ─→ 独立事件采集与标记 ──────────┤→ Loki → Grafana
kubelet / cAdvisor ───────────────────────────────────┐
kube-state-metrics ──────────────────────────────────┤→ Prometheus → Grafana
采集器及后端自身指标 ─────────────────────────────────┘
```

图中的箭头表示职责边界，不是可直接应用的部署配置。文件路径、运行时发现方式、插件可用性、鉴权和缓冲均须按版本验证。

- **Loki**：集中保存和查询日志；不要把它当作 Pod 资源指标库。
- **Prometheus**：抓取并保存数值时间序列，支持规则计算和告警；不是全文日志存储。[Prometheus 概览](https://prometheus.io/docs/introduction/overview/)
- **Grafana**：通过数据源展示和关联信号，不因接入界面就自动具备数据采集、历史保留或租户隔离。[Loki 数据源](https://grafana.com/docs/grafana/latest/datasources/loki/)
- **Events**：补充调度、拉镜像等对象事件线索，可转换为日志保存；不是请求跨度，也不是完整审计记录。[Alloy Events 组件](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes_events/)

## 3. 采集器选择不是品牌排名

以下为选型入口，不是横向基准。项目支持某功能，不等于选定发行包包含所需插件，更不代表整个链路具有相同交付语义。

| 候选 | 本方案的定位 | 必须验证的边界 |
| --- | --- | --- |
| LoongCollector | 保留为应用文件采集、容器运行时发现与元数据关联的首选评估对象 | 输入、处理、输出插件是否在目标构建中；运行时适配、文件可见性、检查点和 Loki 输出行为 |
| Alloy | stdout/stderr 接入 Loki 的替代路径；按权限选 API 或文件来源 | API 路径的 kubelet/网络开销与断连恢复；文件路径的节点覆盖、挂载及位置状态 |
| Fluent Bit | 已有标准化部署时可继续评估 Tail 方案 | 文件偏移数据库、长行处理、轮转、多行与缓冲策略 |
| Vector | 需要显式转换流程时的另一候选 | Kubernetes 日志源的交付边界、源与 sink 的确认能力是否相容 |

LoongCollector 的项目定位见[官方仓库](https://github.com/alibaba/loongcollector)。**官方仓库 v3.4.1 的 `flusher_loki` 文档将该插件标为 Alpha**，并列出批量发送和有限重试参数。此标记仅说明该版本文档中的插件状态，不能外推为整个 LoongCollector 的成熟度，也不能证明其他版本完全相同。投产前应对目标发行版逐项核对；本文不提供未验证的复制即用配置。[v3.4.1 Loki 输出插件文档](https://github.com/alibaba/loongcollector/blob/v3.4.1/docs/cn/plugins/flusher/extended/flusher-loki.md)

Alloy 的 `loki.source.kubernetes` 从 Kubernetes API 跟随容器日志，无需节点文件系统访问；官方同时提示其网络和 kubelet CPU 开销高于文件采集。这是权限与资源的取舍，而非“无需权限”。多个实例也不能未经分片设计就同时采同一目标。[Alloy Kubernetes 日志源](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/)

Fluent Bit Tail 可用数据库记录文件与偏移，并对每文件读取缓冲和长行提供配置，适合纳入故障测试，而不是默认认定重启无损。[Tail 文档](https://docs.fluentbit.io/manual/data-pipeline/inputs/tail) Vector 的 Kubernetes 日志源文档标为 best effort、无 acknowledgements；启用 Loki sink 的确认选项不能凭空给源增加确认能力。[Vector 日志源](https://vector.dev/docs/reference/configuration/sources/kubernetes_logs/) / [Loki sink](https://vector.dev/docs/reference/configuration/sinks/loki/)

## 4. 标签、身份与查询契约

建议把低基数的 `cluster`、`namespace`、`workload`、`container`、`source_kind` 作为初始维度，再根据实测调整。Pod 名称和 UID 虽适合定位，却可能快速变化；应优先评估结构化元数据或正文保留，不能一概升级为索引标签。请求 ID、用户 ID、时间戳不作为默认索引标签。标签组合数而非单个标签个数决定流数量。[Loki 基数说明](https://grafana.com/docs/loki/latest/get-started/labels/cardinality/)

约定字段含义：事件时间来自何处、解析失败如何回退、原始时间是否保留、容器重启如何区分，以及同一内容从文件和 stdout 同时出现时哪条路径负责收集。建议从工作负载下钻至 Pod UID、容器、时间窗，再查看关联日志；不能只用易复用的 Pod 名关联跨生命周期数据。

## 5. 安全与落地顺序

读节点目录、访问运行时接口、访问 Kubernetes API 是不同权限面。先选择最少权限路径，再验证所需能力；不要以“日志采集”为理由默认授予特权容器或整个宿主机写权限。

Loki API 本身不负责授权，需由独立鉴权层等机制实施访问控制。租户头或查询标签不自动构成可信身份边界；应防止客户端伪造租户，并分别测试读取和写入越权。[Loki HTTP API](https://grafana.com/docs/loki/latest/reference/loki-http-api/)

建议按“平台盘点 → 单一来源最小链路 → 完整性与故障验收 → 多节点覆盖 → 看板与告警”推进。只有基础行为可解释，才讨论扩容、更多信号或自动化分析。失败时保留原有通路，记录回退条件；双写验证须明确期限和重复识别方法。

## 6. 保留策略也是架构职责

不要把“已写入 Loki”理解为已配置自动清理。官方文档说明 Compactor 的 retention 默认未启用；启用时需符合目标版本的索引类型、schema 和保留配置要求。若对象存储另有生命周期规则，其期限应长于 Loki 保留期，避免数据被过早删除。查询不可见与物理删除也不是同一时刻；验收应分别检查查询窗口、删除进度和存储占用。[Loki 保留策略](https://grafana.com/docs/loki/latest/operations/storage/retention/)
