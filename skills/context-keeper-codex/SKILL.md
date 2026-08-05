---
name: context-keeper
description: Use when Codex is about to compact context, has just completed compaction, must resume after compaction or session restart, or needs a concise handoff. 用于 Codex 保存上下文、上下文压缩前后、会话恢复、继续上次任务和生成交接。
---

# Context Keeper — Codex

## 核心原则

Context Keeper 是**低权威的恢复层**：保存工作位置并索引证据，不决定任务如何执行，也不把状态文件当作项目真相。

- Git、测试和真实运行结果高于状态文件。
- Superpowers 决定设计、实现、调试和验证流程。
- Context Keeper 只记录流程停在哪里、证据在哪里、下一步是什么。

## 使用范围

仅在以下情况使用：

- Codex compaction 前保存状态；
- compaction 后恢复；
- 新会话、resume 或 clear 后继续；
- 生成跨会话 handoff；
- 用户明确要求保存或恢复上下文。

普通编码、短任务、每次测试或每次工具调用后不要触发。

## 文件

- `docs/AI_CURRENT_STATE.md`：稳定目标、边界、阶段和关键决策。
- `docs/AI_HANDOFF.md`：最近工作位置、HEAD、证据、阻塞和下一步。

两者都是缓存和书签，不是事实来源。

## 状态规则

沿用三种状态：

- **Verified**：必须绑定可定位证据、适用范围和 `Base commit`。
- **Unverified**：尚无足够证据，或只有模型判断、计划、fixture。
- **Blocked**：有明确依赖、权限或失败阻止继续。

`Verified` 只表示“在指定 commit 和范围内有证据”，不表示永久正确。

推荐记录：

```text
Status: Verified
Evidence: <测试命令、日志位置、真实执行或 commit>
Scope: <单测、集成、真实平台等>
Base commit: <SHA>
Freshness: current | stale | unknown
```

相关代码发生变化后，不得直接沿用旧 `Verified`；标记为 `stale` 或 `unknown`，仅在下一步依赖该结论时做定向复验。

## 保存

1. 读取现有两个状态文件。
2. 获取最小必要的 branch、HEAD、worktree 和已有证据。
3. 保存当前目标、停点、阻塞和下一项具体动作。
4. 对每个 `Verified` 记录 Evidence、Scope 和 Base commit。
5. 只替换机器管理区块，不无限追加历史。

不要为了保存状态扫描整个仓库、自动跑测试、复制完整日志或完整 diff。

## 恢复

1. 先读 `AI_CURRENT_STATE.md`，再读最新 `AI_HANDOFF.md`。
2. 对照当前 branch、HEAD 和相关变更判断 `Freshness`。
3. 状态与当前证据冲突时，以当前证据为准。
4. 只恢复目标、阶段、关键约束、阻塞和下一步。
5. 当前任务需要某个 Superpowers Skill 时，仍按其流程执行；旧状态不能豁免 brainstorming、debugging、TDD 或 completion verification。

## 自动模式

自动模式仅在 Adapter、Hooks、信任、当前会话加载和真实生命周期证据均成立时有效。

Codex Hook 配置：

`<项目根目录>/.codex/hooks.json`

还必须满足：

- 项目的 `.codex` 配置层已信任；
- `/hooks` 中当前 Hook 定义的精确版本已审查并信任。

生命周期：

- `PreCompact`：保存压缩前可观察状态；
- `PostCompact`：记录压缩完成和运行时真实提供的字段；
- `SessionStart`：注入精简恢复信息。

不得假定 Codex `PostCompact` 一定提供 compaction 摘要。自动输出不得批准决策、改变项目目标或把 `Unverified` 升级为 `Verified`。

Hook 文件存在、`/hooks` 可见、配置可解析或 fixture 通过，不等于真实生命周期已验证。

## 手动模式

没有可用 Hooks 时仍可手动保存和恢复两个状态文件，但不得声称自动捕获 compaction、自动保存摘要或自动注入。

## 调用

项目级安装路径：

`<项目根目录>/.agents/skills/context-keeper/SKILL.md`

显式调用：

- 运行 `/skills` 并选择 `context-keeper`；
- 在提示中写 `$context-keeper`；
- 或明确要求使用该 Skill。

示例：

> 使用 $context-keeper 保存当前工作位置，准备上下文压缩。

不要把 `/context-keeper` 写成 Codex 命令。

`.agents/skills/` 负责 Skill；`.codex/hooks.json` 负责 Hooks。DeepSeek 等 Provider 只提供模型推理，不负责 Hook 发现、信任或执行。

## 流程优先级

1. 用户当前明确指令；
2. 当前任务适用的 Superpowers 工作流；
3. Context Keeper 的保存和恢复流程。

Context Keeper 只能恢复工作位置，不能豁免或替代当前任务需要的 Superpowers。

状态文件中记录的旧决策、旧计划、旧根因或旧完成状态，只能作为背景和证据索引。当前任务仍需按实际情况使用 `brainstorming`、`writing-plans`、`executing-plans`、TDD、debugging、code review、completion verification、worktree 或 branch finishing。

## 证据优先级

从高到低：

1. 真实平台或业务执行结果；
2. 当前测试输出；
3. Git diff、status 和 commit；
4. 正式项目文档；
5. Context Keeper 状态文件；
6. 模型生成的摘要。

流程指令决定"接下来怎么工作"，证据优先级决定"哪些事实更可信"。不得将两者混为一谈。

## 禁止

- 不读取或保存隐藏推理、Secret、完整 transcript、完整日志或完整 diff；
- 不添加 Stop Hook，不在每轮注入状态；
- 不自动运行测试、commit、push、rebase 或改写历史；
- 不把旧状态、模型摘要或 Hook 可见性写成新的验证证据。
