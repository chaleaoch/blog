---
title: "Harness Engineering：从 Agent Loop 到完整 Agent Runtime"
publish_time: "2026-08-28"
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">本文整理自《最近折腾 Harness 的一些感受和思考》，梳理从最小 Agent Loop 到完整 Agent Runtime 的演进路径。</p>

## 1. Harness 到底是什么？

最简单的 Agent，本质上就是一个 Loop：

```python
while not finished:
    response = LLM(context, tools)

    if response contains tool_call:
        result = execute(tool_call)
        context += result
    else:
        return response
```

模型负责：

```text
理解任务 → 决定下一步 → 调用工具 → 观察结果 → 继续判断
```

但真正做一个可用的 Agent 时，问题很快就会出现在 Loop 之外：

- 工具怎么执行？在哪里执行？有没有权限？失败怎么办？
- 状态怎么保存？任务中断后怎么恢复？上下文太长怎么办？
- 文件放在哪里？多个 Agent 怎么协作？
- 怎么观察 Agent 为什么跑偏？怎么评估一次修改是否真的更好？
- 长时间任务怎么调度？

因此可以把 Harness 理解为：**位于 Model 与真实执行环境之间，负责让 Model 能够稳定执行任务的一整套运行时能力。**

一个比较清晰的划分是：

```text
                Agent
                  │
          ┌───────┴───────┐
          │               │
        Model           Harness
          │               │
     思考 / 推理          执行环境
     判断 / 规划          Tool
                         Skill
                         Sandbox
                         State
                         Memory
                         Context
                         Permission
                         Artifact
                         Trace
                         Scheduling
                         ...
```

因此可以理解为：

> Agent = Model + Harness

- **Model** 决定 Agent 的推理能力上限。
- **Harness** 决定这些推理能力能够有多少真正转化成可执行能力。

## 2. 只有一个 Shell / Execute

最开始，可以只给 Agent 一个 `execute(command)`，例如 `grep`、`find`、`sed`、`git`、`python`、`npm`、`curl`、`wget`。

一个 Shell 实际上已经提供了非常强的组合能力。理论上，一个 `execute` 几乎可以操作整个计算环境——这也是为什么 Coding Agent 可以仅依赖 Shell + 文件编辑能力完成大量任务。

### 2.1 问题：安全

如果模型拥有 `rm`、`ssh`、`curl`、`wget`、`git`、`pip`、`npm`，那么它实际上拥有非常大的执行权限。因此必须引入 **Sandbox**：

```text
Agent
  │
  ▼
Sandbox
  │
  ├── 文件系统隔离
  ├── 网络限制
  ├── CPU / Memory 限制
  ├── 进程隔离
  └── 权限限制
```

### 2.2 轮次太多，导致注意力偏移

```text
Model → Shell → Result → Model → Shell → Result → Model
```

多轮 Shell 调用会导致 token 消耗过多、注意力偏移。

## 3. Structured Tool

Shell 虽然能力强，但有一个明显问题：很多确定性的事情，本来不应该让 LLM 每次自己推理。

例如 `sed -n '1,200p' xxx`。如果直接让模型操作 Shell，它需要自己处理：文件是否存在、是否太大、如何分页、如何截断、编码、路径、错误处理。

这些其实都是确定性的工程问题。因此开始把 Shell 能力封装成结构化 Tool：`read_file()`、`search()`、`edit_file()`、`download()`、`browser()` ……

例如：

```python
read_file(
    path="config.yaml",
    start_line=1,
    end_line=100
)
```

Harness 可以自动处理分页、截断、编码、异常、路径、权限；Model 只需要表达：**我要读取 config.yaml**。

核心原则：**能确定性解决的问题，就不要让 LLM 每次重新推理。**

### 3.1 Tool 带来的另一个能力：Permission

当所有操作都是结构化 Tool 后，就可以在 Tool 执行前增加拦截层：

```text
LLM → Tool Call → Permission / Hook
                    │
              允许？
              ├── Yes → Execute
              └── No  → Reject / Ask User
```

例如 `delete_database()` 可以在执行前走 Permission Check → 需要人工确认 → Approve / Reject。

相比于 Prompt 里写「请不要执行危险操作」，准确性更高，也更安全。

### 3.2 Tool 太多怎么办？

Skill 的分治策略可以通过索引解决 tool 过多、不知道用哪个的问题，但频繁调用 skill 也会导致 token 消耗过多、注意力偏移。

于是可以让 LLM 生成一段编排代码：

```text
LLM → 生成 orchestration logic
        → Search → Read → Filter → Batch → Retry → Dedup
        → 整理结果 → LLM
```

例如：

```python
files = search("foo")

results = []

for file in files[:10]:
    content = read_file(file)
    if matches_condition(content):
        results.append(content)

return results
```

这里 LLM 不再参与每一次 `read(file1)`、`read(file2)`、`read(file3)`。

- **LLM 负责**：决定应该执行什么逻辑。
- **程序负责**：高效执行确定性的逻辑。

## 4. LLM 和程序的职责边界

这是整个 Harness 设计中非常重要的一条原则。

**LLM 负责不确定性**，例如：用户真正想解决什么？这个错误意味着什么？应该选择哪个方案？下一步应该调查什么？当前结果是否足够？应该如何修改代码？

**程序负责确定性**，例如：循环、排序、过滤、并发、Retry、Backoff、Batch、Parse、Dedup、Cache、权限检查、资源限制。

## 5. Context Management

任务执行几十轮以后：

```text
Context → 越来越长 → 超出窗口
```

于是需要 Context Builder、Compaction、Context Management。

## 6. Checkpoint / Resume

任务执行到一半：

```text
Agent → Tool → Process Crash
```

如果没有持久化，任务全部丢失。于是需要 Checkpoint / Resume：

```text
Task → Step 1 → Step 2 → Checkpoint → Step 3 → Crash

恢复后：Checkpoint → Step 3 → 继续执行
```

## 7. State

Agent 执行过程中，需要维护当前运行状态：当前任务、当前步骤、当前工具调用、当前结果、当前 Agent、当前执行上下文。

这就是 **State**——可以简单理解成 Agent 当前运行时状态。

## 8. Memory

如果任务跨 Session：

```text
Session 1 → Session 结束 → Session 2
```

Agent 可能还需要知道：以前发生过什么、用户过去做过什么、之前形成了什么长期信息。于是产生 **Memory**。

因此三者需要区分：

| 概念 | 含义 |
| --- | --- |
| Context | 当前这一次推理需要看到的信息 |
| State | 当前 Agent 的运行状态 |
| Memory | 跨运行 / 跨 Session 保留的信息 |

这三个概念非常容易混淆，但在长期运行 Agent 中必须进行区分。

## 9. Artifact

Agent 不一定只返回文本。它可能产生代码、HTML、PPT、PDF、图片、数据文件、报告——这些不能简单当成普通 LLM Response 处理。

因此需要 **Artifact Lifecycle**：

```text
生成 → 存储 → 版本管理 → 引用 → 修改 → 再次生成 → 交付
```

## 10. Multi-Agent / Orchestration

当一个 Agent 无法独立完成任务时：

```text
Parent Agent
      │
 ┌────┼────┐
 ▼    ▼    ▼
Agent Agent Agent
```

就会出现 **Agent Orchestration**，需要解决：子 Agent 创建、任务分配、结果传递、状态管理、超时、失败处理、并发、资源限制。

这已经开始明显接近一个 **Agent Runtime / Execution Platform**。

## 11. Trace / Observability

Agent 出现问题时，传统日志往往不够。例如：为什么 Agent 选择了这个 Tool？为什么连续调用了 20 次？为什么突然进入死循环？为什么最后结果错误？

需要记录 Model Input / Output、Tool Call / Result、State Change、Context、Latency、Token、Error。于是出现 **Trace / Observability**。

这也是为什么 Agent 系统通常需要类似 LangSmith / Langfuse 这样的可观测性系统。

## 12. Eval / Regression

Harness 改了以后——Tool、Prompt、Skill、Context、Model 任一变化——Agent 可能昨天能完成、今天突然完成不了了。

因此需要 Eval / Regression Test：

```text
修改 Harness → 运行 Agent Evaluation → 比较历史结果 → 判断是否退化
```

这使 Agent 工程逐渐具备传统软件工程中的 Test、Regression、CI/CD 属性。

## 13. Scheduling / Queue / Resource Governance

如果 Agent 任务只运行几秒，问题不大。但如果一个任务运行 30 分钟甚至几个小时，就需要 Queue、Scheduling、Timeout、Retry、Resource Governance、Concurrency Limit。

例如：

```text
                    Scheduler
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
          Agent A             Agent B
             │                   │
          Sandbox             Sandbox
```

此时 Harness 已经不只是一个 LLM + Tool Loop，而开始具备 Runtime、Scheduler、Executor、State Manager、Resource Manager 等基础设施属性。

## 14. Isolation

多用户、多 Agent 场景还需要进一步解决：User Identity、Permission、Tenant Isolation，以及 Filesystem / Process / Network / Resource Isolation。

例如：

```text
User A → Agent A → Sandbox A
User B → Agent B → Sandbox B
```

不能让 Agent A 访问 Agent B 的数据。因此 Isolation 也是生产级 Harness 的重要组成部分。

## 15. Harness 的演进路线

把整个过程压缩起来：

```text
T0  Shell
 ↓
T1  Sandbox + Structured Tool + Permission
 ↓
T2  Skill + Progressive Disclosure
 ↓
T3  Code Orchestration
 ↓
生产级 Harness
    Context / State / Memory / Checkpoint / Resume
    Artifact / Multi-Agent / Trace / Observability
    Eval / Regression / Scheduling / Queue
    Resource Governance / Isolation
```

## 16. Harness 的核心设计原则

整个演进过程实际上可以归纳成几条原则。

### 原则一：不要让 LLM 做确定性的工作

- **LLM**：理解、判断、规划、推理
- **程序**：循环、过滤、排序、解析、Retry、Cache、权限

### 原则二：不要一次性暴露所有能力

错误方式：500 Tools 全部暴露给 Model。

更合理：Category → Skill → Tool，即 **Progressive Disclosure**。

### 原则三：Tool 不只是能力，也是控制边界

Structured Tool 可以提供 Permission、Hook、Validation、Audit、Retry、Timeout。因此 Tool 层同时承担**能力抽象 + 执行控制**。

### 原则四：模型负责「不确定」，程序负责「确定」

这是整个 Harness 设计中最重要的边界：

```text
                Agent
                  │
       ┌──────────┴──────────┐
       │                     │
  不确定的问题           确定的问题
       │                     │
      LLM                  Code
       │                     │
  判断 / 推理             执行 / 控制
```

## 17. Harness ≠ Tool 集合

一个常见误区是：Harness 就是很多 Tool。不是。Tool 只是 Harness 的一个组成部分。

更完整的结构应该是：

```text
Harness
│
├── Model Integration
├── Agent Loop
├── Tool System
├── Skill System
├── Sandbox
├── Permission
├── Context Management
├── State Management
├── Memory
├── Checkpoint
├── Artifact
├── Orchestration
├── Observability
├── Evaluation
├── Scheduling
└── Isolation
```

所以：**Tool ⊂ Harness**，而不是 Harness = Tool。

## 18. Harness ≠ Agent

也不能简单认为 Agent = LLM。更准确的是：

```text
Agent
│
├── Model
└── Harness
```

其中 Model 负责产生智能，Harness 负责把智能变成实际执行能力。

因此同一个 Model：

```text
Model A
  ├── Harness 1 → Agent 1
  ├── Harness 2 → Agent 2
  └── Harness 3 → Agent 3
```

最终表现可能完全不同。

## 19. 与 LangGraph / DeepAgents 的关系

从工程角度看，可以把它们放到同一张图里理解：

```text
                Agent Application
                       │
                     Agent
                       │
                    Harness
                       │
       ┌───────────────┼────────────────┐
       │               │                │
   Agent Loop       Tool/Skill       Runtime
       │               │                │
   LangGraph        LangChain       Sandbox
   StateGraph       Tools           Backend
   Checkpoint       Middleware      Memory
                                   Artifact
                                   Trace
```

需要注意：**LangGraph、DeepAgents 并不等于 Harness 本身。** 它们提供的是构建 Harness 的不同层次能力。

- **LangGraph** 更偏 State、Graph、Execution、Checkpoint、Interrupt、Resume
- **DeepAgents** 更偏 Agent、Tool、Filesystem / Backend、Subagent、Skill、Long-running Agent

而完整生产系统还可能需要 Sandbox、Permission、Observability、Eval、Scheduling、Resource Governance、Artifact。

所以实际项目中经常是：

```text
DeepAgents / LangGraph
  + Sandbox + Storage + Observability + Eval + Business Logic
  ↓
完整 Harness
```

## 20. 最终理解

如果只记住一句话：

> Harness 就是围绕 Model 构建的一整套 Agent Runtime，让模型负责「想什么、怎么判断」，而系统负责「怎么执行、在哪里执行、能执行什么、失败怎么办、状态怎么保存以及如何继续运行」。

进一步压缩：

```text
Model   = Intelligence
Harness = Execution Infrastructure
Agent   = Intelligence + Execution Infrastructure
```

而 Harness 的演进过程，本质上就是：

```text
让 LLM 能做事
    ↓
让 LLM 安全地做事
    ↓
让 LLM 高效地做事
    ↓
让 LLM 长时间稳定地做事
    ↓
让 LLM 可以被观测、评估、恢复、调度和扩展
```

这也是为什么一个最初只有 LLM + Tool Call 的 Agent，最终很容易逐渐演化成：

```text
Model + Agent Loop + Tool + Skill + Sandbox + Permission
+ Context + State + Memory + Checkpoint + Artifact
+ Orchestration + Trace + Eval + Scheduling + Isolation
```

最终形成的已经不是一个简单的 Agent，而是一套 **Agent Runtime / Harness**。

换一个角度看，Harness 很多时候并不是在给模型「加功能」，而是在把东西从模型身上拿下来：

- 别让模型自己管文件分页
- 别让模型每次重新想 Retry
- 别让模型手动 Batch
- 别让它背几百个 Tool
- 别指望它靠 Prompt 自觉保证安全
- 别让它每次重启以后重新猜之前发生了啥

这些东西程序系统本来就能搞定，那就系统搞定。最后留给模型的，还是那些它最值钱的部分：**理解、判断、推理、规划、创造**。

剩下的脏活，Harness 帮它兜住。

所以 DeepSeek 那句：

> Agent = Model + Harness

现在确实越来越能理解。

- **Model** 决定这个 Agent 理论上能有多聪明。
- **Harness** 决定这份聪明到底有多少能真正变成可用的能力。

同一个 Model，换一个 Harness，最后出来的东西，真的可能完全不像同一个 Agent。

<完>

> 参考: https://linux.do/t/topic/2797040
