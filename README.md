# Engineering Notes

面向 Kubernetes 普通业务容器的公开工程参考：把应用日志、Pod 资源与对象状态整理成可查询、可验收的观测链路。

**先给结论：复用已有平台；日志用 Loki、指标用 Prometheus、界面用 Grafana。保留 LoongCollector 作为应用文件与容器发现的候选方案，stdout/stderr 可选择 Alloy。先证明采集完整性与权限边界，再扩展功能。** 

## 从哪里开始

| 你要做什么 | 阅读入口 |
| --- | --- |
| 确定范围与组件职责 | [架构与选型](docs/observability/architecture.md) |
| 排查文件发现、轮转、丢失或重复 | [采集易错点](docs/observability/collection-pitfalls.md) |
| 分清资源指标、对象状态与 Events | [Pod 监控](docs/observability/pod-monitoring.md) |
| 设计可复核的验证 | [验收清单](docs/observability/acceptance-checklist.md) |
| 理解决策理由与变更条件 | [范围决策](docs/decisions/0001-observability-scope.md) |
| 查证技术事实 | [公开来源](docs/observability/sources.md) |

## 阅读边界

- 本仓库仅依据公开资料独立编写，不包含生产配置、真实业务日志或历史故障复盘。所有样例标识均为合成内容。
- 主题是普通容器日志和 Pod 监控，不是分布式追踪、自动根因定位或安全审计平台。
- 当前没有执行集群部署、故障注入、性能基准或端到端验收；清单中的测试均为**待执行方案**。文档阅读核对不等于运行验证。
- 不承诺 exactly-once、不丢日志或自动 RCA。日志可能因源文件删除、有限缓冲、重试边界、权限和查询截断而不可见。
- 编写日期：2026-10-29。来源中的 `latest` 与 `main` 会变化；落地时须固定组件版本、镜像摘要和文档版本，重新核对兼容性。
- 本仓库尚未指定许可证；引用的第三方资料仍遵循各自条款。

## 如何贡献

优先补充可重复的验证方法、适用条件和公开一手来源。实验报告必须写明版本、合成输入、操作步骤、观测结果与限制，不得把预期写成结果。不要提交凭据、真实租户标识、业务数据、内部链接或未获授权的截图。修改边界和选型时同步更新决策记录；维护提示见 [AGENTS.md](AGENTS.md)。
