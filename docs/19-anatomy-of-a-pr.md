# 19 · 案例研究：解剖一个 44 行的 PR

> 研究对象：deepseek-harness PR #2484（merge `10be203b`，2026-08-13，分支 `worktree/fail-web-wildcard-host`）："fix(web): reject unsupported wildcard host"。15 个文件、+44/−25 行。选它正因为它小——小改动最能看清一套工程制度的固定成本与固定收益。

## 改动本身

`dsh web --host 0.0.0.0` 从"帮助文本推荐的功能"变为"启动前拒绝"：

```
error: --host 0.0.0.0 is intentionally not supported yet for safety:
it would expose remote code execution to the network; use 127.0.0.1 instead
```

拒绝发生在 `webStartup` 服务发布之前（什么都没绑定就退出，exit 1）。错误消息三要素齐全：intentionally（这是决策不是缺陷）、yet（不永久关死）、告诉你替代方案。

## 44 行如何铺满 15 个文件

**1. 源码 + 就地文档（5 文件）**：`startup.ts` 加拒绝逻辑的同时，**删掉了** `--host` 选项帮助文本里的 "pass 0.0.0.0 to reach it from another machine" 和示例行——13 篇 D3 原则（"不宣传兑现不了的杠杆"）的镜像应用：**拒绝兑现的杠杆也不许再宣传**。同一 hunk 里 JSDoc 合同同步改写（"`--host 0.0.0.0` 或非数字 `--port` 是用法错误"）。隔壁包 `api-request-trust.ts` 的边界注释把 "CLI-derived LAN IP literals" 改为 "deployment-derived"——CLI 不再能产生 LAN 绑定，注释若不改就成了幽灵引用。散文与行为同 PR 保持一致，没有"下次再改文档"。

**2. 测试三层（3 文件）**：两个包的单元测试更新；关键的是 `built-bin.e2e.ts` 新增的断言——用**纯 node 跑构建产物** `lib/bin.js`（10 篇"真入口路径"），断言四件事：exit code 1、**stdout 为空**、stderr 含逐字错误串、**不含** `dsh web: http://` URL 行。既验证拒绝，也验证"拒绝前没有半启动的痕迹"。

**3. 双语文档三对（6 文件）**：三个包的 README.md + README.zh.md + `.i18n.yaml` 一致性记录全部重录（16 篇的配对系统实动）。英文侧 README 那句超长的 bundle 描述里精确插入一句 "It rejects `--host 0.0.0.0` before publishing that service because the CLI intentionally does not support all-interfaces binding yet."——中文侧同步、blob 哈希重录，配对门禁绿。

**4. 评审回环可见**：分支提交里有一条 "fix(web): address wildcard host review"——`worktree/` 前缀（agent 在 git worktree 里干活）+ 人类评审意见回改，01 篇讲的"agent 产出、人审并署名"责任模型在单 PR 尺度的切片。

## 一个诚实的观察：这个 PR 没有 Agent Note

制度说"非平凡改动 MUST 附 Agent Note"，而这是一个行为变更却没带笔记。可辩护的读法：决策的 why（安全）已完整写进错误消息、JSDoc 与三份 README——"一个事实一个家"原则下，这个决策的家被选在了离用户最近的地方；且"intentionally not supported **yet**"未关闭未来方向，没有产生需要防止重新扯皮的被拒方案。但按字面规则这是一次边界豁免。制度样本里包含边界案例本身就有研究价值：门禁管得住格式与配对，"什么算非平凡"的判断最终仍在评审者手里——与 16 篇"机器管形、评审管义"的分工一致。

## 这个案例说明的事

一行行为变更在这套制度下的**固定税**：帮助文本、JSDoc、跨包注释、三对双语 README、两层单元测试、一个构建产物 e2e——约 15 个文件。**买到的**：改动落地当刻，一切面向用户和 agent 的叙述都已一致，回归被真实入口的逐字断言钉死，中文读者与英文读者看到同一事实。多数团队省下这笔税，然后把它连本带利付给三个月后发现文档撒谎的那个人。这套制度的本质是把那笔迟付的债改成即付的税——而门禁保证税单不可赖账。
