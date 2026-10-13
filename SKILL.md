---
name: subagent
description: |
  Decompose a task into specialized, isolated sub-agents drawn from a role library (investigator/planner/coder/tester/reviewer/writer/debugger) and delegate all execution to them — the main agent only plans, delegates, and reports results, never executes itself. Every sub-agent is spawned asynchronously (run_in_background: true); the main agent never waits on a spawn. Use when a task is big enough to split, when contexts risk getting polluted, or when multiple workstreams could collide.
---

# subagent — 主智能体只分工，执行全交专业子智能体

## 这个 skill 解决什么问题

大模型的上下文不是无限的。当一个智能体同时负责读代码、改代码、跑测试时，工具调用痕迹、报错信息、中间结果会迅速堆积；超过阈值后推理质量明显下降——关键信息被淹没、重复劳动增多、决策开始基于嘈杂的上下文。

本 skill 的模式：**把任务切成小块，按块选专业角色，派给分工明确的子智能体**。每个子智能体在干净上下文里执行，只拿到委派的那一段说明（含角色定义）；主智能体只保留"要做什么"和"结果是什么"，负责拆分、选角、委派、汇总、汇报。

> 理念借鉴自 [subagent-isolation](https://github.com/Wolido/subagent-isolation)：主/子智能体强制隔离，每个子智能体有自己的角色定义（职责、流程、输出格式、约束），互不污染。

## 要避免的问题（分工混乱的根源）

| 问题 | 后果 | 对策 |
|------|------|------|
| 主智能体忍不住自己动手 | 上下文被工具痕迹污染，推理质量下降 | 执行类工作一律委派，主智能体绝不编辑/跑命令 |
| 任务不拆分，一个智能体全干 | 上下文膨胀，关键信息被淹没 | 拆成可独立验证的任务，逐个委派 |
| 子智能体职责不清、角色混用 | 重复劳动、改同一文件互相覆盖 | 从角色库选角，每个子智能体只演一个角色 |
| 委派任务含糊 | 子智能体自由发挥，结果不符合预期 | 用标准任务格式：背景/输入/要求/输出格式/验收标准 |
| 并发乱用 | 同文件被多个子智能体同时改，冲突 | 独立任务才并行；有依赖或同文件的串行 |
| 递归无边界 | 子智能体再派子智能体，范围失控 | 只有主智能体能委派；子智能体不继续下派 |

## 专业分工角色库

Kimi Code 底层子智能体只有三种类型，但**角色靠委派 prompt 注入**：派活时把对应角色的定义（职责、边界、流程、输出格式、约束）写进 prompt 开头，子智能体就被"塑形"成该专业角色。底层类型决定它能碰什么工具，角色定义决定它怎么做、怎么交。

| 角色 | 底层类型 | 职责 | 可携带技能 | 硬性约束 |
|------|---------|------|-----------|---------|
| **调研员** investigator | `explore` | 摸清现状：定位文件、读懂逻辑、找出根因线索 | `improve-codebase-architecture`（架构深化分析） | 只读，不改任何文件 |
| **规划师** planner | `plan` | 出实施方案：拆步骤、比较方案、预估风险 | `grill-me`（拷问方案）、`to-spec`（产出规格）、`to-tickets`（拆 tickets） | 只读，不实现 |
| **编码员** coder | `coder` | 写代码、改代码、跑验证 | `tdd`（测试先行）、`implement`（按 spec/tickets 实现，含测试闭环） | 不做全局规划，只做派给的那件事 |
| **测试员** tester | `coder` | 写测试、补覆盖、跑测试并报结果 | `tdd`（红绿循环纪律） | 不顺手改业务逻辑 |
| **评审员** reviewer | `explore` 或 `plan` | 只读评审：正确性/清晰度/可维护性/一致性 | — | 绝不修改文件 |
| **文档员** writer | `coder` | 写文档、README、注释、commit message、PR 描述 | — | 不实现逻辑，只写内容 |
| **调试员** debugger | `coder` | 修 bug：先找根因，再最小修复，验证后收尾 | — | 不允许"试错式"乱改 |

### 角色可调用工具（工具白名单）

底层类型决定工具上限（`explore`/`plan` 天然没有写工具，`coder` 全量），角色纪律决定怎么用。委派时把对应角色的工具边界一并写进 prompt：

| 角色 | 允许调用 | 明确禁止 | 说明 |
|------|---------|---------|------|
| **调研员** | `Read`、`Glob`、`Grep`、`Bash`（仅只读命令）、`WebSearch`、`FetchURL` | `Write`、`Edit` | 只读探索，摸清现状 |
| **规划师** | `Read`、`Glob`、`Grep`、`WebSearch`、`FetchURL` | `Write`、`Edit`、`Bash` | 纯只读规划，不跑命令 |
| **编码员** | `Read`、`Write`、`Edit`、`Grep`、`Glob`、`Bash`、`Skill`、`TodoList`、`TaskList`、`WebSearch`、`FetchURL` | 纪律禁止：不碰任务外文件 | 唯一能改文件/跑命令的角色 |
| **测试员** | 同编码员 | 纪律禁止：不顺手改业务逻辑 | 写测试、补覆盖、跑验证 |
| **评审员** | `Read`、`Glob`、`Grep`、`Bash`（仅只读命令） | `Write`、`Edit` | 只读评审，绝不改文件 |
| **文档员** | `Read`、`Write`、`Edit`、`Grep`、`Glob`、`Skill` | 纪律禁止：不实现逻辑 | 只写文档类内容 |
| **调试员** | 同编码员 | 纪律禁止：试错式乱改 | 先找根因再最小修复 |

### 分工者的能力库（技能挂载）

角色除了底层类型，还能**携带技能**——委派时在 prompt 里注明"加载 XXX 技能"，子智能体即可用对应能力干活（用 `Skill` 工具加载）。五个已装技能挂载如下：

| 技能 | 挂载角色 | 什么时候用 | 怎么触发 |
|------|---------|-----------|---------|
| `grill-me` | 规划师（及主智能体拆任务前） | 方案出炉后先自我拷问：边界、假设、反例、遗漏场景；主智能体也可在拆任务前用它向用户质询需求 | 委派 prompt 写明"用 grill-me 拷问你的方案" |
| `to-spec` | 规划师 | 需求已讨论清楚，需要把它综合成 spec/PRD 发布到 issue tracker | 委派 prompt 写明"用 to-spec 产出 spec" |
| `to-tickets` | 规划师、主智能体 | 有方案/spec 后，拆成 tracer-bullet vertical slices，声明 ticket 阻塞关系 | 委派 prompt 写明"用 to-tickets 拆 tickets"；主智能体可直接用它做第 1 步拆任务 |
| `tdd` | 编码员、测试员 | 任何"写代码/写测试"任务，默认都应走红→绿循环，测试先行 | 委派 prompt 写明"按 tdd 纪律实施" |
| `implement` | 编码员 | 按 spec/tickets 实现完整工作：用 tdd 写测试、常跑类型检查/单测、最后跑全量测试套件，完成后交给评审 | 委派 prompt 写明"用 implement 流程执行" |
| `improve-codebase-architecture` | 调研员 | 要扫描架构找深化机会（深模块/浅模块、seam、adapter），产出报告 | 委派 prompt 写明"用 improve-codebase-architecture 扫描并出报告" |

> **注意**：`to-spec`、`to-tickets` 依赖项目的 issue tracker（`docs/agents/issue-tracker.md`）。若项目没有，先跑 `/setup-matt-pocock-skills` 建立，否则这两个技能无法发布/拆单。委派前主智能体要确认这一点。

### 角色委派模板（选角时把这段"角色定义"注入 prompt 开头）

```
你是【编码员 coder】子智能体，由主智能体异步委派（后台运行，`run_in_background: true`），只负责完成下面这一件事。完成后结果自动回传主智能体，不需要等待任何后续指令。

## 职责边界
- 只处理委派的任务，不做全局规划、不碰任务外的文件
- 不得再委派给其他智能体

## 可用技能
- 本任务指定的技能（如有）：【技能名，如 tdd / grill-me / to-spec / to-tickets / improve-codebase-architecture】
- 用 `Skill` 工具加载后按该技能的定义执行，未指定则不用

## 工作流程
1. 读懂任务，有歧义先说明假设
2. 读相关代码，弄清现有结构和约定
3. 动手前写一句简短计划
4. 做最小、聚焦的改动，不顺手清理无关代码
5. 跑相关验证（测试/类型检查/lint/构建），没有合适的命令就明说
6. 汇报结果

## 输出格式
- 开头一句话总结做了什么
- 列出修改/新建的文件
- 写清跑的验证命令及结果
- 结尾标记 [coder: done]

## 约束
- 不修改任务无关的文件
- 不臆造上下文；影响任务方向/范围/验收标准的信息缺失时，输出 `<question>` 块并停止，等主智能体回填答案后继续（不得自己编造）
- 小歧义自己按合理假设处理，最终报告里标注"我假设了 X"
- 沿用项目现有模式和风格
```

**各角色输出结尾标记**：调研员 `[investigator: done]`、规划师 `[planner: done]`、编码员 `[coder: done]`、测试员 `[tester: done]`、评审员 `[reviewer: done]`、文档员 `[writer: done]`、调试员 `[debugger: done]`。主智能体收到标记即知任务完成，未收到说明子智能体提前退出或超时。

### 评审员的专项输出格式

```
总体结论：Approve / Approve with minor suggestions / Request changes
阻断问题（blocking）：
- 文件路径 | 严重度 | 描述 | 修复建议
建议（warning / suggestion）：
- ...
```

## 主智能体的职责

1. **拆任务**——把需求拆成可独立验证的块
2. **选角**——每块从角色库选一个角色
3. **委派**——用五段式格式 + 角色定义，一律 `run_in_background: true` 异步派出去（拿 task_id 立刻走，不 await、不 poll）
4. **等回传**——收到全部 `<notification>` 前不决策；每条通知按 `source_id` 关联回对应任务
5. **汇报**——汇总给用户：谁干了什么、验证是否通过、遗留问题

主智能体绝不自己编辑文件、跑命令。它只有读工具 + Agent 委派工具 + 规划能力。

## 委派任务格式（五段式，必须写全）

```
### 背景
任务来源、已完成上下文、为什么做这个

### 输入
相关文件路径、数据、已有结论

### 要求
明确的任务清单，可逐项检查；划清边界（哪些文件不许碰）

### 输出格式
期望返回的结构（按所选角色的输出格式）

### 验收标准
- [ ] 具体检查项（含验证命令）
```

## 流水线示例（完整任务如何走多个角色）

以"把认证中间件重构为 async/await"为例：

1. 派**调研员**（后台）：找出认证中间件位置、现状、依赖、现有测试
2. 收到调研通知 → 派**规划师**（后台）：基于调研结果出重构步骤，列出受影响文件
3. 收到规划通知 → 派**编码员**（后台）：按方案实现重构
4. 收到编码通知 → 派**测试员**（后台）：为重构后的代码补测试并跑通
5. 收到测试通知 → 派**评审员**（后台）：评审改动，给出阻断问题/建议
6. 有阻断问题 → 收到通知后派**编码员**修复 → 再评审；无阻断 → 派**文档员**更新相关文档
7. 主智能体汇总全部回传，向用户汇报

第 3~5 步可视为一条流水线；每个角色独立上下文，只有主智能体掌握全局。流水线的每一步都走「异步派发 → 等 `<notification>` → 派下一步」，主智能体不在任何一步前台等待。

## 并行 vs 串行

- **并行**：多个任务互不依赖、不碰同一批文件 → 同一消息里发多个 `Agent` 调用（全部 `run_in_background: true`，或同角色多输入用 `AgentSwarm`）
- **串行**：后一个任务依赖前一个的结果（如流水线）→ 前一个也异步派发，收到它的 `<notification>` 后再派下一个（用 `Agent(resume=agent_id, ...)` 或新 spawn）
- **禁止**：多个子智能体并发改同一个文件

## 子智能体如何向你提问（问题回传机制）

子智能体看不到你，只能回传结果给主智能体。需要你回答时，按下面的机制把问题"传上来"。

### 子智能体的纪律：该问 vs 不该问

- **该问**：影响任务方向、范围、验收标准、方案取舍的问题（如 tdd 的测试 seams、to-spec 的发布目标、grill-me 拷问出的方案歧义）。
- **不该问**：小歧义、能从代码/上下文推断的事——自己做合理假设并继续，最后在报告里标注"我假设了 X"。

### 提问格式（子智能体遇到必须问的，输出这个块并停止，不得臆造、不得继续）

```
<question>
- 问题：<一句话问题>
- 背景：<为什么需要回答、影响哪个决策>
- 选项（可选）：A. ... / B. ... / C. ...
- 影响：<不回答会怎样>
</question>
```

### 主智能体的转述流程

1. 收到 `<question>` 块。
2. **能自己答就不打扰用户**：如果答案可从已有上下文/项目信息确定，直接答复子智能体，并记入汇报。
3. 不能确定 → **转述给你**。并行场景下先收集所有在途子智能体的问题，合并去重后**一次性**问，减少打断次数。
4. 收到你的回答 → 用 `resume` 恢复对应子智能体，把答案原样注入 prompt，让它继续。
5. 你回答"你自己看着办" → 回传该指令，子智能体按最合理假设执行并在报告里标注。

### 注意

- 交互型技能（如 grill-me）在**主智能体层**直接对你跑更自然——拷问本身是一轮轮问答。委派给子智能体时，让它把问题**批量打包**回传（列成清单），不要一问一停。
- 同一轮转述不超过一批问题；每个问题都要有"影响"说明，方便你判断值不值得回答。

## 异步派发与进度追踪

参考 Kimi Code 的 `Agent` 工具实现（`subagent-host.ts` + `BackgroundManager` + `AgentBackgroundTask`），异步子智能体有三个独立维度：

1. **派发（spawn）**：主智能体发任务出去，立刻拿到 `task_id`（调度层一次性）和 `agent_id`（子智能体实例持久 id）
2. **追踪（track）**：主智能体能查到子智能体的当前状态 / 累计输出
3. **回传（notify）**：子智能体完成 / 失败 / 丢失时，结果自动注入主智能体的下一轮

### 异步派发：拿 task_id 就走（所有派发的默认方式）

本 skill 的**所有**子 agent 委派一律调用 `run_in_background: true`（或你自己的等价标记）异步派发，主智能体立刻拿到返回：

```
task_id: t_001
status: running
agent_id: a_xyz
actual_subagent_type: coder
description: ...
resume_hint: Agent(resume="a_xyz", prompt="...")
```

**主智能体的纪律**：拿到 task_id 后立即返回上层流程，**不要 await 完成、不要 poll**。继续处理用户下一条消息 / 派发其他独立子智能体。

### 回传：`<notification>` 自动注入

子智能体完成（或失败 / 丢失）时，主智能体框架应在其**下一轮 turn 开头**注入一条 `<notification>` 块（参见 Kimi Code 的 `BackgroundTaskNotification` 注入路径）：

```
<notification type="task.completed" source_id="t_001" agent_id="a_xyz">
[子智能体的 summary]
</notification>
```

主智能体看到这条通知，按 `source_id` 关联到之前那个 task，处理结果（汇报给用户 / 衔接下一步流程）。

### 追踪：主智能体怎么"读在途子智能体进度"

**默认不去主动读**子智能体的 transcript。这跟进程内存一样会撞锁、会读到中间态、会让子智能体的 tool-loop 跟你的读互相打架。三个可选：

1. **共享进度文件**：子智能体每一步完成后把"我刚才做了什么 / 下一步要做什么"append 到一个主智能体知道的文件（如 `progress/<agent_id>.md`）。主智能体需要时用 `Read` 主动查——这就是"翻 session 日志"的对应物，但走文件系统，避开内存锁。
2. **resume 续问**：用 `Agent(resume=agent_id, prompt="汇报当前进度")` 把问题塞回去，子智能体读自己的 transcript 回答。这是 Kimi Code 推荐的方式。
3. **TaskOutput(task_id)**（如框架提供）：一次性读子智能体累计输出，**仅在排查异常 / 卡死时用**，正常流程不要走这条。

### resume 用什么 id

**`task_id` ≠ `agent_id`**：

| id | 性质 | 用途 |
|---|---|---|
| `task_id` | 任务调度层（BackgroundManager），子智能体跑完即消亡 | 关联 `<notification>`、查输出 |
| `agent_id` | 子智能体实例持久 id | resume 续上下文、查 profile |

要继续跟子智能体对话 → 用 `Agent(resume="a_xyz", prompt="...")`（agent_id）。
错误地用 task_id resume 会被框架拒绝（"agent not found"）。

## /btw 模式 —— 旁支提问（不是读子智能体进度）

如果用户突然问个旁支问题（跟当前任务无关、又不想打断主任务），派一个**镜像模式**的子智能体（参见 Kimi Code `SessionSubagentHost.startBtw()`）：

- 复制主智能体的 `system_prompt` + 模型 + 工具集（但所有写类工具**全禁**——`DenyAllPermissionPolicy`）
- 共享主智能体的历史（`context.useProjectedHistoryFrom(parent.context)`）——让 btw 能看到主智能体看过的内容
- 加一句 system reminder 告诉它"你是轻量子，只用文本回答，工具都关了"
- btw 答完即丢，不持久化（`InMemoryAgentRecordPersistence`）

**注意**：`/btw` **不是**"主智能体读子智能体在做什么"的机制——它解决的是**用户问旁支问题时不打断主任务**。要读在途子智能体的进度，走上面「异步派发与进度追踪」里的方式。

## 细节纪律

- **委派时给足上下文**：子智能体看不到本会话历史，只拿到你写的那段 prompt——角色定义、路径、背景、要求全部写进去，别让它猜。
- **用 resume 续上下文**：同一智能体的后续步骤（如"给刚才的改动补测试"）用 `resume=agent_id`（**不是 task_id**）恢复原子智能体，保留其上下文，不要重新起一个。
- **异步派发（默认，不再前台等待）**：所有子 agent 一律 `run_in_background: true` 派发，拿 task_id 立刻走。框架会在子智能体完成 / 失败 / 丢失时往主智能体下一轮 turn 开头注入一条 `<notification>`（带 `source_id` 关联回 task）。**主智能体不要 await、不要 poll**；需要结果就等通知到了再衔接，绝不在派发那一步前台阻塞。详见上一节。
- **不重复劳动**：委派出去的任务，主智能体在同一轮里不要再自己搜一遍 / 做一遍，等回传即可。
- **超时处理**：子智能体超时未回 → 用 `TaskOutput` 读累计输出排查，必要时 `TaskStop` 终止或 `Agent(resume=agent_id, ...)` 续跑，别静默重派。
- **btw 不是查询子智能体进度的工具**：用户问"子智能体干到哪了"时不要派 btw，要用 TaskOutput / resume / 共享进度文件。

## 何时不用本 skill

- 小改动（一两行、单文件小修）——直接做，别拆
- 纯对话问答——不需要执行
- 用户明确要求主智能体自己动手的场景
