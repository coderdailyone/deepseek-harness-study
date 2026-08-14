# DeepSeek Harness (dsh) 研究笔记

对 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 的架构与工程方法研究。

**研究对象版本**：commit `47f943859bef60e4160492346772ded9b24f765a`（2026-08-13，0.1.0-rc.5，developer preview）。dsh 迭代很快，文中结论以该 commit 为准。

## 这是什么

DeepSeek Harness（`dsh`）是 DeepSeek 开源的 agent harness，基于 Cordis 插件框架，"一切皆插件"。这个仓库不是对它的介绍，而是两条线的深挖笔记：

1. **它是怎么被造出来的** —— 仓库里保存了完整的"用 AI agent 大规模开发软件"的制度证据：65 天 12,293 个 commit、683 篇 Agent Note 决策档案、11 个固化工作流的 skill、一整套替代人工盯梢的机械门禁。这可能是目前公开可考的最完整的 agent-first 工程方法论样本。
2. **它的架构是怎么设计的** —— 三个奠基决策（事件溯源会话、微内核事件分类学、能力接缝三角色）+ 一个组合系统（profile/bundle/patch），每一层都能在代码与决策档案里指认。

## 目录

| 文档 | 内容 |
|---|---|
| [01 · Agent 开发方法论](docs/01-agent-development-methodology.md) | 他们怎么用 Codex/Claude Code 造出这个项目：四层制度、门禁清单、统计证据 |
| [02 · 架构设计解读](docs/02-architecture.md) | Cordis 地基、三大支柱决策、组合系统，以及架构与 agent 开发方法的互相成就 |
| [03 · 核心代码阅读笔记](docs/03-core-loop-notes.md) | agent-loop 三文件精读：生命周期工厂、ReAct 循环、并行工具调度器、日志投影收件箱 |
| [04 · 源码跑通记录](docs/04-running-from-source.md) | 从 clone 到 headless/Web UI 跑通的完整过程与四个坑的解法（含无 root 环境的 node-pty 绕法） |
| [05 · 后台任务系统](docs/05-background-jobs.md) | run_in_background 与统一 jobs 运行时、完成通知唤醒空闲 agent 的投递设计（含对 Codex/Claude Code/Kimi 同类设计的官方对比） |
| [06 · Subagent 接缝](docs/06-subagent-seam.md) | 六 provider 一接口：one-shot/continuable 双生命周期、委托策略钉死、结算投递顺序规则、真 Codex/Claude Code 产品后端的契约与证据分层 |
| [07 · Compaction 与 Surface](docs/07-compaction-and-surface.md) | 不可变日志上的历史手术：surfaceOp 投影机制、免模型剪枝、KV 缓存复用的摘要调用、溢出恢复 |
| [08 · 沙箱/审批/权限](docs/08-sandbox-approval-permission.md) | 内核裁决的拒绝信号、一次性升权重试、自带 ~300 行 C 的 Landlock 启动器、会话日志即策略存储 |
| [09 · LLM 接缝](docs/09-llm-seam.md) | 双真适配器验证中立词表、失败事实与重试策略分离、单层拥有可见尝试、User-Agent 归因的标准调查 |
| [10 · 测试策略](docs/10-testing-strategy.md) | 五层测试塔："验证世界而非自我报告"、fail-before 纪律、不配给真 API 测试、同 PR snapshot 制度 |
| [11 · Host/Client 架构](docs/11-host-client-architecture.md) | 双编译面的根因、Typert 类型驱动 RPC 网关、SRC 开发回退不降级校验、Chat 节点的可回放纪律 |
| [12 · 缺陷学](docs/12-defect-taxonomy.md) | 七条防御模式规则 + 四篇 postmortem 的执行摘要与它们各自催生的守卫 |
| [13 · 综合](docs/13-lessons-for-harness-builders.md) | 全系列提炼：给 harness 建造者的 20 条可迁移原则（状态/边界/失败/安全/工程过程） |
| [14 · Code Mode 与工作流](docs/14-code-mode-and-workflows.md) | 模型写 TypeScript 调工具：展示与授权分离、敌对对端的 worker 协议、Ralph 固定迭代工作流 |
| [15 · 驾驶层](docs/15-goals-plan-skills-hooks.md) | Goal 的执行时权威认证、Plan Mode 的 KV 缓存约束、Skills 分层注册表、竞品 Hook 协议桥 |
| [16 · 双语文档系统](docs/16-bilingual-docs-system.md) | blob 哈希配对记录、GC 免疫快照 ref、fail-closed 自研 merge driver、结构签名与术语表治理 |

## 方法说明

所有结论来自对源码、文档、git 历史的直接阅读与本机实际运行验证，引用处标注了仓库内路径。原始文档（`docs/architecture.md`、`.agents/notes/`、`AGENTS.md` 等）都在上游仓库中，本笔记是解读而非替代。

## License

本仓库笔记内容采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)。所引用的 deepseek-harness 内容版权归其项目所有（MIT License）。
