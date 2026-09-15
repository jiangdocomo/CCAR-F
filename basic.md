可以。你说的 **CCAR-F = Claude Certified Architect – Foundations**。我按 **2026 年 8 月当前的 Exam Guide v1.0（2026 年 7 月生效）** 来整理。网上有一些较早资料的考试费、Domain 权重甚至题型已经过时，备考时应以当前这版蓝图为准。Anthropic 官方确认 CCAR-F 是其面向生产级 Claude 应用架构的技术认证。([Anthropic][1])

## 一、先记考试结构

| 项目       | 当前考试                         |
| -------- | ---------------------------- |
| 考试代码     | **CCAR-F**                   |
| 题数       | **60 题**                     |
| 时间       | **120 分钟**                   |
| 题型       | 单选 + 多选                      |
| 形式       | **场景题 Scenario-based**       |
| 场景       | 6 个 Scenario 中抽 4 个          |
| 及格分      | **720 / 1000（scaled score）** |
| Domain 数 | 5                            |
| 核心特点     | 考“生产环境下应该怎么设计”，而不是单纯背概念      |

注意：**720 并不等于答对 72%**，因为采用 scaled score。([CCAR-F Academy][2])

### 五大 Domain 权重

| Domain | 内容                                     |      权重 | 约合60题 |
| ------ | -------------------------------------- | ------: | ----: |
| **D1** | Agentic Architecture & Orchestration   | **27%** |   ~16 |
| **D2** | Tool Design & MCP Integration          | **18%** |   ~11 |
| **D3** | Claude Code Configuration & Workflows  | **20%** |   ~12 |
| **D4** | Prompt Engineering & Structured Output | **20%** |   ~12 |
| **D5** | Context Management & Reliability       | **15%** |    ~9 |

所以备考优先顺序我建议：

**D1 → D3 → D4 → D2 → D5**

不是因为 D5 不重要，而是 D1 最值分。([CCAR-F Academy][3])

---

# 二、Domain 1：Agentic Architecture & Orchestration ★★★★★

**27%，全考试最重要。**

这一章的核心不是“Claude 能不能做”，而是：

> **Claude 怎么自主决定下一步，以及程序应该在哪些地方强制控制 Claude。**

包含 7 个 Task Statement。([CCAR-F Academy][4])

### 1. Agentic Loop

这是**必考中的必考**。

基本结构：

```text
User request
    ↓
Claude
    ↓
stop_reason ?
    │
    ├── tool_use
    │       ↓
    │   Execute tool
    │       ↓
    │   Return tool_result
    │       ↓
    └──── Claude again

    └── end_turn
            ↓
           END
```

### 必背

```text
stop_reason == "tool_use"
→ 执行工具
→ 把 tool_result 返回 Claude
→ 继续 loop
```

```text
stop_reason == "end_turn"
→ Claude 已完成
→ 结束 loop
```

Anthropic 官方 API 文档也是这一机制：`tool_use` 表示 Claude 正等待客户端执行工具；自然完成时通常是 `end_turn`。([Claude Platform Docs][5])

### 考试陷阱

看到：

> Run agent exactly 10 iterations.

通常不是最佳答案。

正确思想：

> **stop_reason 控制正常退出。**

Iteration limit 可以作为：

```text
Safety Guard
```

防无限循环，但不是主要的业务终止机制。

---

## 2. Agentic Loop vs Workflow

这个非常容易考。

### Workflow

程序控制流程：

```text
A
↓
B
↓
C
↓
D
```

顺序提前确定。

### Agent

Claude 自己决定：

```text
Claude
 ↓
Choose tool
 ↓
Observe result
 ↓
Decide next action
```

### 判断口诀

> **固定流程 → Workflow**
>
> **下一步需要模型判断 → Agent**

例如：

```text
读取 Invoice
→ Validate
→ DB Insert
→ Send Email
```

流程严格固定：

**Workflow 更合适。**

而：

```text
调查客户投诉
→ 决定查订单还是退款记录
→ 根据结果继续调查
```

更适合：

**Agentic Loop**

---

# 三、Multi-Agent ★★★★★

CCAR-F 非常喜欢考。

典型架构：

```text
              Coordinator
             /     |      \
            /      |       \
      Research   Analysis   Verification
       Agent      Agent       Agent
            \      |       /
             \     |      /
               Coordinator
                    ↓
                Final Result
```

关键思想：

### Coordinator

负责：

```text
任务拆分
分配任务
传递 context
汇总结果
处理失败
最终决策
```

### Subagent

只负责：

```text
一个清晰、有限的任务
```

考试更偏向：

```text
Coordinator
   ↓
Subagent A

Coordinator
   ↓
Subagent B
```

而不是：

```text
Agent A → Agent B → Agent C
```

也就是：

> **Coordinator 是中心。**

([CCAR-F Academy][4])

---

# 四、什么时候应该 Parallel？

高频题。

假设：

```text
Research US market
Research Japan market
Research EU market
```

三者没有依赖：

```text
          Coordinator
          /    |    \
        US    JP    EU
```

应该：

**Parallel execution**

而：

```text
读取文件
↓
分析内容
↓
根据分析生成报告
```

有依赖：

**Sequential**

记忆：

> **Independent → Parallel**
>
> **Dependent → Sequential**

---

# 五、Prompt 不能代替程序控制 ★★★★★

这是整个 CCAR-F 最重要的“出题哲学”之一。

例如业务规定：

```text
必须：
1 Validate
2 Approve
3 Execute payment
```

错误做法：

```text
Prompt:
"Always validate before payment."
```

因为模型可能不遵守。

更可靠：

```text
Application code
   ↓
Validate
   ↓
Approval Check
   ↓
允许 payment tool
```

或者通过：

```text
Hooks
Tool permissions
Workflow enforcement
```

强制执行。

### 考试口诀

> **Suggestion → Prompt**
>
> **Guarantee → Code / Hook / Tool restriction**

看到：

```text
must
always
never
security
financial transaction
authorization
```

一般优先考虑：

**程序级 enforcement，而不是“再加强 Prompt”。**

---

# 六、Hooks ★★★★

Agent SDK Hooks 的思想：

```text
Claude
 ↓
Tool call
 ↓
HOOK
 ↓
Validation / Normalization / Security
 ↓
Actual Tool
```

适合处理：

```text
权限验证
输入标准化
安全检查
强制规则
Logging
```

如果题目问：

> 如何保证某个 tool 每次执行前都检查权限？

比：

```text
Prompt Claude to remember...
```

更好的答案通常是：

**Hook / programmatic enforcement**

---

# 七、Session Resume / Fork ★★★★

要区分：

### Resume

继续原 session：

```text
Session A
   ↓
Resume
   ↓
继续 A
```

保留原来的上下文。

Anthropic Claude Code CLI 中也有 session resume，例如：

```bash
claude -r "<session-id>"
```

([Claude Platform Docs][6])

### Fork

从某个状态产生：

```text
         Session
        /       \
     Fork A    Fork B
```

适合：

```text
尝试不同方案
并行分析
不污染原 session
```

---

# 八、Domain 2：Tool Design & MCP ★★★★★

18%。

虽然权重不最高，但**题目非常具体，非常容易拿分。**

共 5 个 Task。([CCAR-F Academy][7])

核心：

```text
Tool description
Structured error
tool_choice
MCP
Built-in Tools
```

---

## 1. Tool Description ★★★★★

这部分一定要记。

模型选择 Tool 时，非常依赖：

```text
name
description
input_schema
```

Anthropic 官方文档明确强调：**详细、清晰的 tool description 是影响工具选择效果的最重要因素之一**。描述应该说明它做什么、何时使用、何时不要使用、参数含义和限制。([Claude Platform Docs][5])

错误：

```json
{
  "name": "search",
  "description": "search data"
}
```

好：

```text
Search customer orders by customer ID.

Use when:
- customer asks about an existing order

Do not use when:
- searching product catalog

Returns:
- order_id
- status
- order_date
```

### 高频考试陷阱

Agent 调错 Tool：

第一反应通常不是：

```text
加更长 Prompt
```

而是检查：

```text
Tool description 是否重叠
Tool name 是否模糊
Tool boundary 是否明确
```

---

# 九、tool_choice ★★★★★

必背。

Anthropic API 当前工具机制中主要有：

```text
auto
any
tool
none
```

### auto

```text
Claude 自己决定是否调用工具
```

### any

```text
必须调用某个工具
Claude 自己选哪个
```

### tool

```text
强制调用指定 tool
```

### none

```text
禁止调用 tool
```

例如：

> 必须调用 create_report。

应该：

```json
{
  "type": "tool",
  "name": "create_report"
}
```

而不是：

```text
Prompt Claude:
"Please always use create_report."
```

官方 Tool Use 文档明确给出了这四种模式。([Claude Platform Docs][5])

---

# 十、MCP ★★★★★

MCP：

**Model Context Protocol**

考试不要把它理解成：

```text
database protocol
Claude-specific API
plugin
```

而应该理解成：

> **标准化 AI Application 与外部数据源 / Tools 连接的协议。**

Anthropic 官方把它比作 AI 应用的“USB-C”。([Claude Platform Docs][8])

基本：

```text
Claude / Agent
      ↓
   MCP Client
      ↓
   MCP Server
    /      \
 Tools   Resources
```

### Tool vs Resource

简单记：

**Resource**

偏：

```text
Read information
Content
Documents
Catalog
Reference data
```

**Tool**

偏：

```text
Action
Search
Create
Update
Execute
```

例如：

```text
company handbook
→ Resource
```

```text
create Jira ticket
→ Tool
```

---

# 十一、Structured Error ★★★★

Tool 不应该只返回：

```text
ERROR
```

最好返回结构化信息，例如：

```json
{
  "error_type": "permission_denied",
  "message": "User cannot refund this order",
  "retryable": false
}
```

让 Agent 可以判断：

```text
Retry？
换 Tool？
向 Coordinator 报告？
Escalate？
```

核心原则：

> **Recover locally when possible.**

不要一出现任何 Tool error：

```text
→ Human
```

---

# 十二、Domain 3：Claude Code ★★★★★

**20%，非常值得拿分。**

主要 6 项：CLAUDE.md、Skills / slash commands、path-specific rules、Plan Mode、iterative refinement、CI/CD。([CCAF Preparation][9])

---

## CLAUDE.md ★★★★★

非常可能直接考。

### Project

```text
./CLAUDE.md
```

适合：

```text
项目架构
编码规范
build/test command
团队共同规则
```

### User

```text
~/.claude/CLAUDE.md
```

适合：

```text
个人偏好
所有项目共同设置
```

Anthropic 官方文档也明确区分项目 memory 与用户 memory。([Claude Platform Docs][10])

### import

```markdown
@docs/git-instructions.md
```

可以拆分 CLAUDE.md。

例如：

```text
CLAUDE.md
 ├─ architecture.md
 ├─ coding.md
 └─ testing.md
```

官方文档当前支持 `@path/to/import`，且可递归导入。([Claude Platform Docs][10])

---

# 十三、CLAUDE.md vs Rules vs Skill

这组特别容易混。

### CLAUDE.md

```text
长期
项目级
默认需要
```

例如：

> 全项目 Java 使用 Java 21。

### `.claude/rules/`

路径条件规则：

```text
src/frontend/**
```

例如：

```text
frontend → React rules
backend → Spring rules
```

### Skill

更像：

```text
需要时调用的能力 / workflow
```

记：

> **Always relevant → CLAUDE.md**
>
> **Path dependent → Rules**
>
> **On demand → Skill**

---

# 十四、Plan Mode ★★★★

什么时候用 Plan Mode？

适合：

```text
Architecture change
Large refactor
Multiple files
Unknown codebase
有多个设计选择
```

例如：

> 将 monolith authentication 改为 OAuth architecture。

先：

**Plan**

再：

**Execute**

而：

> 修改一个变量名。

通常：

**Direct execution**

考试常用判断：

> **Complex / architectural / cross-file → Plan**

---

# 十五、Claude Code CI/CD ★★★★

要认识：

```bash
claude -p "..."
```

`-p` 适合：

```text
non-interactive
CI/CD
automation
```

例如：

```bash
claude -p "Review this code for security problems"
```

官方 CLI 文档也是把 `-p` 定义为执行 query 后退出，非常适合自动化。([Claude Platform Docs][6])

---

# 十六、Domain 4：Prompt & Structured Output ★★★★★

20%。

这部分非常容易出“两个答案看起来都对”的题。

六大内容：([CCAR-F Academy][11])

```text
Explicit criteria
Few-shot
JSON schema / tool use
Validation / Retry
Batch
Multi-pass Review
```

---

## Explicit Criteria

差：

```text
Only report serious bugs.
```

好：

```text
Report:
- security vulnerabilities
- runtime exceptions
- data corruption

Do not report:
- naming preferences
- formatting
- minor style issues
```

核心：

> **Explicit criteria > vague instruction**

---

# 十七、Few-shot ★★★★★

如果：

```text
Instructions 已经很多
```

但模型：

```text
格式还是不一致
边界案例判断不稳定
```

考试经常正确答案：

**Few-shot examples**

例如：

```text
Input A
→ Correct Output A

Input B
→ Correct Output B

Now process Input C
```

而不是：

```text
再写 3 页 instructions
```

---

# 十八、Structured Output ★★★★★

下游系统要求：

```json
{
  "name": "...",
  "amount": 100,
  "date": "..."
}
```

不要只：

```text
Please return JSON.
```

更可靠的设计：

```text
Tool
   ↓
input_schema
   ↓
JSON Schema
```

再根据需要：

```text
tool_choice
```

强制使用。

考试思想：

> **Prompt 可以要求格式。**
>
> **Schema 才负责约束格式。**

---

# 十九、Nullable ★★★★

这是一个很好的考试点。

原文件没有：

```text
invoice_number
```

如果 Schema 要求：

```text
invoice_number: string
```

模型可能：

**编一个值。**

更好：

```text
invoice_number:
    string | null
```

原则：

> **Unknown ≠ guess**

允许：

```text
null
```

---

# 二十、Validation → Retry → Feedback ★★★★★

推荐流程：

```text
Claude Extraction
      ↓
JSON Schema
      ↓
Semantic Validation
      ↓
Valid?
 /       \
Yes       No
 ↓         ↓
Save    Error feedback
            ↓
          Claude
```

例如：

```text
subtotal + tax != total
```

不能只是：

```text
Retry
```

而应该把错误告诉 Claude：

```text
Validation failed:
subtotal + tax does not equal total.
Please correct the extraction.
```

口诀：

> **Retry without feedback = 再赌一次**
>
> **Retry + specific error = Self-correction**

([CCAR-F Academy][11])

---

# 二十一、Batch API ★★★★

当前考试蓝图中的重要判断：

```text
大量任务
不要求即时完成
```

→ **Batch**

例如：

```text
夜间处理 100,000 documents
```

适合。

```text
用户在线等待结果
```

→ **Synchronous API**

当前 Anthropic 文档中 Batch processing 对 input/output token 都提供 **50% discount**。([Claude Platform Docs][12])

记：

> **Latency important → Sync**
>
> **Volume / cost important → Batch**

---

# 二十二、Multi-pass / Independent Review ★★★★

例如 Claude：

```text
Generate code
```

然后让**同一上下文里的 Claude**：

```text
Review your own code.
```

问题是：

它仍然带着自己刚刚做设计时的 reasoning/context，很容易延续原判断。

重要任务可以：

```text
Agent A
↓
Generate
↓
Agent B
↓
Independent Review
```

或者：

```text
Pass 1 → Generate
Pass 2 → Review
Pass 3 → Verify
```

---

# 二十三、Domain 5：Context Management & Reliability ★★★★

15%。

主要是“系统跑久了以后怎么办”。([CCAF Preparation][13])

---

## Context Degradation

不是只有：

```text
Context window 满了
```

才会出问题。

即使没满：

```text
大量 logs
大量 tool output
长对话
重复信息
```

都会造成：

```text
lost in the middle
```

典型症状：

```text
早期发现了 FooPaymentService
↓
几十轮以后
↓
Claude 开始说
"typically a payment service might..."
```

即：

**从具体事实退化成泛泛而谈。**

解决不是简单：

```text
扩大 Context Window
```

而是：

```text
Trim tool output
Structured facts
Summarization
重要信息单独保存
```

---

# 二十四、Case Facts ★★★★

例如客服 Agent。

这些数据：

```text
customer_id
order_id
amount
refund status
dates
```

不要只藏在越来越长的 conversation history 里。

保存：

```json
{
  "customer_id": "C001",
  "order_id": "O500",
  "amount": 120,
  "refund_status": "pending"
}
```

也就是：

**structured state / case facts**

然后继续带入后续任务。

---

# 二十五、Escalation ★★★★★

什么时候找 Human？

正确：

```text
用户明确要求人工
Policy gap
Policy ambiguity
无法继续取得必要信息
高风险/不可逆操作需要审批
```

不应该：

```text
Case 看起来复杂
Claude 自己感觉不确定
用户语气不好
```

就一律 escalate。

关键：

> **Escalation criteria 应提前定义。**

---

# 二十六、Error Propagation

Multi-Agent：

```text
Agent C fails
```

不要只返回：

```text
failed
```

给 Coordinator：

```text
error_type
attempted_action
partial_results
reason
possible_alternative
```

Coordinator 才能决定：

```text
retry
different agent
different tool
partial answer
human escalation
```

---

# 二十七、Confidence Calibration ★★★★

如果模型说：

```text
confidence = 0.95
```

不代表真的：

**95% 正确。**

需要：

```text
Labelled validation data
↓
Compare predicted confidence
↓
Actual accuracy
```

还要看：

```text
document type
field
category
```

例如：

```text
Overall = 97%
```

看起来非常好。

但：

```text
invoice_number = 99%
customer_address = 74%
```

那就不能说整个 extraction 已经达到 97% 可自动化。

---

# 二十八、Provenance ★★★★★

Research Agent 特别重要。

不要最后只得到：

```text
Revenue grew 20%
```

而应该保存：

```text
Claim
↓
Source
↓
Relevant evidence
↓
Date
```

例如：

```text
Claim:
Revenue grew 20%

Source:
2026 Annual Report

Date:
2026-03-10
```

如果 Source A 和 B 冲突：

错误：

```text
Claude 自己选一个看起来正确的
```

更好：

```text
保留两个 source
标记 conflict
说明不确定性
```

---

# 二十九、我建议你优先死记这 15 个判断

这是考前可以直接看的版本：

1. **`stop_reason=tool_use` → 执行工具继续 loop**
2. **`stop_reason=end_turn` → 正常结束**
3. **固定步骤 → Workflow；动态决定下一步 → Agent**
4. **独立任务 → Parallel subagents**
5. **Coordinator 管理 subagents**
6. **必须保证的规则 → Code / Hook，不靠 Prompt**
7. **Tool 调错 → 首先检查 tool description / boundary**
8. **`tool_choice:auto / any / tool / none` 要会区分**
9. **MCP Resource 偏信息；Tool 偏动作**
10. **全项目规则 → `CLAUDE.md`；路径规则 → `.claude/rules/`**
11. **复杂架构改动 → Plan Mode；小改动 → Direct**
12. **格式一致性差 → Few-shot**
13. **机器可解析结构 → JSON Schema / Tool Use**
14. **Validation 失败 → Retry + specific error feedback**
15. **长 Context → 保存 structured facts / provenance，不是无脑扩大 context**

如果你把这 15 条真正理解，而不是单纯背下来，已经能覆盖相当大一部分 CCAR-F 场景题的判断逻辑。相关机制与当前 Claude Tool Use、Claude Code memory/CLI 官方文档一致。([Claude Platform Docs][5])

### 备考策略

你有开发经验的话，我不建议从头把所有 Claude 文档通读一遍。更有效的是：

**第一阶段：D1 + D2，把 Agent / Tool / MCP 体系彻底搞懂。**

**第二阶段：D3，把 `CLAUDE.md`、rules、Skills、Plan Mode、CI 参数这种容易直接拿分的知识点背熟。**

**第三阶段：D4，重点刷“哪个方案更可靠”的场景题。**

**第四阶段：D5，专门记 context / escalation / provenance / confidence 的判断标准。**

最后一定要按 **60题 / 120分钟** 做整套模拟，因为平均只有 **2分钟/题**，而且一组题会共享同一个 Scenario。([CCAF Preparation][14])

下一步我建议直接给你做一份 **《CCAR-F 中文考试复习资料》**，按照 **D1～D5 共30个 Task Statement** 展开，每一个 Task 都包含 **“必背知识 → 容易出错点 → 考试陷阱 → 例题”**。这样比单纯看上面的概要更适合正式备考。

如果你要，我下一条就从 **Domain 1 的 7 个 Task 开始详细讲，并给你出 CCAR-F 风格模拟题。**

我也可以按固定节奏每天给你一组 CCAR-F 练习题。

[1]: https://www.anthropic.com/news/claude-partner-network?cmid=03693516-1d21-4130-ac10-3f9892c0929b&utm_source=chatgpt.com "Anthropic invests $100 million into the Claude Partner Network"
[2]: https://ccar-f.17labs.ai/exam/?utm_source=chatgpt.com "How the CCA-F exam works: format, scoring, and what is not tested · CCAR-F"
[3]: https://ccar-f.17labs.ai/study/?utm_source=chatgpt.com "CCA-F study guide — all 30 task statements · CCAR-F"
[4]: https://ccar-f.17labs.ai/domains/1/?utm_source=chatgpt.com "Domain 1: Agentic Architecture & Orchestration · CCAR-F"
[5]: https://docs.anthropic.com/ko/docs/agents-and-tools/tool-use/implement-tool-use?utm_source=chatgpt.com "도구 사용 구현 방법 - Anthropic"
[6]: https://docs.anthropic.com/en/docs/claude-code/cli-usage?utm_source=chatgpt.com "CLI reference - Anthropic"
[7]: https://ccar-f.17labs.ai/domains/2/?utm_source=chatgpt.com "Domain 2: Tool Design & MCP Integration · CCAR-F"
[8]: https://docs.anthropic.com/en/docs/mcp?utm_source=chatgpt.com "Model Context Protocol (MCP) - Anthropic"
[9]: https://www.ccafpreparation.com/learn/claude-code?utm_source=chatgpt.com "Claude Code Configuration & Workflows — CCAR-F Domain 3 (20% of the Exam) — CCAF Preparation"
[10]: https://docs.anthropic.com/zh-CN/docs/claude-code/memory?utm_source=chatgpt.com "管理 Claude 的内存 - Anthropic"
[11]: https://ccar-f.17labs.ai/domains/4/?utm_source=chatgpt.com "Domain 4: Prompt Engineering & Structured Output · CCAR-F"
[12]: https://docs.anthropic.com/en/docs/about-claude/pricing?4810b549_page=3&73cdfb14_page=2&939688b5_page=1&e768fcd2_page=2&utm_source=chatgpt.com "Pricing - Anthropic"
[13]: https://www.ccafpreparation.com/learn/context-reliability?utm_source=chatgpt.com "Context Management & Reliability — CCAR-F Domain 5 (15% of the Exam) — CCAF Preparation"
[14]: https://www.ccafpreparation.com/mock-exam?utm_source=chatgpt.com "Free CCAR-F Mock Exam — 60 Questions, 120 Minutes, Timed — CCAF Preparation"
