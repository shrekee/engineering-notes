# Pod 监控：资源使用、对象状态与事件分层

## 1. 三类指标来源不互相替代

| 来源 | 回答的问题 | 不应据此推断 |
| --- | --- | --- |
| kubelet / cAdvisor 等资源统计路径 | 容器和 Pod 消耗多少 CPU、内存等资源 | 业务请求是否成功；某个依赖一定是根因 |
| kube-state-metrics（KSM） | Kubernetes 对象声明和当前状态是什么 | 容器实时 CPU/内存使用量 |
| metrics-server / Metrics API | 为自动扩缩容及 `kubectl top` 提供基础 CPU/内存视图 | 完整历史监控、长期存储、所有资源维度 |

kubelet 的具体统计来源与端点受运行时、特性及版本影响，不要把所有数据都简称为“cAdvisor 指标”。metrics-server 是基础资源管线的一环，不应替代完整监控平台。[Kubernetes 资源监控](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/) / [节点指标](https://kubernetes.io/docs/reference/instrumentation/node-metrics/)

KSM 根据 API 对象生成状态指标；它不是资源消耗采样器，展示值也不必与经过显示逻辑处理的 `kubectl` 文本完全相同。[KSM 官方说明](https://github.com/kubernetes/kube-state-metrics)

## 2. 最小可用视图

建议先形成四张逻辑视图，复用现有看板而不是强制新建四个 dashboard：

1. **资源趋势**：CPU 使用、内存使用与 requests/limits 并列显示，写清单位和分母。CPU 核数与百分比不可混用；未配置 limit 的容器不能用零作为分母。
2. **对象状态**：Pod 调度、就绪、容器等待/终止、重启变化、控制器期望与实际副本。明确数据时间和对象身份，避免同名 Pod 重建后曲线误接。
3. **关联线索**：同一对象与时间窗内的日志、Events 和变更信息。时间对齐只是线索，不是因果判定。
4. **观测链路健康**：抓取失败、样本新鲜度、目标覆盖、采集器重试/丢弃/队列水位及后端写入错误。没有数据不应显示为资源用量为零。

指标名称与稳定级别应按目标 Kubernetes、KSM 和采集配置核对，不直接把网上的表达式视为通用模板。检查重复抓取、聚合范围和容器身份，防止相同样本被重复累加。工作负载级聚合要保留到 Pod/容器的排查入口。

### 建议纳入的监控维度

| 维度 | 观察内容与解释边界 |
| --- | --- |
| CPU | 使用核数与 requests/limits；条件允许时查看 CFS throttling。限流时间或次数不等同业务延迟，需结合负载解释。 |
| 内存 | working set、RSS 与 limit 分开显示；两者不是可互换指标，也不等同应用堆内存。 |
| 生命周期 | readiness、重启增量、等待/终止原因；只有相应状态证据才能报告 OOMKilled，不能从内存曲线或重启次数直接猜测。 |
| 网络 | 接收/发送速率、错误与丢包；确认网络命名空间和接口聚合，避免重复计数。 |
| 存储 | 按需观察 PVC 已用/可用容量及 inode；是否存在相关统计取决于卷类型、CSI/驱动、运行时和暴露配置。容器文件系统使用量不等同 PVC 使用量。 |

[cAdvisor 指标目录](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md)、[KSM Pod 指标](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/pod-metrics.md)与 [Kubernetes 指标参考](https://kubernetes.io/docs/reference/instrumentation/metrics/)用于核对定义。上述是候选维度，不保证每个集群都暴露相同指标；缺少的项目标明“不支持/未采集”，不要用零填充。

### 可复用的开源部署入口

若没有现成监控平台，可评估 Prometheus 社区的 [kube-prometheus-stack Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)，它组合 Prometheus Operator、Prometheus、Alertmanager、Grafana，以及 KSM、node-exporter 等组件和预置规则/看板。按目标 chart 的 values 选择或禁用组件，已有平台优先复用，避免重复抓取和重复告警。

这不是零成本安装包：评审 CRD 生命周期、集群 RBAC、节点访问、持久化存储和 CPU/内存预算；预置看板也需验证指标与标签是否匹配。node-exporter 补充节点主机维度，不代替 Pod 或应用观测。本文未执行该 chart 的安装和升级验证。

## 3. 告警应引导下一步行动

以下是待设计和验证的告警类别，不含已校准阈值：

- 工作负载持续未就绪或副本不足：核对调度、容器状态和近期 Events。
- 重启增加或终止异常：关联对应容器生命周期日志，不能仅凭重启次数判定 OOM。
- 资源持续紧张：结合配额、requests/limits、历史基线与业务负载，而非单点读数。
- 采集或抓取缺失：先检查覆盖、认证、网络和后端健康，避免把监控缺口解释为应用恢复。

每条规则应包含持续时间、缺失数据策略、负责人、排查入口、恢复条件和演练方式。阈值由目标负载与服务目标决定；没有实际基线时不填写看似精确的通用数值。告警风暴、抑制和恢复通知也属于验收范围。

## 4. Events 是补充证据，不是 tracing

Events 记录 Kubernetes 对象相关事件。Alloy 可把 API Events 转成日志并送往 Loki；此路径应独立标记来源，限定命名空间和权限，并确认多实例是否导致重复。[Events 组件](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes_events/)

不要依赖对象 API 永久保存 Events，也不要将事件次数直接当成独立故障次数。需验证更新、重复、重新连接和源端保留边界；采集启动前已经不存在的事件不能靠后端恢复。

监控与日志回答“资源、状态和记录如何变化”；分布式 tracing 需要请求上下文传播、跨度采集和相应后端。即使日志含合成请求 ID，也不代表具备端到端追踪。跨服务调用图、性能剖析、自动根因诊断不在本阶段范围；若确有需求，另写决策和验收标准。
