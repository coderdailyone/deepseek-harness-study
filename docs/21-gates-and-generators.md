# 21 · 门禁与生成器机器：124 个脚本如何看守一个仓库

> 研究对象：deepseek-harness @ `47f9438`。依据：`scripts/`（124 个 TS 脚本：~35 个 `verify-*` 门禁、17 个 `gen-*` 生成器，均自带 spec 测试）、`scripts/run-gates.ts`（门禁清单权威）。01 篇讲过门禁的存在，这一篇看机器本身。

## 版图

**生成器（17 个）**：tool-catalog（每个模型面工具的 schema 与描述）、config-catalog（全部插件配置字段）、persistence-catalog（持久化词表）、module-graph、doc-graphs（能力接缝图，即 02 篇引用的那张）、cordis-api / cordis-catalog / client-catalog、scoped-events、third-party-notices、translation-brief……英文侧目录全部由源码生成 + 新鲜度门禁（改了源码不重新生成就红），中文侧走 16 篇的配对流程。

**门禁（~35 个 verify-\*）**：文档类（md-links 死链、md-wrap 一段一行、doc-budgets 字数、doc-refs、mermaid、type-equiv 类型粘贴一致、translation-pairing 双语配对、translation-prompt）、Agent Note 类（format、classification、archived 封存）、包契约类（package-invariants、package-paths、export-jsdoc、node-next-types、built-package-invariants、dsh-package-licenses、vendored-links、runtime-closure、config-source-ownership、cordis-config）、以及门禁的门禁（lint-rule-fingerprint、ci-workflow.spec 检查 CI 工作流本身）。**每个门禁脚本自带 spec 测试**——看守者也被看守。

## 两个值得单独立传的门禁

### verify-package-readme-model-experience

每个模型可见的包 README 必须携带 `## Model Experience` 节，其下强制三个四级小节：

- `#### What the model sees` — 模型看到什么（prompt 段/schema/上下文注入的原文或规则）
- `#### Token effect` — 这个包花模型多少 token、在哪花
- `#### KV Cache effect` — 对请求前缀缓存的影响（追加式？失效点在哪？）

这是把"**模型也是用户，它的体验要写文档**"制度化成机检契约——业内我尚未见过第二家。全系列引用过的那些精彩段落（子代理的 delegation-scope 声明、compaction 检查点前言、结算通知的措辞）都是这个门禁逼出来的正式文档。模型无关的包必须进**带理由的豁免审计表**——"缺席的节不可能被误认为忘写的文档"。

### verify-package-readme-limitations

每个包 README 必须携带 `## Known Limitations and Deferred Work` 节且至少一条真 bullet；审计过的无局限包只有一个（`util/brand`，理由："纯类型的名义品牌原语，无运行时行为无延期工作"）。换句话说：**制度强制每个包承认至少一条局限**。全系列反复引用的那些诚实清单（subagent 的十条、compaction 的五条、jobs 的三条）不是作者美德，是门禁产出。诚实被工程化了。

## 机器的三个设计特征

**1. 生成器与门禁配对成"新鲜度闭环"。** 生成的目录不是一次性产物：源码变更 → 对应 catalog 的门禁红 → 必须重新生成 → 中文对侧失配 → 配对门禁红 → 同 PR 更新。agent 的"地图"（60+ 服务表、工具目录、事件生产者-消费者表）因此永不过期——这是 02 篇"架构与 agent 开发互相成就"的机械基础。

**2. 豁免必须带理由入册。** 覆盖率豁免（coverage-exempt）、Model Experience 豁免、无局限豁免、pre-format Agent Note 的 alternatives 豁免——每一类例外都是**代码里的审计表**（路径 → 一句话理由），评审可见、有据可查。例外不消失在配置暗处。

**3. 门禁不是 lint 的堆砌，是制度的可执行投影。** 对照关系一一成立：Agent Note 制度 ↔ format/classification/archived 三门禁；文档分层制度 ↔ budgets/refs/type-equiv；双语制度 ↔ pairing/prompt；"模型即用户" ↔ model-experience；诚实文化 ↔ limitations。01 篇说"质量不靠人盯靠门禁"，这一篇的补充是：**门禁本身是制度文本的编译产物**，每个 verify 脚本头部都链接着拥有其 rationale 的 Agent Note。

## 可迁移性

规模可以缩，结构不必变：哪怕一个五人项目，"生成器+新鲜度门禁"（README 里的 CLI 用法表从 --help 生成）、"豁免带理由入册"（一个 JSON 表）、"强制局限节"（CI 里十行脚本）都是当天可落地的。124 个脚本是终态不是起点——git 历史显示它们是 65 天里随制度逐条长出来的。
