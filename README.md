# superagent-playground

dsh-superagent 的试验仓库。给 agent 派活：在本仓库开 Issue 或评论 `@<agent-account> <需求>`。

## 文档与补丁

- [`docs/windows-setup-and-dsh-next-bridge.md`](docs/windows-setup-and-dsh-next-bridge.md) ——
  在 Windows 上自建 DSH Superagent、完成端到端验证（Issue → Ticket → 执行 Session → PR）、
  再把 Superagent 面板桥接进 DSH NEXT 的完整记录；含 21 条踩坑与解决办法、复现清单、脱敏说明。
- [`patches/`](patches/README.md) —— 基于上游 `6337fc8` 的两份补丁：
  `1-windows-adapt.patch`（Windows 服务管理器与路径/控制台/PATH 适配）、
  `2-standalone-embed.patch`（管理面板的独立构建入口，供 DSH NEXT 内嵌）。
- 配套插件：[dsh-superagent-bridge](https://github.com/deepseekv41flash/dsh-superagent-bridge)（服务侧）、
  [dsh-superagent-panel](https://github.com/deepseekv41flash/dsh-superagent-panel)（DSH NEXT 侧）。

由 dsh-superagent 试验仓库维护
