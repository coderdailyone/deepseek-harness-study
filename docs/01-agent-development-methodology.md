# 01 · 他们怎么用 agent 造出这个项目

> 研究对象：deepseek-harness @ `47f9438`（2026-08-13）。所有统计与引用基于该 commit 的 git 历史与仓库文件。

## 证据：这是一个 agent-first 项目

**从第一天就是。** 首个 commit（2026-06-10）的信息是 "Initialize repo with README, AGENTS.md, and CLAUDE.md symlink"——仓库出生时就带着给 agent 的指令文件。

**速度只有 agent 大规模写码才解释得通。** 到 2026-08-13 约 65 天：12,293 个 commit、2,520+ 个 PR、约 5,610 个 merge，日均约 190 commit / 39 PR。

**分支前缀直接暴露工具链。** 对全部 merge 的来源分支统计：

| 前缀 | 数量 | 含义 |
|---|---|---|
| `worktree/` | 210 | agent 在 git worktree 隔离中工作 |
| `codex/` | 203 | OpenAI Codex 产出的 PR |
| `agent/` | 15 | 通用 agent 分支 |
| `claude/` | 3 | Claude Code 产出的 PR |
| `feat/` `fix/` 等 | ~200 | 常规前缀（其中也可能有 agent 产出） |

还有 `docs/readme-human-polish-3` 这类分支名——主体 agent 写，人来抛光。

**commit 作者全是真人署名**（无 Co-Authored-By 机器人 trailer）：人对每个 PR 负责，agent 干活。这是一个明确的责任模型选择。

**自举闭环。** dsh 自己内置 `hooks-claude-code`/`hooks-codex` 桥接包（兼容两家的 hook 协议），依赖里直接带着 `@openai/codex` 的二进制，还有 `self-modification` 包让 agent 检视并修改自己的运行时插件（`pnpm run demo:cordis`）。他们用别家 agent 开发自己的 agent harness，同时让自己的 harness 能跑同类工作流。

## 制度：四层结构

### 第一层：AGENTS.md = agent 的常驻军规

根目录 `AGENTS.md`（`CLAUDE.md` 是它的 symlink，root/packages/examples 三处），149 行，有字数预算门禁（≤1,600 词）强制它保持浓缩。内容全是行为规则，例如：

- 非平凡改动必须同 PR 附 Agent Note
- waterfall 监听器必须调 `next()`，否则短路
- 永不默认跑全量测试；选覆盖改动面的最小检查集，CI 拥有穷尽覆盖
- 沙箱挡住命令时用最窄的升权重试，但绝不绕过真实的测试失败
- 跨包边界的不透明 id 必须用 branded type，不许裸 string
- 空 catch 必须写明吞掉了什么、为什么别的东西到不了这里

子树（`packages/`、`docs/`、`.agents/notes/`）各有自己的 AGENTS.md，只写该子树特有的规则，与根文件"一个事实一个家"。

### 第二层：.agents/skills/ = 固化的工作流

11 个 Claude Code SKILL.md 格式的技能文件，把重复工作流固化：

- `dsh-pre-push-checks` — 推送前跑什么检查
- `dsh-code-review` — 按本仓库标准审 PR（先跑 change-scope 报告再读 diff）
- `dsh-prose-standard` / `dsh-doc-standards` — 注释与文档的编辑标准
- `dsh-translate-docs` — 双语文档翻译（仅限用户显式调用）
- `dsh-archive-agent-notes` — 决策档案的归档判断
- **`dsh-trim-cot-leakage`** — 最有意思的一个：专门清理"推理转录腔"

`dsh-trim-cot-leakage` 的存在本身就是"文档主体由 agent 写"的证明。它定义的病症：引用只有创作会话能看到的产物（"(decision N)"、"§N of uncommitted drafts"）、叙述变更而非状态（"used to"、"no longer"）、栈/评审视角（"a later PR in this stack"）、对已离场 reviewer 的辩解。判据一句话：**HEAD 处的读者，不看任何会话记录，能否解析每个引用、核实每个断言？**

### 第三层：.agents/notes/ = 683 篇决策档案（Agent Note）

类似 ADR 但更严格的制度：

- **路径编码两个轴**：生命周期（`proposed/` → `implemented/` 或 `rejected/`，低价值的进冻结的 `archived/`）× 类别（feature/bug-fix/architecture/process/simplification/testing，封闭集合，门禁拒绝其他目录）
- **强制节 "Alternatives considered"**：每篇必须记录真实考虑过的替代方案及其败因——"没有记录打败了谁的决策，会招来重新扯皮（re-litigation）——这正是 Agent Note 要防止的失败"
- **implemented 笔记与现状同步维护**：代码后来移动了文件、改了默认值，同一改动里更新笔记（只改事实不改决策）；禁 spec 腔（"should"、迁移计划、验收清单），门禁直接拒绝这些标题出现在 implemented 笔记里
- **归档即冻结**：永不再编辑、永不作为现状权威引用
- 格式由 `verify-agent-note-format` 机器门控；跨引用必须用相对 markdown 链接（机器可查死链），禁止裸文号

数量分布（本 commit）：implemented 505 篇（architecture 129 / feature 170 / bug-fix 77 / process 69 / simplification 48 / testing 12）、proposed 25、rejected 11、archived 142。

### 第四层：机械门禁替代人工盯梢

这是让 agent 产出可信的关键——质量不靠人盯，靠门禁：

- **per-file 100% 测试覆盖率**（CI gate，`test:coverage`），不可测的行必须写 `/* v8 ignore next -- 为什么不可达 */` 书面自辩
- **跨文件 TypeScript 克隆检测**（`duplication`）
- **`doc-sync` 文档门禁族**：死链检查、字数预算（`verify-doc-budgets`）、Agent Note 格式、双语配对一致性、每段一物理行
- **`type-equiv`**：文档里粘贴的类型声明用 TS parser 与源码断言逐字一致，注册进 manifest，防漂移
- **keyless snapshot 测试**：无 API key 回放录制的完整应用 transcript，模型/用户可见行为的改动必须同 PR 更新快照
- **PR 标签分类法**：一个 `kind/*`、所有涉及的 `area/*`、原生 Issue Type，统一门控

### 反 AI-slop：文档的"slop 清单"

`docs/AGENTS.md` 维护一份猎杀清单，`dsh-doc-standards` skill 按它做审计：

- 同一规则出现在多个家（grep 特征短语，留一个家其余放链接）
- 叙述历史（"previously/now/no longer"、PR 号、commit 号出现在长期文档里）
- 实现状态标注（"implemented!"、"future: ..."——状态会腐烂，仓库布局和 manifest 才是权威）
- 手抄的目录/清单（生成器是权威时）
- 推理转录（逐步实现叙述、显然分支的证明、测试走读）
- 强调通胀（到处加粗 = 没有重点）
- 段落墙（一段塞多条规则加括号旁白）

### Postmortem 制度

与 Agent Note 相对：**Agent Note 记录深思熟虑的决策，postmortem 记录失败**——bug 到了不该到的地方（真实用户、已合并 PR、发布版），有意思的部分是"为什么每层安全网都没拦住"，而非一行修复。触发标准：机制微妙（认真工程师也要重新艰难推导）× 系统性（逃逸原因是测试/工具/惯例的缺口）× 重发现代价高。每篇开头是 30 秒可读完的执行摘要。

## 一句话方法论

**把"对 agent 的要求"全部制度化成可机检的门禁**（覆盖率、格式、文档、链接、字数、翻译一致性），**把"决策记忆"外置成强制的决策档案**（防止 agent 与人重新扯皮已决事项），**把"重复工作流"固化成 skills**，然后放 Codex/Claude Code 在 worktree 里高速产出，人只做 review 与 polish、并对每个合并署名负责。
