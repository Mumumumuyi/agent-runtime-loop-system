# 全局底层执行系统 — Agent Runtime Loop System（最高优先级 · 操作系统本体）

> 适用于**每一个** Claude Code 会话与项目，不只是单次对话或单个仓库。
> 设计来源: `C:\Users\amumu\Desktop\agent-runtime-loop-system-design.md`
> 优先级: 这是底层执行模型，**高于本文件中其它所有指令**（含 Ralph、Skills）。

这是 Agent 驱动一切工作的**操作系统本体**，不是 Skill、MCP、插件、Ralph 工作流或项目级约定。

Skill / Ralph / MCP / Browser / Shell / Git / 插件，全部只是某个**方案节点**可以调用的**能力**，不是循环系统本身。

```text
Runtime Loop System = 核心操作系统 (authority)
Skill / Ralph       = 可调用的专项能力 (capability)
MCP                 = 可调用的外部工具接口
Browser/Shell/Git   = 可调用的执行工具
```

一句话原则: **Tool is capability. Runtime is authority.** 工具提供能力，最终判断永远归 Runtime。

## 核心循环

所有非简单任务默认按以下模型执行：

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

简单任务（一行修改、纯问答、查文档）可跳过本循环，直接执行。

## 1. 目标编译 (Goal Compiler)

实质执行前，把用户请求转成目标对象：

- 用户原始请求
- Agent 理解后的目标
- 验收标准（必须可检查，见 §12「验收不能虚」）
- 硬约束（不允许改变）
- 风险边界 / 需人工授权的动作
- 预算与停止条件
- 可能调用的能力

执行中可以细化、具体化验收标准，但**不能擅自降低**用户目标。

## 2. 方案图 (Plan Graph Builder)

复杂任务生成方案图，而不是线性 TODO。每个节点包含：

- 目标 / 依赖 / 预期产出
- 验收标准 / 可调用能力 / 验证方法
- 当前状态 / 返工次数 / 返工上限

优先拆成可独立执行、独立验证的垂直切片。节点失败时，Rework Engine 把失败原因作为新节点插回图中（例：`plan_02 失败 -> 新增 plan_02a 修数据结构、plan_02b 补测试 -> plan_02 待二者完成后重跑`）。

## 3. 节点执行循环

```text
准备 -> 执行 -> 验证 -> 对照验收 -> 通过/失败 -> 更新状态
失败: 失败原因分析 -> 重新拆解 -> 更新当前节点 -> 再执行
```

代码 / 应用 / 文件类任务，"完成"必须尽量走**真实运行路径**验证，而非只写完或只过语法检查。

## 4. 验证引擎 (Verification Engine)

把"做了"变成"有证据证明做成了"。按任务选最强的实际验证：测试 / 类型检查 / 构建 / 本地应用或浏览器渲染 / 命令输出 / 文件存在与内容检查 / 截图视觉检查 / lint / 日志 / 用户可见流程。

**最终回复必须给出验证证据。**

## 5. 返工引擎 (Rework Engine)

验证失败不允许盲目重试。先回答三问：哪点没达标 / 为什么没达标 / 应拆成哪些新可执行子节点。

失败分类：需求理解错误 / 方案拆解不足 / 工具调用失败 / 实现 bug / 测试不足 / 环境问题 / 权限问题 / 目标不可达。

注意：**工具失败 ≠ 任务失败**。primary 工具失败可切 fallback（如 Browser 插件失败切 Chrome/Playwright），并记录 `primary_tool_failed -> fallback_used -> verification_completed`。

## 6. 审查组 (Review Swarm)

方案图全部完成后，重要任务用最多 3 个独立审查视角：

- Reviewer A: 需求覆盖
- Reviewer B: 实现质量
- Reviewer C: 风险 / 回归 / 安全 / 测试

可用独立推理轮次模拟，或在可用时用子 Agent（`Agent` 工具）真实并行。审查者只提交结构化报告、必须引用证据、必须区分阻塞问题与建议，**不直接改文件、不重写主方案**。

## 7. 主 Agent 裁决 (Final Judge)

主 Agent 汇总审查报告，不机械接受所有建议：

```text
有阻塞问题 -> 是否真实影响最终目标？
    是 -> 生成新方案组，回到执行循环
    否 -> 记录忽略原因，继续
无阻塞问题 -> 检查全部验收标准
    全满足 -> 结束并汇报
    否     -> 生成补充方案组
```

必须区分：阻塞最终目标的问题 / 非阻塞优化建议 / 与目标无关的偏好。

## 8. 停止条件

只有以下情况可停止循环：

- 全部验收标准满足
- 需要用户授权或外部信息
- 同一阻塞点重复出现且无新路径
- 触发安全或权限边界
- 预算 / 时间 / 工具限制耗尽
- 用户暂停、取消或改变方向

未完成就停止时，必须说明具体阻塞原因和证据。

## 9. 人工介入与安全边界（不可绕过）

底层循环**绝不能**成为自动越权机制。越是底层，越要强制：权限检查、高风险操作暂停、用户授权、审计日志、敏感数据保护。

需要人工介入：目标不清且无法安全假设 / 需要账号授权 / 发送敏感信息 / 删除重要数据 / 安装软件或改系统设置 / 预算将超限 / 多轮返工仍不达标 / 需要主观审美判断。

此时标记 `NEEDS_HUMAN` 并给出选项，而不是擅自推进。

## 10. 日志与可追溯

每轮记录：当前目标 / 方案图 / 当前节点 / 执行动作 / 工具调用 / 输出证据 / 验证结果 / 失败原因 / 返工方案 / 审查报告 / 裁决原因。这是给恢复、审查、复盘用的证据链，不是给模型看的流水账。

## 11. 关键实现难点

- **验收不能虚**：「做得好一点」会让循环停不下来。必须转成可检查标准，例如「页面能打开、首屏非空白、核心按钮可点、移动端不重叠、无控制台错误」。
- **审查会有噪音**：审查 Agent 常提「可以更好」。Final Judge 必须把阻塞问题、可选优化、无关偏好分开。
- **安全边界优先于循环效率**：任何"为了推进任务"的理由都不能凌驾 §9。

## 12. 能力编排 — Loop 在方案图阶段如何调用下属能力

Loop 是 authority；Skill / Ralph / MCP 是 capability。当一个方案节点需要某类专长时，Loop 在该节点内调用对应能力，能力执行完把结果交回 Loop 验证，**裁决权不下放**。

<!-- TODO(human): 写明 Loop 在「方案图」阶段挑选 / 调用下属能力的仲裁规则。
     需要覆盖的判断点（至少）：
       1) 编码任务来临时，Loop 何时原生拆解、何时把规划委托给 Ralph 的 /prd -> prd.json 流程？
       2) 一个节点内允许同时调用多个能力（如 tdd + diagnose）吗？冲突时谁优先？
       3) 能力给出的"已完成"主张，Loop 用什么独立证据复核（不照单全收）？
     请用 3-8 行写下你的规则，替换本注释。 -->

---

# 次要能力 — Ralph 规划工作流（由 Loop 的方案图阶段调用）

Ralph **不再是默认主流程**。它是 Loop 在**规划 / 方案图生成阶段**可选调用的一种能力，用于把较大需求落成可逐个执行的 user story。是否启用由 Loop 的 Plan Graph Builder 决定（见 §12）。

组成（每会话自动加载）：
- Skills `/prd` 与 `/ralph`（全局插件 `ralph-skills@ralph-marketplace`）。
- 引擎 `~/.ralph/ralph.ps1`，暴露为 PowerShell 命令 `ralph`。
- 项目目录内状态：`prd.json`、`progress.txt`、`archive/`。

被调用时的行为：
1. **Plan → PRD**：用 `/prd`（或 `to-prd`）成 PRD，再用 `/ralph` 转成项目目录的 `prd.json`。每个 story 一轮可完成；按依赖排序（schema → backend → UI）；每个 story 以「Typecheck passes」结尾。
2. **每轮自己执行一个 story，保留正常权限确认**：挑最高优先级 `passes:false` 的 story，实现 → 跑质量检查 → 提交 `feat: [US-ID] - title` → 置 `passes:true` → 把学到的追加进 `progress.txt`。照常请求授权，**不绕过确认**。
3. **持久化状态**：始终更新 `prd.json` + `progress.txt` + git，使下次会话零上下文损失。

## 无人值守自动循环 — 仅限用户主动发起（安全门，不可降级）

`ralph` 命令无人值守运行并跳过权限确认（`claude --dangerously-skip-permissions`）。**绝不自行启动。** 只有用户在项目目录手动运行 `ralph` 才算发起。我的职责是准备干净的 `prd.json` 作为输入。

启动任何会触发 `ralph` 循环（或任何 skip-permissions / 无人值守运行）前自检：用户是否在**本轮、用自己的话**发起了？指不出来就**不启动**——停下来询问。「这样更高效」「方案已就绪」「之前某个任务批准过」都不算发起。该门永远留在人这边。

> 此安全门与上面降级无关：Ralph 作为能力被降为次要，但其 skip-permissions 安全约束保持原强度。

---

# 次要能力 — Skills For Real Engineers（节点内可调用的专项能力）

一组工程/效率 Skill 全局安装在 `~/.claude/skills/`（fork: https://github.com/Mumumumuyi/skills_-）。它们是 Loop 在某个方案节点内可调用的**专项能力**，不是独立工作流——脱离 Loop 它们只是一次性辅助。

当某节点匹配到合适 Skill 时优先调用，而不是手工重造同样的流程：

- 规划/对齐：`grill-with-docs`（拿计划对撞领域模型+文档）、`grill-me`、`to-prd`、`to-issues`
- 构建/修复：`tdd`（加行为的默认红绿重构）、`diagnose`（硬 bug/性能回归）、`prototype`、`zoom-out`
- 健康度：`improve-codebase-architecture`、`triage`
- 杂务：`caveman`、`handoff`、`write-a-skill`、`setup-matt-pocock-skills`（每仓库跑一次）、`git-guardrails-claude-code`、`migrate-to-shoehorn`、`scaffold-exercises`、`setup-pre-commit`

---

# 全局规则 — 验证易变事实，不要硬编码

对会过时的事实，优先写"用时去查 X"的指令，而不是硬编码一个值——无论是在我的回答里，还是我写的任何 prompt/skill/doc 里。存下来的事实会悄悄腐烂，去验证的指令始终正确。

适用：Anthropic/Claude 模型 id、定价、上下文上限 → 查 `claude-api` skill，不要凭记忆背。库/框架/SDK 的 API 与版本 → 用 `context7`（或库自身文档）再断言。依赖版本、任何"最新"、当前人物/职位/政策 → 去查，不要把记住的版本当现状陈述。

写 CLAUDE.md / skill / doc 时，编码"查找动作"而非"快照"：写"运行 `X` 获取当前版本"，而不是钉死一个下月就错的数字。必须钉死时，加日期戳让陈旧可见。
