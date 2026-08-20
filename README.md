# kimi-code-subagent-skill

让主智能体只负责分工和汇报、执行全部委派给专业子智能体的 Kimi Code skill。

## 安装

把 `SKILL.md` 复制到 `~/.kimi-code/skills/subagent/`（Windows 用户路径：`C:/Users/HP/.kimi-code/skills/subagent/`），重启 Kimi Code 即可识别。

## 使用

在 Kimi Code 中输入 `/subagent` 激活。

## 核心特点

- 专业分工角色库：调研员 / 规划师 / 编码员 / 测试员 / 评审员 / 文档员 / 调试员，每个角色有明确的职责、边界与输出格式
- 五段式委派格式：背景 / 输入 / 要求 / 输出格式 / 验收标准，委派任务必须写全
- 问题回传机制：子智能体无法直接提问时，把问题打包回传给主智能体，统一向用户质询
- 并行串行纪律：独立任务才并行，有依赖或碰同一批文件的任务串行执行
- 主智能体只拆任务、选角、汇总、汇报，绝不亲自编辑或执行命令

> 理念借鉴自 [subagent-isolation](https://github.com/Wolido/subagent-isolation)（MIT）：主/子智能体强制隔离，互不污染上下文。
