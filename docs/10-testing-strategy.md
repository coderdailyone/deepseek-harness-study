# 10 · 测试策略："验证世界，而非自我报告"

> 研究对象：deepseek-harness @ `47f9438`。依据：`docs/testing.md`（49 行的高密度政策文档）、`.agents/notes/implemented/testing/`、postmortem 0001。这一篇与 01 篇（方法论）互为表里：机械门禁管代码质量，这套测试哲学管"agent 写的东西真的能跑"。

## 五层测试塔

| 层 | 命令 | 管什么 |
|---|---|---|
| Unit | `test` | 包内规格；每个注册表配 HMR 安全测试（销毁贡献 fiber、断言清理） |
| **Coverage gate** | `test:coverage` | CI 门禁：`packages/*/*/src` **逐文件 100%** |
| Real-API e2e | `test:e2e` | 带 key 打真 DeepSeek 与各 provider；无 key 自跳 |
| Snapshot | `test:snapshot` | **无 key** 回放录制好的完整应用 transcript，diff 归一化输出与重持久化日志 |
| Web browser snapshot | `test:web` | Chromium 对比回放的浏览器渲染输出；CI 强制只读回放、永不改写期望 |

## 六条值得抄的纪律

**1. "未覆盖的行往往是门禁正确标记的死代码，而不是待补的测试。"** 逐文件 100% 覆盖率的辩护词把因果反过来了：门禁的价值不是逼你写测试，是逼你删死代码或书面自辩（`/* v8 ignore next -- 为什么不可达 */`）。同时诚实声明边界："行覆盖是必要的、永不充分——它证明行跑过，不证明特性如出厂般工作。"

**2. "We are DeepSeek — do not ration real-API tests."**（我们就是 DeepSeek——不要配给真 API 测试。）推理对他们便宜，所以带 key 测试覆盖写文件提示、多轮对话、工具使用、流中取消。最高价值的是 **smoke 测试**：起真实示例、发一条提示、检查世界——专抓"单元测试全绿、产品是坏的"这类 mock 抓不到的缺陷（postmortem 0001 就是实例：`export default` 吞掉了插件的 `inject`，一切单元测试通过，ACP 服务器一连就崩）。自跳过是为了不阻塞无 key 贡献者，"不是成本信号"。

**3. "验证世界，而非自我报告。"** 这是 agent 时代特有的测试智慧，原文值得全文引用：

> An e2e assertion re-runs the command or re-reads the file externally; a keyword probe on the agent's own output lets a cheating agent pass.（e2e 断言重新执行命令或从外部重读文件；对 agent 自己输出做关键词探测，会让作弊的 agent 通过。）

还要断言**未触碰的文件逐字节不变**。当被测对象本身是会"编造成功"的模型时，测试必须绕过被测者的嘴、直接摸世界。

**4. "守卫只有在回归真的会让它变红时才算守卫。"** 配套操作规程：**引入回归、看着变红、再撤销**（introduce the regression, watch red, revert）。文档给了具体例子：无 `inject` 的组合插件，default export 顶掉命名导出时 Loader smoke 依然绿——所以要显式断言 `expect('default' in mod).toBe(false)` 并证明这条断言抓得住。fail-before 是成文纪律，不是个人习惯。

**5. "Mock 只到昂贵或不确定的边界，下游全部真跑。"** LLM 适配器、网络、时钟可以 mock；工具、执行器、组合装配必须真。"手搓的替身证明桥能搬字节，不证明出厂的工具行为如断言"。桥接工具测试用脚本化 mock 模型 + 真 `dsh-bash-local` + 真 `dsh-tool-bash` 跑真 `echo`。

**6. "测真实入口路径。"** 三个层层递进的要求：产品可见插件必须有**真组合测试**（test-only `cordis.yml` 走 Loader + 真应用进程，手工 `ctx.plugin(...)` 不算）；`bin` 必须以纯 node 跑**构建产物** `lib/bin.js`（暴露 tsx 掩盖的 settle 竞态、模块解析、吞掉的加载失败）；跨 bundle 共享的单例模块有专门的构建产物冒烟。"真入口"指发布的工件，不是开发环境的近似。

## Snapshot 制度：模型可见行为的回归锚

每个非平凡的模型/协议/人可见改动，**同 PR** 必须经可运行示例的 snapshot 套件加或改一个无 key 场景——"包测试、e2e 断言、mock 组合、PR 说辞都不能替代装配后的 transcript"。工程细节：一个场景（`text-turn`）钉完整 system prompt/工具 schema 原文，其余场景 token 化——改一处 prompt 只搅动一行 diff。录制（有 key）与回放（无 key）分离，每次 JSONL 与期望输出的 diff 都过评审。

## 为什么这套哲学与 agent 开发互相成就

四个映射：**覆盖率门禁**替代人盯行级质量；**fail-before 纪律**防 agent 写出永远绿的假守卫；**"验证世界"**防 agent 在测试里信自己的嘴；**同 PR snapshot**让模型可见行为的每次改动都留下可回放的证据。整套设计的对手模型不是马虎的人类，而是"高产但会自信地错"的 agent——这正是它对其他 agent-first 项目的参考价值。
