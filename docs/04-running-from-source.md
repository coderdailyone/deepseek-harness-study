# 04 · 源码跑通记录（含无 root 环境的绕法）

> 环境：Ubuntu 22.04（Linux 5.15）、Node v24.18、无 root 权限、中国大陆网络（经本地代理）。目标：从 clone 到 headless 单任务与 Web UI 全链路跑通。

## 标准流程（顺利时）

```sh
git clone https://github.com/deepseek-ai/deepseek-harness
cd deepseek-harness
corepack enable          # 仓库钉 pnpm@11.7.0
pnpm install
pnpm run build           # Host tsc → Host tsdown → Client tsc → Client tsdown → Web
DEEPSEEK_API_KEY=sk-... pnpm dsh --profile headless "任务"
DEEPSEEK_API_KEY=sk-... pnpm dsh web    # Web UI: http://127.0.0.1:3080
```

实测构建全绿约几分钟；headless 默认模型为 `deepseek-v4-flash`（`agent-default-model` 配置行可 patch）。

## 踩到的四个坑与解法

### 坑 1：npmmirror 镜像 tarball 间歇超时

大体积包（esbuild 二进制、`@openai/codex` ~60MB——dsh 把 Codex CLI 作为依赖内置）在 `registry.npmmirror.com` 反复 `TimeoutError`。解法：改走官方源（如需代理自行配置）并放宽超时，pnpm 会复用已下载的 store，只补缺失部分：

```sh
npm_config_registry=https://registry.npmjs.org pnpm install --fetch-timeout 180000
```

### 坑 2：`pnpm install | tail` 吞掉真实退出码

管道的退出码来自最后一个命令（tail），第一次失败的 install 表面 exit 0。教训（通用）：**退出码判断永远不许把被测命令放在管道里**；输出落文件、`rc=$?` 直取：

```sh
pnpm install > install.log 2>&1; rc=$?; echo "REAL_EXIT=$rc"; tail install.log
```

### 坑 3：node-pty 需要本地编译，而机器没有 make/gcc 也没有 root

`node-pty@1.1.0` 上游只带 win32 预编译，Linux 走 node-gyp（`gyp ERR! stack Error: not found: make`）。无 root 装不了 build-essential 时的两步绕法：

**第一步**：让 install 先过——`pnpm-workspace.yaml` 的 `allowBuilds` 里把 `node-pty: true` 本地改为 `false`（pnpm 10+ 的脚本审查机制：false = 拒绝其构建脚本但安装继续）。

**第二步**：补预编译二进制。[`@lydell/node-pty`](https://www.npmjs.com/package/@lydell/node-pty) 是专门发布预编译产物的 node-pty 分发，**版本号与上游一一对应**。取对应平台包里的 `pty.node` 放进 node-pty 的 prebuilds 目录（node-pty 1.1.0 的加载器按 `build/Release → build/Debug → prebuilds/<platform>-<arch>` 顺序找 `pty.node`）：

```sh
curl -sL -o pty.tgz https://registry.npmjs.org/@lydell/node-pty-linux-x64/-/node-pty-linux-x64-1.1.0.tgz
tar xzf pty.tgz
PTY_DIR=$(ls -d node_modules/.pnpm/node-pty@1.1.0*/node_modules/node-pty)
mkdir -p "$PTY_DIR/prebuilds/linux-x64"
cp package/pty.node "$PTY_DIR/prebuilds/linux-x64/"
```

验证（从消费包目录内跑，pnpm 严格 node_modules 下根目录 resolve 不到）：

```sh
cd packages/subprocess/subprocess-local
node -e "const pty=require('node-pty'); const p=pty.spawn('echo',['ok'],{}); p.onData(d=>process.stdout.write(d))"
```

注意：重装 node_modules 后第二步要重做；spawn-helper 仅 macOS 需要，Linux 无此文件是正常的。

### 坑 4：为什么不能干脆禁用终端功能绕过 node-pty

试过：`dsh-subprocess-local`（bash 工具的底座）**顶层** `import * as nodePty from 'node-pty'`，boot 硬依赖。这是他们"misconfiguration fails loud"哲学的体现——没有静默降级路径，缺二进制就是起不来。所以只能给它二进制（上面第二步），不能配置绕过。

## 对读者的两个提示

- `pnpm dsh --profile headless --dump-config` 不挂载插件即可打印完整组合树，是理解 profile/bundle/patch 组合系统的最快入口
- 遥测默认 DISABLED（`session-telemetry-otel` 行 `mode: process.env.DSH_TELEMETRY_MODE || 'DISABLED'`），可在 dump 里亲自核实
