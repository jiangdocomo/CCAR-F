# CCAR-F Daily Practice — Day 2

Today’s set focuses on **Domain 2: Tool Design & MCP Integration**, with some cross-domain questions involving **structured output, reliability, and programmatic enforcement**. These are deliberately scenario-based and include distractors that are plausible but architecturally weaker.

**Suggested exam mode:** answer all 8 questions first without looking at the explanations. Target time: **16 minutes**.

---

## Scenario: Enterprise Support Agent

**Northstar Systems is building a Claude-based enterprise support agent.**

The agent can:

* search internal knowledge articles;
* retrieve customer account information;
* query order status;
* issue eligible refunds;
* create support tickets;
* retrieve company policies through an MCP server.

The engineering team has observed several production issues:

* Claude sometimes selects the wrong search tool.
* Tool failures are difficult for Claude to recover from.
* Refund operations must follow strict authorization rules.
* Some downstream applications require machine-readable output.
* The company wants to expose additional internal systems through MCP.

---

## Question 1 — Tool Description

Northstar provides two tools:

```text
search_orders
Description: Search orders.

search_products
Description: Search products.
```

Claude frequently calls `search_products` when users ask about the status of an existing order.

What is the **BEST first improvement**?

**A.** Increase the model temperature so Claude explores more tool choices.
**B.** Add detailed descriptions explaining when each tool should and should not be used.
**C.** Add a system prompt saying, “Always select the correct tool.”
**D.** Merge both tools into one generic `search` tool.

### Correct Answer: **B**

**English Explanation:**
Tool descriptions are a primary mechanism Claude uses to determine which tool is appropriate. Clearly specifying intended use, boundaries, parameters, and situations where the tool should not be used improves tool selection.

For example:

```text
search_orders

Search existing customer orders by order ID or customer ID.

Use when:
- checking shipping status
- checking an existing order
- investigating an order problem

Do not use when:
- searching the product catalog
- recommending products
```

### 中文解说

这题非常值得记。

题目说的是：

> Claude **选择错了工具**

首先应该检查的是：

**Tool description / tool boundary**

而不是马上修改整个 system prompt。

A 的 temperature 与工具语义不清没有直接关系。

C 虽然听起来合理，但：

> “Always select the correct tool.”

实际上没有告诉 Claude **什么叫正确**。

D 更糟糕。两个语义本来就不同的工具合并为一个 generic tool，通常会让边界更加模糊。

**考试速记：**

> **Wrong tool selection → inspect tool descriptions first.**

---

## Question 2 — `tool_choice`

Northstar has a workflow in which Claude **must call `create_support_ticket`** before the process can continue.

The application should not allow Claude to choose another tool instead.

Which configuration is MOST appropriate?

**A.** `tool_choice = auto`
**B.** `tool_choice = any`
**C.** Force the specific `create_support_ticket` tool.
**D.** Keep `tool_choice = auto` and write “You must create a ticket” in the prompt.

### Correct Answer: **C**

**English Explanation:**
When a specific tool must be invoked, the application should explicitly force that tool rather than relying on model discretion.

`auto` allows Claude to decide whether and which tool to call. A mode that requires some tool still does not necessarily guarantee the specific required tool.

### 中文解说

这里关键词是：

> **must call `create_support_ticket`**

不是：

> must call **a tool**

而是：

> must call **this specific tool**

因此应该强制指定 `create_support_ticket`。

区分这几个概念：

```text
auto
→ Claude decides whether to use a tool.

any
→ Claude must use a tool,
  but chooses which one.

specific tool
→ Claude must use the named tool.
```

D 又是经典陷阱：

**Prompt instruction ≠ deterministic enforcement**

如果业务流程要求 **必须**执行某个结构化动作，就不要只靠自然语言。

**考试速记：**

> **Must use some tool → `any`**
> **Must use this tool → force the specific tool**

---

## Question 3 — Structured Tool Errors

The `lookup_order` tool currently returns this when the order service is unavailable:

```text
ERROR
```

Claude often responds poorly because it cannot determine whether it should retry or choose another action.

Which replacement is BEST?

**A.**

```json
{
  "error": true
}
```

**B.**

```json
{
  "message": "Something went wrong."
}
```

**C.**

```json
{
  "error_type": "service_unavailable",
  "message": "Order service is temporarily unavailable.",
  "retryable": true,
  "retry_after_seconds": 10
}
```

**D.** Return an empty successful result.

### Correct Answer: **C**

**English Explanation:**
Structured errors give the agent actionable information. The error type, retryability, and retry guidance allow Claude or the orchestration layer to choose an appropriate recovery strategy.

### 中文解说

工具错误不能只考虑：

> “人能不能看懂？”

还要考虑：

> **Agent 能不能根据错误继续做决定？**

C 给出了：

```text
error_type
retryable
retry_after
message
```

于是 Agent 可以判断：

```text
Temporary?
    ↓
Retry

Permanent?
    ↓
Alternative / escalate
```

A、B 虽然也是错误信息，但缺少 recovery information。

D 是危险做法，因为：

> failure 被伪装成 success。

这可能让后面的 Agent 基于错误前提继续推理。

**考试速记：**

> **Errors should be actionable, not merely descriptive.**

---

## Question 4 — MCP

Northstar has several internal systems:

```text
CRM
Order database
Policy repository
Ticketing system
```

Different AI applications need standardized access to these systems.

What is the primary architectural value of **MCP**?

**A.** MCP increases Claude's context window.
**B.** MCP provides a standardized protocol for connecting AI applications with external tools and data sources.
**C.** MCP automatically converts every deterministic workflow into an autonomous agent.
**D.** MCP replaces authentication and authorization for enterprise systems.

### Correct Answer: **B**

**English Explanation:**
The Model Context Protocol provides a standardized way for AI applications to connect to external systems that expose tools, resources, and related capabilities. It addresses integration interoperability rather than context-window size or authorization policy.

### 中文解说

MCP 是考试中的基础定义题，但经常包装成 Scenario。

可以把它记成：

```text
AI Application
      │
      │ standardized interface
      ↓
     MCP
      ↓
 ┌────┼─────┐
 CRM  DB   Files
```

核心是：

> **Standardized connectivity**

A 错：MCP 不负责扩大 context window。

C 错：MCP 是 integration protocol，不是 agent orchestration framework。

D 特别容易误选。

MCP 可以连接企业系统，但：

**连接协议 ≠ 自动解决安全权限**

Authentication、authorization 仍然需要设计。

**考试速记：**

> **MCP solves integration standardization, not every application concern.**

---

## Question 5 — MCP Resource vs Tool

Northstar wants to expose two capabilities through MCP:

1. The employee handbook, which Claude should read for reference.
2. `create_support_ticket`, which creates a new record in the ticketing system.

How should these capabilities MOST naturally be modeled?

**A.** Both should be Resources.
**B.** Both should be Tools.
**C.** Handbook → Resource; `create_support_ticket` → Tool.
**D.** Handbook → Tool; `create_support_ticket` → Resource.

### Correct Answer: **C**

**English Explanation:**
Resources naturally expose information or content for consumption, while tools represent callable operations or actions.

The handbook is reference information. Creating a ticket changes external state and is naturally represented as a tool.

### 中文解说

这是非常适合直接背的判断。

### Resource

偏向：

```text
Read
Reference
Content
Information
```

例如：

```text
Employee handbook
Documentation
Policy documents
Catalog
```

### Tool

偏向：

```text
Action
Execute
Create
Update
Search operation
```

例如：

```text
create_ticket
send_email
issue_refund
query_database
```

所以：

```text
Handbook
→ Resource

Create Ticket
→ Tool
```

**考试速记：**

> **Resource = information**
> **Tool = action**

实际场景会比这更复杂，但作为 CCAR-F 判断原则非常有用。

---

## Question 6 — Security Boundary

Northstar exposes this tool:

```text
issue_refund(order_id, amount)
```

Only refunds up to the customer's authorized refund limit may be executed.

Which architecture provides the **STRONGEST protection**?

**A.** Describe the refund limit clearly in the tool description.
**B.** Add several few-shot examples of unauthorized refunds.
**C.** Validate authorization programmatically before the refund operation executes.
**D.** Ask Claude to output its confidence score before issuing the refund.

### Correct Answer: **C**

**English Explanation:**
Authorization is a hard security boundary and should be enforced outside model discretion. Tool descriptions and examples help guide behavior but should not be the sole mechanism preventing unauthorized state-changing operations.

### 中文解说

这道题故意把 Domain 1 和 Domain 2 混在一起。

Tool description 很重要，但它主要帮助：

> Claude **正确选择和使用 Tool**

它不能替代：

> **security enforcement**

Refund 是：

```text
state-changing
financial
potentially irreversible
```

所以权限检查必须放在：

```text
Claude
   ↓
refund tool request
   ↓
Authorization Check
   ↓
Allowed?
 /      \
Yes      No
 ↓        ↓
Execute  Reject
```

A 是一个很有迷惑性的选项。

上一题说：

> Tool description 很重要。

但不能把它推导成：

> Tool description 可以保证安全。

**考试特别喜欢这种“正确概念用错地方”的 distractor。**

**考试速记：**

> **Descriptions guide. Code guarantees.**

---

## Question 7 — Structured Output

After resolving a support case, a downstream system requires exactly these fields:

```text
case_id
resolution
refund_amount
escalated
```

The downstream parser frequently fails because Claude sometimes changes field names or adds explanatory prose.

What is the BEST design?

**A.** Add “Return valid JSON only” to the end of the prompt.
**B.** Increase temperature to improve formatting diversity.
**C.** Define the required structure using a schema/tool-based structured output mechanism and validate the result.
**D.** Ask Claude to generate the response twice and select the longer one.

### Correct Answer: **C**

**English Explanation:**
Machine-consumed output should use explicit structural constraints and validation. Natural-language instructions can help, but schema-based output is more reliable when downstream software requires predictable fields and types.

### 中文解说

题目关键词：

> **downstream parser**

说明结果不是主要给人看的，而是：

> **给程序消费**

这种情况下：

```text
"Please return JSON."
```

不够可靠。

更好的架构：

```text
Claude
  ↓
Schema-constrained output
  ↓
Validation
  ↓
Downstream System
```

如果 validation 失败：

```text
Specific validation error
       ↓
      Claude
       ↓
Correction
```

A 是非常经典的 distractor。

Prompt 可以提高遵守率，但题目问：

> **BEST design**

应该选择 schema + validation。

**考试速记：**

> **Human-readable preference → Prompt may be enough.**
> **Machine-readable contract → Schema + validation.**

---

## Question 8 — Retry Strategy

The ticketing tool returns:

```json
{
  "error_type": "timeout",
  "retryable": true
}
```

The agent immediately retries indefinitely until the request succeeds.

What is the MOST important architectural improvement?

**A.** Replace structured errors with plain-English messages.
**B.** Use a bounded retry policy with limits/backoff and propagate failure context if retries are exhausted.
**C.** Escalate every timeout immediately to a human operator.
**D.** Tell Claude in the system prompt not to retry too many times.

### Correct Answer: **B**

**English Explanation:**
Transient errors can often be recovered locally, but retries must be bounded. A robust system applies a retry policy—often including backoff—and returns structured failure information when recovery is unsuccessful.

### 中文解说

这里有两个极端：

```text
Never retry
```

和：

```text
Retry forever
```

都不对。

正确思路：

```text
Transient error
     ↓
Retry
     ↓
bounded attempts
     ↓
Backoff
     ↓
Still failing?
     ↓
Return structured failure
     ↓
Coordinator decides
```

C 属于 **over-escalation**。

一次 timeout 很可能只是暂时故障，不应该马上人工介入。

D 又是在用 Prompt 解决 application control 问题。

**考试速记：**

> **Retryable does NOT mean retry forever.**

---

# Day 2 — Quick Review

今天建议把下面 **8 个英文判断句**直接背下来：

| Exam signal                            | Preferred answer                                |
| -------------------------------------- | ----------------------------------------------- |
| **Claude chooses the wrong tool**      | Improve tool descriptions and boundaries        |
| **Claude must use a specific tool**    | Force that specific tool                        |
| **Tool operation fails**               | Return structured, actionable error information |
| **Standardized external integration**  | MCP                                             |
| **Reference information**              | Resource                                        |
| **External action / state change**     | Tool                                            |
| **Authorization / security guarantee** | Programmatic enforcement                        |
| **Machine-readable output**            | Schema + validation                             |

再记住今天最重要的四组区别：

```text
Tool description ≠ Security enforcement

Prompt instruction ≠ Hard guarantee

Resource ≠ Tool

Retryable ≠ Retry forever
```

### Common CCAR-F Trap

今天最值得注意的出题方式是：

> **An option can describe a valid technique but still be the wrong architectural layer.**

例如：

**“Improve the tool description.”**

本身是正确技术，但如果问题是：

> How do you **guarantee authorization**?

它就不是最佳答案。

反过来，如果问题是：

> Claude keeps selecting the wrong tool.

此时直接增加 application authorization code，也没有解决真正的问题。

所以做 CCAR-F 场景题时，可以先问自己一句：

> **What layer is actually failing — model guidance, tool interface, orchestration, security enforcement, output contract, or reliability?**

先判断“哪一层出了问题”，再选答案，通常比逐个分析四个选项更快。
