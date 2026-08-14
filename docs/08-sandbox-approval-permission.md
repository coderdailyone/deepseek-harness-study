# 08 · 沙箱/审批/权限：内核裁决、一次升权、日志即策略存储

> 研究对象：deepseek-harness @ `47f9438`。依据：`.agents/notes/implemented/feature/2026-07-06-sandbox.md`（39KB，全仓库质量最高的设计档案之一）、`2026-07-06-approval-seam.md`、`native/landlock-run/`、`packages/sandbox/*`。

## 产品路径一句话

bash 子进程（含搭它便车的 hook 命令）**默认受限**运行；**当且仅当沙箱真的拒绝了某个操作**，模型才可以为同一操作请求一次用户审批，批准后以更宽权限重试一次。

这句话里每个限定词都是防御：不是"每个工具都有边界"（fs/web/todo 在进程内执行，`execve` 包装器对闭包毫无意义——诚实划界）；不是"模型可以随时要权限"（升权必须以真实拒绝为根据）；不是"批准后就宽了"（授权只覆盖那一次重试，不持久化）。

## `ctx.sandbox` 接缝：把"限制"做成可组合能力

`confine(argv, policy)` 返回**替代调用者原 argv 的包装 argv**（进程及其一切子进程都受限）+ 该后端达到的 `enforcement` 完整度 + 拒绝方言（`denialSignatures`，该内核在文件拒绝时打的 stderr 子串）+ 结构化 runner 失败证据。**没有可用后端时抛 fail-closed 的 `SANDBOX_UNAVAILABLE`，绝不静默降级为无限制执行。**

三档模式是**只关文件效果**的封闭词表：`read-only` / `workspace-write` / `danger-full-access`——"网络和进程可见性不在声明范围"，能管什么就说什么。

两个容易做错的边界他们划得很清楚：

- **策略随调用走，不随 provider 配置**：同一时刻 bash 可以在 read-only 下跑而某受限子 agent 保持自己状态目录可写；升权重试就是一个带更宽策略的新调用——config 固定模式表达不了这些
- **本接缝只管同世界子进程**：容器/microVM/远程执行器**不是**这个接缝的后端——它们作为环境一致的整组替换 `ctx.shell`+`ctx.fs` 的 provider，"bash 在容器里跑而 fs 工具写宿主机 = 活在两个分裂世界的 agent"

## 自带的 Landlock 启动器：~300 行 C

首选 runner bwrap 恰恰在最需要沙箱的主机上不可用（最小容器、禁 userns、LSM 拒 mount），所以 SDK 自带 fallback：`native/landlock-run`，约 300 行 C11 直接写 Landlock 原始 UAPI，除静态链接的 musl 外零依赖——"审计面就是这一个文件加内核的稳定 syscall 契约"。`--ro/--rw` 授权、给自己装规则集后 `exec`（规则集跨 execve 继承）、restrict 前设 `no_new_privs`、`--probe` 在短命子进程里验证内核真的强制执行。退出码纪律：启动器失败一律 125 + 致命 `landlock-run:` 行；老 ABI 打"partial enforcement"通知行后照常执行孩子（该行不是致命证据）。

平台链：Linux 探测 bwrap → Landlock；macOS Seatbelt；**Windows 保留为空 = fail-closed**（评估过第三方 landstrip，rejected 档案原文："对安全不变量来说不够身经百战"）。

被拒绝的替代方案里有两条工程价值观：**提交编译好的二进制**——"diff 里的二进制不可评审且搅动历史"；**安装时编译**——"把 C 工具链推给每个消费者；只在恰好有编译器的地方存在的 fallback 不是 fallback"。

## 升权：一次经批准的更宽重试

- 升权字段（`sandbox_permissions` + `justification`）**按能力门控**：只有挂载了受限执行器才出现在 bash 工具 schema 里。被拒绝的替代方案说得好："在 `bash-local` 下它们是死杠杆——宣传一个 harness 无法兑现的选项就是在制造注定失败的授权"
- 目标必须**严格宽于该调用的会话有效模式**；审批在执行前解决；`rejected`/`cancelled`/`unavailable`/审批服务缺席全部 fail closed 且各有可区分的结果
- **重试是一个新的、被记录的工具调用**。被拒绝的"同一调用内自动重试"方案的理由直击事件溯源纪律："日志无法重建的隐藏重入：一个 `tool/call` 将产生两次不同策略的执行"
- 被拒绝的"命令串启发式预检"："无法理解展开/子进程/符号链接；严格尝试（跑它，让内核裁决）是唯一可信的拒绝信号"
- 拒绝标记 `[sandbox: file access denied under <mode> mode]` 带明确指示：不要绕过拒绝换路重试

## 会话日志即策略存储

每会话的运行时策略开关不进配置文件、不进外部存储——**就是会话日志里的事件**：

```
effective(session) = findLast(该会话的开关事件)?.value ?? 组合配置默认值
```

每个开关由其领域包拥有同一套三件套：事件声明（`sandbox/mode`、`approval/policy`）、纯 fold（`effectiveSandboxMode(events)`，就是一个类型化 findLast）、唯一写路径（切换即事件，无带外状态变更）。重启免疫和多会话隔离**从回放自动获得**。第三个开关照抄这个 ~40 行模式即可——无共享 owner 服务、无泛型 facts map、无注册表。

委托的孩子（06 篇）在委托边界快照父的显式覆盖、以 `source: 'delegation'` 事件种进孩子日志——"委托不能回退到更宽的默认值"。

`permission-presets` 是 UI 面：预设表把两个开关捆成一档（workspace-write / danger-full-access），写穿到两个领域 setter；表外组合如实报 `custom`。自动化用的 ACP 组合不挂 UI 服务、显式选模式。

## 跨家族一致性

进程内的 fs 工具经 `dsh-fs-sandbox` 强制**同一套模式词表**（read-only/workspace-write 对文件系统工具是真边界，不是 bash 专属的近似），且 `ctx.sandboxPolicy` 是唯一策略家——"bash 和 fs 不可能被限制到不同的根"。web 工具诚实地不设栏（其唯一效果是网络，在文件效果词表之外）。

## 测试分四层

单元（平台选择/失败分类/升权校验/preset 折叠）→ **keyless 真 runner**（bwrap/Landlock/Seatbelt 打真实文件效果，一个真 Cordis 上下文并发驱动两个项目会话证明"自己根成功、兄弟根拒绝"；CI 拒绝静默全跳过）→ 带 key（真 ACP 组合里模型驱动的写被真拒绝、再走批准与拒绝两分支）→ snapshot（钉策略上下文与两个审批分支的脚本化 transcript）。

## 为什么这篇档案值得单独一读

它把一个安全特性的**每一条边界都写成了可辩护的决策**：能管什么（文件）不能管什么（网络）、在哪一层管（execve 包装 vs 进程内 seam 策略）、失败往哪边倒（一律 fail closed）、证据怎么分级（内核拒绝 vs runner 失败 vs 普通命令失败，各有结构化判据）。Alternatives considered 列了 15+ 条被拒方案，每条都有一句话可引用的败因——这正是 01 篇讲的"决策记录防止重新扯皮"制度的最佳样本。
