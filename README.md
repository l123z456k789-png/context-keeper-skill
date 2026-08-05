# Context Keeper 双平台 Skill 详细说明书

## 1. 本版本的定位

本包提供两份平台专用 Skill：

```text
skills/
├── context-keeper-claude-code/
│   └── SKILL.md
└── context-keeper-codex/
    └── SKILL.md
```

安装后两边都叫：

```text
context-keeper
```

这次修订把 Context Keeper 明确收缩为：

> **低权威、证据索引型、按需触发的上下文恢复层。**

它负责保存“工作停在哪里”，不负责决定“工作应该怎么做”。

它不替代：

- Claude Code 或 Codex 的原生 compaction；
- Git；
- 测试和真实运行；
- 项目正式文档；
- Superpowers；
- 人类批准的设计与计划。

详细 README 面向使用者和维护者，正常情况下不会自动注入每次模型上下文。真正影响运行时上下文成本的是对应平台的 `SKILL.md`。

---

## 2. 为什么要再次精简

上一版两份 Skill 各约 200 行，虽然边界完整，但存在三个风险。

### 2.1 触发范围过宽

原 description 包含：

- long-running tasks；
- repeated work；
- goal drift。

这些词也会出现在普通规划、开发和调试任务中，可能让 Context Keeper 与 Superpowers 同时自动触发。

新版 description 只保留明确生命周期信号：

- compaction；
- `/compact`；
- save context；
- restore/resume；
- session restart；
- handoff；
- 保存上下文；
- 会话恢复；
- 继续上次任务。

“长期任务”“减少重复工作”“防止目标漂移”仍是产品价值，但不再作为宽泛自动触发条件。

### 2.2 `Verified` 容易被误解为永久真相

状态文件由模型维护。如果只写：

```text
功能 X：Verified
```

后续模型可能把它当成永久权威，从而跳过必要校验。

新版不删除 `Verified`，避免破坏现有模板和 Adapter 兼容性，但重新定义它：

> `Verified` 只表示某项结论在指定 `Base commit`、指定 `Scope` 内有可定位的 `Evidence`。

它必须能回答：

1. 证据在哪里？
2. 验证覆盖什么范围？
3. 对应哪个 commit？
4. 当前是否仍新鲜？

### 2.3 与 Superpowers 的优先级不够硬

旧状态只能告诉模型”上次做到哪里”，不能让模型跳过当前请求需要的流程。

新版将原来混在一起的优先级拆为两条独立规则——流程优先级决定”怎么工作”，证据优先级决定”相信什么”：

- 流程优先级：用户当前明确指令 > 当前任务适用的 Superpowers 工作流 > Context Keeper 的保存和恢复流程；
- 证据优先级：真实平台或业务执行结果 > 当前测试输出 > Git diff/status/commit > 正式项目文档 > Context Keeper 状态文件 > 模型摘要。

详见第 11 节。

如果新任务需要 brainstorming、TDD、systematic-debugging 或 verification-before-completion，仍须正常使用。状态文件不能作为豁免理由。

---

## 3. 最终职责模型

推荐把整个系统理解为：

```text
Superpowers = 驾驶员
Git / Tests / Runtime = 仪表与事实
Context Keeper Hooks = 行车记录仪
Context Keeper Skill = 书签和恢复说明
Markdown = 持久化载体
```

### 3.1 Superpowers 负责

- 需求澄清；
- brainstorming；
- 设计；
- writing-plans；
- executing-plans；
- TDD；
- systematic debugging；
- code review；
- completion verification；
- worktree 和 branch 收尾。

### 3.2 Context Keeper 负责

- 压缩前保存工作停点；
- 压缩后保存平台可观察结果；
- 新会话恢复最小状态；
- 保存证据索引；
- 标记状态是否过期；
- 生成 handoff。

### 3.3 Context Keeper 不负责

- 批准设计；
- 判断新功能是否应该做；
- 选择实现方案；
- 自动决定任务完成；
- 自动把测试结果外推为真实 E2E；
- 替代任何 Superpowers Skill；
- 创建第二套计划系统。

---

## 4. 为什么仍保留两份 Skill

Claude Code 和 Codex 的平台差异是真实存在的。

| 项目 | Claude Code | Codex |
|---|---|---|
| Skill 安装路径 | `.claude/skills/context-keeper/` | `.agents/skills/context-keeper/` |
| 显式调用 | `/context-keeper` | `/skills` 或 `$context-keeper` |
| Hook 配置 | `.claude/settings.json` / `.claude/settings.local.json` | `.codex/hooks.json` |
| Hook 信任 | 项目权限和命令授权 | 项目配置层 + `/hooks` 精确 Hook 信任 |
| `PostCompact` 摘要 | 可使用 `compact_summary` | 不默认假定存在摘要 |
| Provider 关系 | 模型与 Hook 分离 | DeepSeek 等 Provider 不实现 Hook |

把两边写进同一份 Skill，会增加错误命令、错误路径和能力串用的风险。

不采用“共享核心文件 + 两个薄入口”，因为那会增加：

- 安装器复制规则；
- 相对路径依赖；
- 额外文件加载；
- references 缺失风险；
- 维护检查范围。

目前两份短 Skill 的少量重复是合理成本。

---

## 5. 推荐源码和安装结构

### 5.1 源码仓库

```text
skills/
├── context-keeper-claude-code/
│   └── SKILL.md
└── context-keeper-codex/
    └── SKILL.md
```

### 5.2 安装到 Claude Code 项目

源文件：

```text
skills/context-keeper-claude-code/SKILL.md
```

目标：

```text
<目标项目>/.claude/skills/context-keeper/SKILL.md
```

显式调用：

```text
/context-keeper
```

### 5.3 安装到 Codex 项目

源文件：

```text
skills/context-keeper-codex/SKILL.md
```

目标：

```text
<目标项目>/.agents/skills/context-keeper/SKILL.md
```

显式调用：

```text
/skills
```

选择 `context-keeper`，或者在提示中写：

```text
$context-keeper
```

Codex 不应使用：

```text
/context-keeper
```

### 5.4 Codex 两个隐藏目录的分工

```text
.agents/skills/context-keeper/
```

负责 Skill 发现。

```text
.codex/hooks.json
```

负责 Hook 配置。

两者目录名不同是正常架构，不是路径冲突。

---

## 6. 两个状态文件

固定使用：

```text
docs/AI_CURRENT_STATE.md
docs/AI_HANDOFF.md
```

当前不增加任意路径配置，以避免 Adapter、Hook、安装器和 Skill 之间产生更多分支。

### 6.1 `AI_CURRENT_STATE.md`

保存相对稳定的信息：

- 项目目标；
- 非目标和安全边界；
- 当前阶段；
- 已批准的关键决策；
- 关键能力状态；
- 稳定 Git 检查点；
- 当前主要阻塞；
- 下一阶段。

它不是实时日志，不应因每个文件修改而重写。

### 6.2 `AI_HANDOFF.md`

保存最近工作现场：

- 保存原因和时间；
- branch 和 HEAD；
- worktree 状态；
- 当前任务和停点；
- 关键变更范围；
- 证据索引；
- 当前阻塞；
- 下一项具体动作；
- 平台真实提供时的脱敏摘要。

它应覆盖最新机器管理区块，不应无限追加。

---

## 7. `Verified` 的新定义

为了兼容现有文件，仍保留：

- `Verified`
- `Unverified`
- `Blocked`

但必须按以下规则使用。

### 7.1 Verified

必须同时具备：

```text
Evidence
Scope
Base commit
Freshness
```

示例：

```markdown
- Status: Verified
- Evidence: `pytest tests/test_installer.py -q` → 39 passed
- Scope: Installer unit tests only
- Base commit: `abc1234`
- Freshness: current
```

这表示：

- 在 `abc1234`；
- 安装器单元测试范围内；
- 有 39 tests passed 的证据。

它不表示：

- 真实 Claude Code 生命周期已通过；
- 真实 Codex Hook 已触发；
- 所有平台都兼容；
- 后续代码变化后仍然成立。

### 7.2 Unverified

以下情况必须标为 Unverified：

- 只有代码阅读；
- 只有模型判断；
- 只有设计文档；
- 只有 fixture；
- 测试范围不足以支持产品级结论；
- 当前 HEAD 与 Base commit 不同且影响不明。

### 7.3 Blocked

示例：

- Hook 尚未信任；
- 项目配置未加载；
- 平台不提供所需字段；
- 真实测试环境不可用；
- 权限失败；
- 已观察到确定错误。

Blocked 必须记录具体阻塞，而不是笼统写“暂时无法继续”。

---

## 8. Freshness 过期机制

建议状态条目或机器管理区块包含：

```text
Last updated: 2026-08-05
Base commit: abc1234
Freshness: current | stale | unknown
```

### 8.1 current

当前 HEAD 与 Base commit 相同，或者变化明确与该结论无关。

### 8.2 stale

与结论有关的代码、配置、依赖或环境已经变化。

旧证据仍可作为历史参考，但不能继续当作当前 Verified。

### 8.3 unknown

HEAD 已变化，但无法通过最小检查判断是否影响该结论。

不要猜测，应保持 unknown。

### 8.4 恢复时的判断

```text
HEAD == Base commit
    → 通常 current

HEAD 不同，相关文件未变化
    → 可以保留 scope，但注明 commit 差异

HEAD 不同，相关文件已变化
    → stale

无法判断关联范围
    → unknown
```

不要求每次恢复都运行完整测试。

只有当下一步依赖某个 stale/unknown 结论时，才运行最小定向验证。

---

## 9. 自动模式的安全边界

自动模式仍然保留，但它必须是保守的。

### 9.1 自动模式可以记录

- 时间；
- trigger；
- branch；
- HEAD；
- worktree clean/dirty；
- changed files 或 diffstat；
- 已存在测试结果的引用；
- 当前阻塞；
- 上一次明确的 next action；
- 平台真实提供的 compaction summary。

### 9.2 自动模式不能决定

- 设计已批准；
- 功能已经完成；
- 根因已经证明；
- 测试覆盖真实 E2E；
- 项目阶段已经结束；
- 可以跳过 Superpowers；
- `Unverified` 可以升级为 `Verified`。

Hook 和 Adapter 输出最多是“观察结果”。语义升级必须由可定位证据支持，并受 Scope 和 Base commit 限制。

### 9.3 真实自动模式条件

不能仅凭文件存在宣称自动模式有效。

#### Claude Code

需要：

1. Adapter 已安装；
2. Hooks 已配置；
3. 项目权限和命令授权允许执行；
4. 当前会话已加载；
5. 真实 `PreCompact` 已观察；
6. 真实 `PostCompact` 已观察；
7. compact 后 `SessionStart` 已观察；
8. 状态文件和恢复输出符合预期。

#### Codex

需要：

1. Adapter 已安装；
2. `.codex/hooks.json` 可解析；
3. 项目 `.codex` 配置层受信任；
4. `/hooks` 中当前精确 Hook 定义已信任；
5. 当前会话已发现；
6. 真实 `PreCompact` 已观察；
7. 真实 `PostCompact` 已观察；
8. compact 后 `SessionStart` 已观察；
9. 状态文件和恢复输出符合预期。

以下都不能替代真实执行：

- fixture；
- 单元测试；
- Hook 列表可见；
- JSON 解析成功；
- 手动运行 Adapter 成功；
- 模型推测。

---

## 10. 生命周期事件

两个平台都使用正式名称：

```text
PreCompact
PostCompact
SessionStart
```

### 10.1 PreCompact

职责：

- 记录压缩前工作位置；
- 保存最小 Git 状态；
- 保存已有证据索引；
- 保存阻塞和 next action。

不自动扫描全仓库，不自动执行整套测试。

### 10.2 PostCompact

Claude Code：

- 可以保存平台提供的 `compact_summary`；
- 必须先脱敏；
- 摘要的权威级别低于 Git、测试和真实运行。

Codex：

- 记录压缩完成；
- 只使用运行时真实提供的字段；
- 不默认假定存在 compaction summary；
- 不尝试读取隐藏会话历史来补造摘要。

### 10.3 SessionStart

职责：

- 读取两个状态文件；
- 对照当前 HEAD；
- 标记 freshness；
- 注入最小恢复信息。

推荐恢复输出只包含：

1. 当前目标；
2. 当前阶段或任务；
3. branch / HEAD；
4. 最关键约束；
5. 当前阻塞；
6. 下一项动作；
7. 需要重新验证的 stale/unknown 结论。

不要把完整状态文件逐字注入上下文。

---

## 11. 与 Superpowers 的冲突规避

### 11.1 流程优先级

```text
用户当前明确指令
    >
当前任务适用的 Superpowers 工作流
    >
Context Keeper 的保存和恢复流程
```

这条优先级只回答：

> 接下来应该按照什么流程工作？

Context Keeper 只能恢复停点、证据索引和下一步，不能让模型跳过当前任务应使用的 Superpowers。

状态文件中记录了"架构已批准"，不代表新的设计变化可以跳过 `brainstorming`。

状态文件中记录了"根因已确认"，不代表新的错误表现可以跳过 `systematic-debugging`。

状态文件中记录了"测试通过"，不代表当前 HEAD 变化后可以跳过 `verification-before-completion`。

### 11.2 证据优先级

```text
真实平台或业务执行结果
    >
当前测试输出
    >
Git diff、status 和 commit
    >
正式项目文档
    >
Context Keeper 状态文件
    >
模型摘要
```

这条优先级只回答：

> 多个信息来源冲突时，哪个事实更可信？

用户指令和 Superpowers 工作流不属于事实证据，不应放进证据可信度排序。

流程优先级决定"怎么工作"，证据优先级决定"相信什么"。两者不能混用。

### 11.3 典型情形

#### 新增功能

状态文件写着“架构已批准”，但用户现在提出新功能。

结果：

- 新功能仍需 `brainstorming`；
- 旧架构决策只作为背景；
- Context Keeper 不得宣布跳过设计。

#### 新 Bug

handoff 写着“根因已确认”，但现在错误表现发生变化。

结果：

- 使用 `systematic-debugging`；
- 旧根因标记为历史证据；
- 不直接套用旧结论。

#### 继续现有计划

handoff 清楚记录计划已批准并停在第 3 阶段。

结果：

- Context Keeper 恢复停点；
- `executing-plans` 决定如何继续；
- Context Keeper 不执行计划。

#### 声称工作完成

状态文件写着“全部通过”，但当前 HEAD 已变化。

结果：

- 使用 `verification-before-completion`；
- 对受影响范围重新验证；
- 不依据状态文件直接宣布完成。

### 11.4 不需要同时触发的情形

普通短任务、单次代码解释、无跨会话风险的修改，不应加载 Context Keeper。

---

## 12. 推荐使用频率

使用：

- compaction 前；
- compaction 后；
- 新会话恢复；
- 长任务跨窗口；
- 阶段性交接；
- 用户明确要求保存或恢复。

不要使用：

- 每次代码修改后；
- 每次测试后；
- 每次工具调用后；
- 普通问答；
- Superpowers 每个小步骤后；
- 没有上下文丢失风险的短任务。

通常一个长会话最多需要：

1. 压缩前一次；
2. 压缩后一次；
3. 最终 handoff 一次。

自动 Hooks 可以按生命周期运行，但输出必须保持精简、去重和低权威。

---

## 13. 手动使用示例

### 13.1 Claude Code 压缩前

```text
/context-keeper 保存当前工作位置，准备执行 /compact。
```

预期动作：

1. 读两个状态文件；
2. 取 branch、HEAD、worktree；
3. 记录现有证据和停点；
4. 不自动运行新测试；
5. 不自动 commit。

### 13.2 Claude Code 恢复

```text
/context-keeper 根据当前 Git 恢复最近工作位置，并标出过期结论。
```

### 13.3 Codex 压缩前

```text
使用 $context-keeper 保存当前工作位置，准备上下文压缩。
```

### 13.4 Codex 恢复

```text
使用 $context-keeper 恢复最近工作位置；状态文件只作索引，以当前 Git 和证据为准。
```

### 13.5 无 Hooks

```text
使用 context-keeper 手动更新 AI_HANDOFF.md。
不要声称自动生命周期已经运行。
```

---

## 14. 推荐恢复摘要格式

SessionStart 或手动恢复输出建议控制为：

```markdown
## Context Keeper Recovery

- Goal:
- Current task:
- Branch / HEAD:
- Evidence-backed position:
- Stale or unknown items:
- Blocker:
- Next action:
- Required workflow:
```

`Required workflow` 可写：

```text
superpowers:systematic-debugging
```

但 Context Keeper 只提示恢复后的流程入口，不代替该 Skill。

---

## 15. 安全与脱敏

不得保存：

- API Key；
- Access Token；
- Cookie；
- 密码；
- Authorization Header；
- 私钥；
- 完整环境变量；
- 带凭据 URL；
- 完整 transcript；
- 完整日志；
- 完整 diff；
- 隐藏推理。

可以保存：

```text
Authentication configured: yes
Observed error: HTTP 401
Secret source: environment variable
```

不能保存真实值。

Claude Code `compact_summary` 或其他平台摘要在写入前也必须执行同样脱敏。

---

## 16. 上下文成本控制

两份 `SKILL.md` 保留：

- 核心原则；
- 明确触发范围；
- 状态证据规则；
- 保存和恢复步骤；
- 自动/手动模式边界；
- 平台调用差异；
- Superpowers 优先级；
- 安全规则。

以下细节放在本 README，而不进入每次 Skill 加载：

- 完整设计理由；
- 大量使用示例；
- 生命周期验收清单；
- 维护规则；
- 冲突场景分析；
- 详细过期机制解释。

两份 Skill 不依赖 `references/`，安装时只复制对应 `SKILL.md` 即可使用手动模式。

---

## 17. 安装器要求

安装器应：

1. 根据平台选择正确源文件；
2. 安装后的目录名统一为 `context-keeper`；
3. 使用 UTF-8，无 BOM；
4. 不交叉安装平台文件；
5. 不让 Skill 依赖未复制文件；
6. 备份未知用户文件；
7. 验证 frontmatter；
8. 不自动启用或信任未知命令；
9. 不因静态安装成功就宣称真实 lifecycle 已通过。

自动模式还需分别安装和验证 Adapter、Hooks 与状态模板。

---

## 18. 维护规则

修改任一 Skill 时，检查共享语义：

- 状态文件是低权威索引；
- Verified 绑定证据、范围和 commit；
- freshness 规则一致；
- Superpowers 优先；
- 自动 Hook 不升级语义状态；
- 安全边界一致。

不要机械同步平台差异：

- 安装路径；
- 调用命令；
- Hook 配置；
- 信任机制；
- PostCompact 输入；
- Provider 说明。

发布前回答：

1. description 是否只覆盖明确 compaction/recovery/handoff 场景？
2. 是否可能因普通编码而自动触发？
3. 是否把状态文件写成项目真相？
4. 是否允许旧状态跳过 Superpowers？
5. 是否把 fixture 当成真实 lifecycle？
6. 是否新增未复制依赖？
7. 是否明显增加 Skill 长度？
8. 是否确认两份 Skill 的以下共享章节逐字一致：核心原则、使用范围、文件、状态规则、保存、恢复、流程优先级、证据优先级、禁止。

未发现现有静态验证脚本，因此本轮没有创建新的验证程序；共享语义漂移暂由 README 发布检查约束。

---

## 19. 验收标准

### Claude Code Skill

- `name: context-keeper`；
- description 以 `Use when` 开头；
- description 不使用宽泛的 long-running/repeated-work/goal-drift 触发；
- 明确 `/context-keeper`；
- 正确安装路径；
- 正确事件名；
- `PostCompact` 可使用 `compact_summary`；
- 状态文件低于真实证据；
- 不能跳过 Superpowers。

### Codex Skill

- `name: context-keeper`；
- description 以 `Use when` 开头；
- 明确 `/skills` 和 `$context-keeper`；
- 不把 `/context-keeper` 当命令；
- 正确安装路径和 Hook 路径；
- 正确说明项目与 Hook 信任；
- 不默认存在 compaction 摘要；
- Provider 不实现 Hook；
- 不能跳过 Superpowers。

### 共同要求

- 每个 Verified 绑定 Evidence、Scope、Base commit；
- 支持 current/stale/unknown；
- 自动模式不升级语义状态；
- 不自动测试、commit 或 push；
- 不依赖 references；
- UTF-8，无 BOM；
- 主 Skill 保持紧凑；
- README 详细但不自动加载。

---

## 20. 最终决策

最终保留：

- 两份平台专用 Skill；
- 两个固定状态文件；
- 自动 Hooks；
- 手动保存和恢复；
- Verified / Unverified / Blocked 兼容标签。

最终新增约束：

- 状态文件只是证据索引；
- Verified 必须带证据、范围、commit 和 freshness；
- 相关代码变化后旧 Verified 失效或待确认；
- 自动 Hook 不负责语义升级；
- Superpowers 拥有执行流程优先级；
- description 只在明确 compaction、recovery 和 handoff 场景触发。

这使 Context Keeper 成为 Superpowers 的“停点保存器”，而不是与其竞争的第二套工作流。
