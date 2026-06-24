# Agent Runtime Loop System

把 AI 编码 Agent（Claude Code / Codex 类）从 **"一次性回答器"** 改造成 **"目标驱动的循环执行系统"** 的底层执行模型。

> 不是更长的 Prompt，不是更多的 Skill，不是更多的 MCP——
> 而是把 Agent 的底层执行模型改成一个**可验证、可返工、可审查、可停止**的循环操作系统。

---

## 核心思想

原本 Agent 的执行方式：

```text
用户输入 -> 模型推理 -> 工具调用 -> 输出结果
```

改造后：

```text
用户请求
  -> 目标编译 (Goal Compiler)
  -> 方案图生成 (Plan Graph Builder)
  -> 节点执行循环 (Node Execution Loop)
  -> 验证引擎 (Verification Engine)
  -> 返工引擎 (Rework Engine)        [验证失败时]
  -> 审查组 (Review Swarm)           [方案图全部完成后]
  -> 主 Agent 裁决 (Final Judge)
  -> 最终汇报并请求人工检查
```

一句话原则：**Tool is capability. Runtime is authority.**
Skill / Ralph / MCP / Browser / Shell / Git 都只是某个方案节点可调用的**能力**，
最终判断权（拆解、验收、返工、裁决、停止）永远归 Runtime。

---

## 仓库结构

| 路径 | 内容 |
|------|------|
| [`design/agent-runtime-loop-system-design.md`](design/agent-runtime-loop-system-design.md) | **设计文档**：完整规格——架构、状态机、执行循环伪代码、数据结构、实现难点 |
| [`implementation/CLAUDE.global.md`](implementation/CLAUDE.global.md) | **实现**：把上述设计落成可运行的全局 `CLAUDE.md`，作为 Agent 每会话加载的最高优先级指令 |
| `README.md` | 本文件 |

`design` 是"它应该怎样工作"，`implementation` 是"现在如何让 Agent 真的这样工作"。

---

## 如何使用

1. 把 [`implementation/CLAUDE.global.md`](implementation/CLAUDE.global.md) 的内容放进你的**用户级**配置：
   `~/.claude/CLAUDE.md`（Windows: `C:\Users\<你>\.claude\CLAUDE.md`）。
   该文件被每一个 Claude Code 会话、每一个项目自动加载。
2. 这样 Runtime Loop System 成为最高优先级的执行模型；Ralph、Skills 等降为"节点可调用的能力"。
3. 按需填写实现文件 §12「能力编排」里的仲裁规则（Loop 何时把规划委托给 Ralph 等）。

> 诚实边界：这是**行为执行纪律**（每轮加载进上下文并遵守），不是被改写的 Claude Code 二进制运行时。
> 真正的持久状态 / 多 Agent 并行 / 返工计数，用现成能力近似实现：
> 子 Agent 当审查组、Task 列表当方案图状态、git + 进度文件当 State Store。

---

## 设计要点（摘）

- **目标优先**：执行前先编译目标对象（验收标准 / 硬约束 / 停止条件），执行中可细化但不得擅自降低验收标准。
- **方案图驱动**：复杂任务用 Plan Graph 而非线性 TODO，优先可独立验证的垂直切片。
- **每节点自带闭环**：`准备 -> 执行 -> 验证 -> 对照验收 -> 通过/失败 -> 更新状态`。
- **验证要有证据**：把"做了"变成"有证据证明做成了"（测试 / 构建 / 真实运行 / 截图 / 日志）。
- **返工不是重试**：先答"哪点没达标 / 为什么 / 拆成哪些新子节点"。
- **审查与执行分离**：最多 3 个独立审查视角只出报告，主 Agent 裁决，不机械接受建议。
- **硬停止条件 + 安全边界不可绕过**：底层循环绝不能成为自动越权机制。

完整内容见 [设计文档](design/agent-runtime-loop-system-design.md)。

---

## 更新记录

- **v0.2** — 对照 2025–2026 业内 long-running agent harness 实践补三条纪律：
  - §4 **防作弊棘轮**：验证只能新增证据，禁止删/弱化既有测试换"通过"。
  - §10 **追加式日志 + 可重建状态**：新会话仅凭日志+git 即可重建方案图状态。
  - §10 **上下文交接触发**：撞上下文墙或跨会话时先写交接文件再重置，状态外置。
- **v0.1** — 首个版本：目标编译→方案图→节点循环→验证→返工→审查→裁决→停止。

---

版本：v0.2
