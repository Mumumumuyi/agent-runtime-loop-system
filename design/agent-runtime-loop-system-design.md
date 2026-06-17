# Agent Runtime Loop System Design

版本: v0.1  
定位: 直接改造 Claude Code / Codex 类 Agent 的底层执行逻辑  
核心目标: 让 Agent 在执行复杂任务时自动形成 "拆解 - 执行 - 自检 - 返工 - 审查 - 再执行 - 完成" 的闭环，而不是依赖人工反复催促。

## 1. 系统定位

这个系统不是 Skill、MCP、Prompt 模板或外部工作流。

它应该被设计为 Agent Runtime 的底层控制层，直接介入 Claude Code / Codex 的任务执行主循环。

也就是说，原本 Agent 的执行方式大致是:

```text
用户输入 -> 模型推理 -> 工具调用 -> 输出结果
```

改造后变成:

```text
用户输入
  -> 目标建模
  -> 方案图生成
  -> 方案节点执行循环
  -> 自动验证
  -> 未通过则返工
  -> 全部通过后启动并行审查组
  -> 主 Agent 裁决
  -> 不通过则生成新方案组继续循环
  -> 通过则最终汇报并请求人工检查
```

Skill 和 MCP 不再是系统主体。

它们只是底层 Runtime 在某个方案节点里可以调用的能力:

```text
Runtime Loop System = 核心操作系统
Skill = 可调用的专项能力
MCP = 可调用的外部工具接口
Browser/Shell/Git = 可调用的执行工具
```

## 2. 为什么要做底层改造

现在的 Agent 常见问题是:

1. 单轮推理过强依赖一次性判断。
2. 计划写得很好，但执行时不会持续对照验收标准。
3. 测试失败后容易局部修补，而不是重新拆解失败原因。
4. 完成判断经常由 Agent 自己凭感觉给出。
5. 审查和执行混在同一个上下文里，容易自我确认。
6. 长任务需要用户不断追问、提醒、复核。

这个系统的目标是把 Agent 从 "一次性回答器" 改造成 "目标驱动的循环执行系统"。

## 3. 核心原则

### 3.1 目标优先

Agent 开始执行前必须先生成明确的目标对象。

目标对象包含:

- 用户原始需求
- Agent 理解后的任务目标
- 明确的验收标准
- 不允许改变的硬约束
- 需要人工授权的风险动作
- 预算限制
- 停止条件

Agent 可以在执行中细化验收标准，但不能擅自降低验收标准。

### 3.2 方案图驱动

复杂任务不能只有线性 TODO。

Runtime 应该生成方案图 Plan Graph:

- 每个节点是一个可执行方案
- 节点之间有依赖关系
- 节点有状态
- 节点有验收标准
- 节点有工具权限
- 节点有失败返工路径

### 3.3 每个方案节点必须自带闭环

每个方案节点都执行:

```text
准备 -> 执行 -> 运行验证 -> 对照验收 -> 通过/失败 -> 更新状态
```

如果失败:

```text
失败原因分析 -> 重新拆解 -> 更新当前方案节点 -> 再执行
```

### 3.4 审查必须和执行分离

执行 Agent 不应该独自决定最终完成。

方案组完成后，Runtime 创建最多 3 个审查 Agent，并行检查:

- 需求覆盖
- 实现质量
- 测试与回归风险

审查 Agent 只输出报告，不直接修改结果。

主 Agent 负责汇总报告并裁决:

- 是否真的阻碍最终目标
- 是否需要返工
- 是否形成新的方案组
- 是否可以最终结束

### 3.5 防止无限循环

系统必须有硬性停止条件:

- 达成最终验收标准
- 连续多轮失败且无新信息
- 超出时间/工具/成本预算
- 触发安全或权限边界
- 需要用户提供外部信息

## 4. 底层架构

```text
+------------------------------------------------------+
|                  Agent Runtime                       |
+------------------------------------------------------+
|  User Request Intake                                 |
|  Goal Compiler                                       |
|  Plan Graph Builder                                  |
|  Loop Orchestrator                                   |
|  Tool Execution Layer                                |
|  Verification Engine                                 |
|  Rework Engine                                       |
|  Review Swarm Manager                                |
|  Final Judge                                         |
|  Memory / Trace / State Store                        |
+------------------------------------------------------+
```

### 4.1 User Request Intake

负责接收用户输入，并判断任务类型。

输出:

```json
{
  "request_id": "task_001",
  "raw_user_request": "...",
  "task_type": "code_change | document | research | debugging | architecture | mixed",
  "risk_level": "low | medium | high",
  "requires_clarification": false
}
```

### 4.2 Goal Compiler

把用户自然语言转换成可执行目标。

输出:

```json
{
  "goal_id": "goal_001",
  "objective": "构建可交互的系统设计文档",
  "acceptance_criteria": [
    "产出文件位于用户桌面",
    "内容明确说明这是底层 Runtime 改造",
    "不把 Skill/MCP 描述为系统本体",
    "包含架构、状态机、执行循环、审查机制和停止条件"
  ],
  "hard_constraints": [
    "不得删除用户文件",
    "不得擅自执行高风险外部操作"
  ],
  "human_approval_required_for": [
    "安装软件",
    "删除重要数据",
    "提交外部服务",
    "发送敏感信息"
  ],
  "stop_conditions": [
    "全部验收标准满足",
    "遇到权限阻塞",
    "连续返工达到上限",
    "预算耗尽"
  ]
}
```

### 4.3 Plan Graph Builder

将目标拆成方案图。

节点示例:

```json
{
  "node_id": "plan_01",
  "title": "分析目标与边界",
  "depends_on": [],
  "status": "pending",
  "acceptance_criteria": [
    "识别用户真正想改造的是 Agent Runtime",
    "明确 Skill/MCP 只是可调用能力"
  ],
  "allowed_tools": ["filesystem_read", "shell", "browser"],
  "max_rework_count": 3
}
```

方案图不是固定不变的。

如果某个节点失败，Rework Engine 可以把失败原因插回方案图:

```text
plan_02 执行失败
  -> 新增 plan_02a: 修正数据结构
  -> 新增 plan_02b: 补充测试策略
  -> plan_02 等待 plan_02a 和 plan_02b 完成后重新执行
```

### 4.4 Loop Orchestrator

这是系统核心。

它负责决定下一个动作:

```text
while task_not_done:
    current_node = select_next_plan_node()
    execute_node(current_node)
    verification = verify_node(current_node)

    if verification.passed:
        mark_node_complete(current_node)
    else:
        rework_node(current_node, verification.failures)

    if all_plan_nodes_complete:
        review_reports = run_review_swarm()
        final_decision = judge(review_reports)

        if final_decision.passed:
            finish_task()
        else:
            create_new_plan_group(final_decision.required_fixes)
```

它不只是调工具，而是持续维护任务状态。

### 4.5 Tool Execution Layer

工具层只负责执行，不负责判断最终完成。

可调用工具包括:

- Shell
- 文件系统
- Git
- Browser
- 测试框架
- 编译器
- Skill
- MCP
- 其他本地或远程工具

关键原则:

```text
Tool is capability.
Runtime is authority.
```

工具可以提供能力，但不能替代 Runtime 做最终判断。

### 4.6 Verification Engine

验证引擎负责把 "做了" 变成 "有证据证明做成了"。

验证类型:

- 文件存在性检查
- 单元测试
- 集成测试
- 编译检查
- 类型检查
- 浏览器真实渲染检查
- 运行日志检查
- 截图检查
- 需求覆盖检查
- 用户验收标准对照检查

输出:

```json
{
  "node_id": "plan_03",
  "passed": false,
  "evidence": [
    "npm test failed",
    "browser screenshot shows overlapping panel"
  ],
  "failures": [
    {
      "type": "visual_overlap",
      "severity": "medium",
      "description": "右侧审查卡片在 1440x950 视口下贴近底部时间线"
    }
  ]
}
```

### 4.7 Rework Engine

返工引擎不只是重试。

它必须回答三个问题:

1. 没有达标的具体点是什么?
2. 为什么没有达标?
3. 应该拆成哪些新的可执行子问题?

返工策略:

```text
失败 -> 分类 -> 定位原因 -> 生成修复子节点 -> 更新方案图 -> 重新执行
```

失败分类:

- 需求理解错误
- 方案拆解不足
- 工具调用失败
- 实现 bug
- 测试不足
- 环境问题
- 权限问题
- 目标不可达

### 4.8 Review Swarm Manager

审查组最多 3 个 Agent。

推荐角色:

```text
Reviewer A: Requirement Coverage
Reviewer B: Implementation Quality
Reviewer C: Risk, Regression, and Safety
```

审查 Agent 的限制:

- 不直接修改文件
- 不重写主方案
- 只输出结构化审查报告
- 必须引用证据
- 必须区分阻塞问题和建议问题

审查报告格式:

```json
{
  "reviewer": "Requirement Coverage",
  "passed": false,
  "blocking_findings": [
    {
      "title": "缺少停止条件定义",
      "evidence": "文档没有说明循环何时停止",
      "required_fix": "补充硬停止条件和预算限制"
    }
  ],
  "non_blocking_suggestions": [
    "可以增加状态机图"
  ]
}
```

### 4.9 Final Judge

主 Agent 汇总审查报告。

它不能机械接受所有建议。

裁决逻辑:

```text
如果存在阻塞问题:
    分析阻塞问题是否真实影响最终目标
    如果真实影响:
        生成新方案组
        返回 Loop Orchestrator
    如果不影响:
        记录忽略原因
        继续判断

如果没有阻塞问题:
    检查最终验收标准
    如果全部满足:
        结束任务
    否则:
        生成补充方案组
```

## 5. 状态机设计

任务级状态:

```text
CREATED
  -> GOAL_COMPILED
  -> PLAN_BUILT
  -> EXECUTING
  -> VERIFYING
  -> REWORKING
  -> REVIEWING
  -> JUDGING
  -> COMPLETED
  -> NEEDS_HUMAN
  -> BLOCKED
  -> FAILED
```

节点级状态:

```text
PENDING
  -> READY
  -> RUNNING
  -> VERIFYING
  -> PASSED
  -> FAILED
  -> REWORKING
  -> BLOCKED
```

审查状态:

```text
NOT_STARTED
  -> RUNNING
  -> REPORT_SUBMITTED
  -> ACCEPTED_BY_JUDGE
  -> REJECTED_BY_JUDGE
```

## 6. 核心执行循环伪代码

```python
def run_agent_task(user_request):
    task = intake(user_request)
    goal = compile_goal(task)
    plan_graph = build_plan_graph(goal)

    while True:
        if should_stop(task, goal, plan_graph):
            return stop_with_reason(task)

        node = select_next_ready_node(plan_graph)

        if node is None:
            if plan_graph.all_complete():
                reports = run_review_swarm(goal, plan_graph)
                decision = final_judge(goal, plan_graph, reports)

                if decision.passed:
                    return complete_task(goal, plan_graph, reports)

                new_nodes = build_rework_plan_group(decision.failures)
                plan_graph.add_nodes(new_nodes)
                continue

            return block_task("No executable node available")

        result = execute_node(node)
        verification = verify_node(node, result, goal)

        if verification.passed:
            plan_graph.mark_passed(node.id, verification.evidence)
            continue

        if node.rework_count >= node.max_rework_count:
            return block_task("Rework limit reached", node, verification)

        rework_nodes = decompose_failures_into_nodes(node, verification.failures)
        plan_graph.replace_or_extend(node.id, rework_nodes)
```

## 7. 底层 Prompt 与 Runtime 的关系

这个系统不能只靠系统提示词实现。

系统提示词只能规定行为倾向，但不能可靠提供:

- 持久状态
- 多 Agent 并行
- 任务图维护
- 自动重试上限
- 工具调用证据存储
- 跨轮返工追踪
- 审查报告裁决

所以需要同时改造:

1. System Prompt
2. Agent Runtime Loop
3. Tool Scheduler
4. State Store
5. Review Agent Manager
6. Verification Engine

System Prompt 的作用是约束 Agent 思考方式。

Runtime 的作用是强制执行循环。

## 8. 和 Skill / MCP 的边界

### 8.1 Skill 不是系统本体

Skill 是某类任务的专家流程。

例如:

- TDD Skill
- Diagnose Skill
- Frontend Design Skill
- Browser Testing Skill

它们可以被某个方案节点调用。

但如果没有 Runtime Loop，它们仍然只是一次性的辅助流程。

### 8.2 MCP 不是系统本体

MCP 是工具接口协议。

例如:

- GitHub MCP
- Browser MCP
- Supabase MCP
- Figma MCP

它们提供能力，但不负责:

- 任务拆解
- 验收判断
- 返工循环
- 审查裁决
- 停止条件

这些必须在 Agent Runtime 里实现。

## 9. 人工介入机制

系统不是完全不需要人，而是不需要人在每一步反复催促。

需要人工介入的情况:

- 目标不清且无法安全假设
- 需要账号授权
- 需要发送敏感信息
- 需要删除重要数据
- 需要安装软件或改变系统设置
- 预算即将超限
- 多轮返工后仍无法达标
- 需要用户主观审美判断

人工介入点应该被 Runtime 明确标记:

```json
{
  "status": "NEEDS_HUMAN",
  "reason": "需要确认是否允许删除指定目录",
  "options": [
    "允许删除",
    "只做只读分析",
    "停止任务"
  ]
}
```

## 10. 日志与可追溯性

每次循环都应该记录:

- 当前目标
- 当前方案图
- 当前节点
- 执行动作
- 工具调用
- 输出证据
- 验证结果
- 失败原因
- 返工方案
- 审查报告
- 主裁决原因

日志不是给模型看的流水账，而是给系统恢复、审查和用户复盘用的证据链。

## 11. 最小可行实现

如果要先做 MVP，建议改造顺序是:

1. 增加 Goal Compiler
2. 增加 Plan Graph 数据结构
3. 增加节点级执行状态
4. 增加 Verification Engine
5. 增加 Rework Engine
6. 增加 Review Swarm Manager
7. 增加 Final Judge
8. 增加持久化 State Store

MVP 可以先只支持单主 Agent + 本地审查模拟。

然后再升级为真正的多 Agent 并行审查。

## 12. 推荐数据结构

### 12.1 TaskState

```json
{
  "task_id": "task_001",
  "status": "EXECUTING",
  "goal": {},
  "plan_graph": {},
  "current_node_id": "plan_03",
  "loop_count": 4,
  "created_at": "2026-06-16T00:00:00Z",
  "updated_at": "2026-06-16T00:05:00Z"
}
```

### 12.2 PlanNode

```json
{
  "id": "plan_03",
  "title": "执行方案节点",
  "description": "...",
  "depends_on": ["plan_01", "plan_02"],
  "status": "RUNNING",
  "acceptance_criteria": [],
  "allowed_tools": [],
  "evidence": [],
  "failures": [],
  "rework_count": 1,
  "max_rework_count": 3
}
```

### 12.3 VerificationResult

```json
{
  "node_id": "plan_03",
  "passed": true,
  "criteria_results": [
    {
      "criterion": "文件存在于桌面",
      "passed": true,
      "evidence": "C:\\Users\\amumu\\Desktop\\agent-runtime-loop-system-design.md exists"
    }
  ],
  "failures": []
}
```

### 12.4 ReviewReport

```json
{
  "reviewer_id": "reviewer_a",
  "role": "Requirement Coverage",
  "passed": true,
  "blocking_findings": [],
  "non_blocking_suggestions": []
}
```

## 13. 关键实现难点

### 13.1 完成标准不能太虚

如果验收标准是 "做得好一点"，Loop 会很难停止。

必须转成可检查标准:

```text
错误: 做一个好看的页面
正确: 页面能打开、首屏非空白、核心按钮可点击、移动端不重叠、无控制台错误
```

### 13.2 审查 Agent 可能产生噪音

审查 Agent 经常会提出 "可以更好" 的建议。

Final Judge 必须区分:

- 阻塞最终目标的问题
- 非阻塞优化建议
- 与目标无关的偏好

### 13.3 工具失败和任务失败要分开

工具调用失败不等于任务失败。

例如 Browser 插件失败，可以切到 Chrome 或 Playwright 验证。

Runtime 应该记录:

```text
primary_tool_failed -> fallback_tool_used -> verification_completed
```

### 13.4 不能让循环绕过安全边界

底层循环不能成为自动越权机制。

越是底层系统，越要强制:

- 权限检查
- 高风险操作暂停
- 用户授权
- 审计日志
- 敏感数据保护

## 14. 最终效果

改造完成后，Agent 的行为应该从:

```text
我理解了 -> 我做了 -> 你看看
```

升级为:

```text
我理解目标
我拆成方案图
我逐个执行
我自己验证
失败则拆解返工
全部通过后启动审查组
审查不过则生成新方案组继续循环
最终满足验收后向你汇报并请求人工检查
```

这就是这个系统的本质:

```text
不是更长的 Prompt。
不是更多的 Skill。
不是更多的 MCP。

而是把 Agent 的底层执行模型改造成一个可验证、可返工、可审查、可停止的循环操作系统。
```

