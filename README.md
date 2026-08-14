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

## 方法说明

所有结论来自对源码、文档、git 历史的直接阅读与本机实际运行验证，引用处标注了仓库内路径。原始文档（`docs/architecture.md`、`.agents/notes/`、`AGENTS.md` 等）都在上游仓库中，本笔记是解读而非替代。

## License

本仓库笔记内容采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)。所引用的 deepseek-harness 内容版权归其项目所有（MIT License）。
