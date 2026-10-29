# 公开来源与核验边界

整理日期：2026-10-29。以下均为项目官方文档或官方仓库。本文使用独立表述，不复制整段第三方文档；链接用于支撑紧邻的技术事实，设计建议和验收步骤由本仓库组织。

## 来源目录

| 主题 | 公开一手来源 | 支撑范围 |
| --- | --- | --- |
| Kubernetes 日志 | [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) | 集群日志后端、节点代理与 sidecar、轮转及 `kubectl logs` 边界 |
| 资源监控 | [Tools for Monitoring Resources](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/) | 基础资源管线与完整监控的区别 |
| 节点统计 | [Node metrics data](https://kubernetes.io/docs/reference/instrumentation/node-metrics/) | kubelet 指标来源和端点相关说明 |
| 对象状态 | [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) | 根据 API 对象生成状态指标，不等同资源消耗采样 |
| LoongCollector | [官方仓库](https://github.com/alibaba/loongcollector) | 项目定位、发行与文档入口 |
| Loki 输出插件 | [v3.4.1 flusher-loki](https://github.com/alibaba/loongcollector/blob/v3.4.1/docs/cn/plugins/flusher/extended/flusher-loki.md) | 该版本文档中的 Alpha 标记、批处理与有限重试参数 |
| Alloy 日志源 | [loki.source.kubernetes](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes/) | API 采集、节点访问需求和 kubelet/网络开销取舍 |
| Alloy Events | [loki.source.kubernetes_events](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.kubernetes_events/) | Events 转换为日志、命名空间与读取权限 |
| Fluent Bit | [Tail](https://docs.fluentbit.io/manual/data-pipeline/inputs/tail) | 文件偏移数据库、读取缓冲和长行行为 |
| Vector 输入 | [Kubernetes logs](https://vector.dev/docs/reference/configuration/sources/kubernetes_logs/) | 节点文件读取、元数据、best-effort 与确认边界 |
| Vector 输出 | [Loki sink](https://vector.dev/docs/reference/configuration/sinks/loki/) | sink 确认选项不替代 source 能力 |
| Loki 标签 | [Cardinality](https://grafana.com/docs/loki/latest/get-started/labels/cardinality/) | 高基数风险、低基数标签与结构化元数据 |
| Loki API | [HTTP API](https://grafana.com/docs/loki/latest/reference/loki-http-api/) | 查询参数、条数限制、独立授权职责 |
| 日志保留 | [Log retention](https://grafana.com/docs/loki/latest/operations/storage/retention/) | Compactor 保留开关、索引要求和对象存储生命周期约束 |
| 容器指标定义 | [cAdvisor Prometheus metrics](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md) | CPU、内存、网络等指标名称、单位与配置边界 |
| Pod 状态定义 | [KSM Pod metrics](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/pod-metrics.md) | readiness、重启与终止原因等候选指标 |
| Kubernetes 指标目录 | [Metrics Reference](https://kubernetes.io/docs/reference/instrumentation/metrics/) | kubelet 卷等指标的定义与稳定级别核对入口 |
| 监控部署入口 | [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) | 社区 chart 的组件组合、依赖、CRD 与版本说明 |
| 指标管线 | [Prometheus Overview](https://prometheus.io/docs/introduction/overview/) | 时间序列、抓取、规则与图形查询职责 |
| 展示入口 | [Grafana Loki data source](https://grafana.com/docs/grafana/latest/datasources/loki/) | 日志查询与跨信号关联入口 |

## 使用来源时的限制

- 链接中 `latest`、`main` 及无版本路径是移动文档。核验的是阅读时的公开说明，不是永久版本兼容性承诺；选型时应补充目标发行版资料。
- v3.4.1 插件链接固定到公开 tag，用于限定 Alpha 的陈述范围；插件文档不是完整链路可靠性证明。
- “支持”“可配置”与“已在目标环境验证”是不同命题。本仓库没有实际部署或测试结果。
- 文档可能包含与目标集群不一致的新特性、默认值或示例。本文有意不把移动页面中的最新版本号和默认值写成环境事实。
- 若官方资料无法确认行为，保留未知项并设计测试；不能用营销定位、社区经验或组件名称替代证据。

后续补充实验时，应把公开资料事实与实验观察分开，附上合成数据、版本和可复现步骤。来源更新不应悄悄改写旧实验结论。
