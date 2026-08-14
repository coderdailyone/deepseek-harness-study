# 17 · 自动化接入面：ACP、TypeScript/Python SDK、headless

> 研究对象：deepseek-harness @ `47f9438`。依据：`packages/acp/acp/README.md`、`packages/sdk/README.md`、`python/README.md`、相关 Agent Note。

dsh 有四条程序化接入路径，各自的定位边界划得很清楚：

| 路径 | 形态 | 定位 |
|---|---|---|
| `dsh --profile headless "task"` | 一次性进程 | 单任务 runner，无服务器 |
| ACP（`demo:acp`） | JSON-RPC stdio 服务器 | **automation-only** 的 Agent Client Protocol；仓库内主要客户是 `subagent-acp`（06 篇） |
| TypeScript SDK（`packages/sdk/`） | protocol/client/server 三包 | 从另一个进程驱动 harness 运行时 |
| Python SDK（`python/`） | `deepseek-harness-sdk` + 捆绑运行时 | Python 生态接入，ndjson JSON-RPC over stdio |

## ACP：一个"诚实的传输适配器"

自我定位第一句就是排除法："**传输适配器，不是 UI 集成也不是能力接缝**"——不暴露编辑器导航、transcript 回放、命令、模式、配置选择器、elicitation、reasoning、计划、标题、工具展示。协议面收到最小：initialize 只宣告 baseline 提示能力（无图像/音频/嵌入上下文），不宣告 session/editor/terminal/filesystem/MCP 能力。

三个设计判断值得记：

**1. 只输出已提交的消息。** `session/update` 每个已提交 `assistant/message` 的非空文本块发一个 chunk——原始 delta 与非消息事件一概不发。权衡写得明白："**刻意用逐 token 延迟换干净的自动化结果**——未提交的 provider chunk 与重试尝试不可能泄漏部分文本"（09 篇的重试机制在这里兑现了它的可见性承诺：失败尝试的残渣绝不会流进自动化客户端）。

**2. stopReason 不装懂。** ACP 要求每个 prompt 响应带 stopReason，但桥"不声称 prompt 特定的 turn 结局"——已提交消息横跨整段活动流，转向和注入的工作都可能在 idle 前掺入，所以 token 上限的 turn 结局不上升为 prompt 级 stop reason（照报 `end_turn`）。不知道的事不编。

**3. teardown 的精确范围。** 客户端断连与 Cordis 销毁共享一个 memoized teardown：先拒新会话新 prompt、结算未决 prompt、**只在本连接确切拥有的 Agent 之下**排干 continuable 后代、并行销毁句柄并等全部结果。"共享同一 Context 的其他前端保留它们的 continuable 森林与准入——ACP 插件重载不留孤儿 agent，也不误伤邻居。"（03/06 篇的生命周期纪律在传输层的投影。）

## SDK：产品边界的减法

TypeScript SDK 三包分层（wire 协议 / 客户端 API / stdio 服务器），且有一条明确的产品边界决策（`2026-08-11-remove-sdk-project-toolchain.md`）："**调用者提供运行时可执行文件和它的 cordis.yml；本组不创建、配置、构建或启动开发者项目**"——SDK 不做脚手架。这类"删掉一整块潜在功能面"的决定也走 Agent Note 留档（simplification 类），删功能与加功能同等被记录。

Python SDK 的分发结构：`deepseek-harness-sdk`（高层 turns API + 底层 JSON-RPC 客户端）与 `deepseek-harness-runtime-bin`（**捆绑的运行时二进制** + 默认 agent 配置）分开发布——SDK 默认启动匹配版本的捆绑运行时，也可显式选 channel；"客户端选 channel 并提供默认配置；运行时自身永远要求显式配置"。04 篇 pnpm-workspace 里那个 `python/sdk-runtime` 成员的注释在此对上了："单可执行构建的部署根：纯依赖清单，其闭包就是 exe 捆绑什么、Python 运行时分发什么"。

## 观感

四条接入面的共同点是**负空间的清晰**：ACP 列出十种它不做的东西、SDK 声明它不碰开发者项目、headless 是"没有服务器"的 runner。每个面只认领自己能兑现的语义（stopReason 的处理是缩影），与 13 篇 B3 原则（"明说不管什么，和管什么一样重要"）同构。对照很多框架"每个入口都想全能"的倾向，这种按接入场景裁剪能力面的做法明显更可维护。
