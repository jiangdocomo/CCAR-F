# CCAR-F Daily Practice — Day 18

Today’s set is a **hard architecture set** emphasizing newer production-agent patterns: **tool discovery, progressive disclosure, programmatic tool calling, context isolation, harness design, blast radius, outcome-based evaluation, and workflow-vs-agent selection**. These topics align with Anthropic’s current guidance on tool use, context engineering, managed agents, and agent evaluation. ([Anthropic][1])

**Exam mode:** 10 questions. **Target time: 20 minutes.** Questions 5 and 9 are **Choose TWO**.

---

## Scenario: Aegis Enterprise Operations Platform

Aegis Corp. operates a Claude-based enterprise operations platform.

The platform connects to more than 40 internal services and exposes approximately **1,200 tools** through several MCP servers.

The system includes:

* `CoordinatorAgent` — decomposes operational requests.
* `FinanceAgent` — investigates financial records.
* `InfrastructureAgent` — diagnoses infrastructure incidents.
* `SecurityAgent` — investigates security events.
* `ReviewAgent` — independently reviews high-impact actions.

Available capabilities include:

```text
search_invoices
get_customer
search_logs
get_service_health
retrieve_runbook
restart_service
update_firewall_rule
create_incident
issue_refund
send_notification
```

Some requests require only a few tools, while complex investigations can run for several hours.

---

## Question 1 — Large Tool Catalog

Aegis initially loads the definitions of all **1,200 tools** into Claude’s context before every request.

Most requests use fewer than five tools.

Engineers observe increased token usage and slower processing.

What is the BEST architectural improvement?

**A.** Remove tool descriptions so all 1,200 definitions become shorter.  
**B.** Use tool discovery/search so Claude loads detailed definitions only for tools relevant to the current task.  
**C.** Increase the context window and continue loading everything.  
**D.** Randomly expose 100 tools on each request.  

### Correct Answer: **B**

**English Explanation:**
Large tool catalogs benefit from **progressive disclosure**. Claude can discover relevant tools first and load their detailed schemas only when needed, reducing unnecessary context consumption.

### 中文解说

这是今天第一个重点：

**Progressive Disclosure**

问题不是：

> 1,200 个 Tool 太多，所以 Claude 不能使用。

而是：

> **没有必要把 1,200 个 Tool 的完整定义全部提前塞进 Context。**

更合理：

```text
User task
   ↓
Tool discovery
   ↓
Find relevant tools
   ↓
Load detailed schemas
   ↓
Execute
```

例如用户问：

> Investigate server CPU usage.

可能只需要：

```text
search_logs
get_service_health
retrieve_runbook
```

完全没必要让 Context 同时包含：

```text
issue_refund
search_invoices
customer CRM tools
...
```

Anthropic 当前 advanced tool-use guidance specifically describes on-demand tool discovery as a way to avoid loading huge tool libraries into context upfront. ([Anthropic][2])

A 会降低 Claude 正确使用工具的能力。

C 只是扩大容器，没有消除浪费。

D 会导致需要的 Tool 可能根本没有被暴露。

**Key exam takeaway:**

> **Large tool library + small relevant subset → progressive tool discovery.**

---

## Question 2 — Programmatic Tool Calling

`FinanceAgent` must examine 500 invoices.

For each invoice it must:

```text
retrieve invoice
→ extract amount
→ compare amount to threshold
→ keep only exceptions
```

The current architecture performs a separate natural-language model turn for every invoice, placing every intermediate result into Claude’s context.

What is the BEST optimization?

**A.** Ask Claude to produce longer explanations for each invoice.  
**B.** Use programmatic tool calling/code execution to perform the repetitive loop and filtering, returning only relevant exceptions to the model.  
**C.** Create 500 permanent subagents.  
**D.** Put all 500 full invoice results into the system prompt.  

### Correct Answer: **B**

**English Explanation:**
Repetitive deterministic orchestration, filtering, and data transformation are strong candidates for programmatic execution rather than requiring a full inference step for every operation.

### 中文解说

这是今天第二个重要概念：

**Programmatic Tool Calling**

假设有 500 个 Invoice。

错误架构：

```text
Claude
 ↓
Tool #1
 ↓
Claude
 ↓
Tool #2
 ↓
Claude
 ↓
Tool #3
 ...
```

每次都发生：

**Inference + Context accumulation**

但实际逻辑只是：

```text
for invoice in invoices:
    data = get_invoice(invoice)
    if data.amount > threshold:
        exceptions.append(data)
```

这种：

* loop；
* filter；
* condition；
* data transformation；

非常适合程序执行。

最后只给 Claude：

```text
17 exceptional invoices
```

而不是 500 个完整 Tool Result。

Anthropic’s current guidance specifically highlights code/programmatic tool execution for loops, conditionals, data transformations, and reducing intermediate context consumption. ([Anthropic][2])

**Key exam takeaway:**

> **Use inference for judgment; use code for repetitive deterministic orchestration.**

---

## Question 3 — Workflow or Agent?

Every employee password-reset request follows exactly:

```text
1. Verify employee identity
2. Verify account status
3. Generate reset token
4. Send reset notification
```

The sequence and conditions are completely known in advance.

Which architecture is BEST?

**A.** A fully autonomous agent that decides what step to perform next.  
**B.** A deterministic workflow, using Claude only where interpretation is actually needed.  
**C.** A team of four agents voting on every step.  
**D.** An open-ended research agent.  

### Correct Answer: **B**

**English Explanation:**
When the required path is predictable and deterministic, a workflow is simpler and more reliable. Agent autonomy is more valuable when the necessary sequence cannot be known in advance.

### 中文解说

这是 CCAR-F 必须非常熟练的判断：

### Workflow

```text
Path known beforehand
A → B → C → D
```

### Agent

```text
Goal
 ↓
Claude investigates
 ↓
Chooses next action
 ↓
Observes result
 ↓
Chooses again
```

题目明确：

> sequence and conditions are completely known

所以没有必要让 Claude 自己决定流程。

Anthropic distinguishes workflows as predefined code paths from agents that dynamically direct their own process, and recommends starting with the simplest architecture that meets the requirement. ([Anthropic][3])

**Key exam takeaway:**

> **Known path → workflow. Unknown path → consider agent.**

---

## Question 4 — Ground Truth from Environment

`InfrastructureAgent` executes:

```text
restart_service("payments-api")
```

Claude then says:

> “The service has recovered successfully.”

What should determine whether the incident is actually resolved?

**A.** Claude’s confidence score.  
**B.** Whether Claude used the phrase “recovered successfully.”  
**C.** Fresh environment evidence such as service health, logs, or monitoring results.  
**D.** Whether the restart tool was called.  

### Correct Answer: **C**

**English Explanation:**
Agents should use environmental feedback as ground truth. Calling a remediation tool does not establish that the remediation achieved its intended outcome.

### 中文解说

注意三个不同状态：

```text
Tool requested
      ↓
Tool executed
      ↓
Desired outcome achieved
```

三者不能画等号。

例如：

```text
restart_service
→ SUCCESS
```

只代表 restart 操作执行了。

但 Service 可能：

```text
restart
→ crash again
```

所以必须重新检查：

```text
get_service_health
search_logs
monitoring
```

Anthropic’s agent guidance emphasizes obtaining ground truth from the environment during execution so agents can assess progress. ([Anthropic][3])

**Key exam takeaway:**

> **Tool success ≠ task success. Verify the intended outcome.**

---

## Question 5 — Blast Radius

### Choose TWO.

`InfrastructureAgent` investigates production incidents.

It currently has unrestricted credentials that can:

* restart every service;
* modify all firewall rules;
* delete production databases;
* administer unrelated HR systems.

Which TWO changes MOST directly reduce potential **blast radius**?

**A.** Scope credentials to the systems and actions required for the agent’s role.  
**B.** Isolate dangerous execution capabilities behind stronger permission boundaries.  
**C.** Give Claude unrestricted access but add “BE CAREFUL” to the system prompt.  
**D.** Increase the context window so Claude better understands the consequences.  

### Correct Answers: **A and B**

**English Explanation:**
Blast radius is reduced by limiting what a failure can affect. Capability scoping and containment provide stronger protection than relying solely on behavioral guidance.

### 中文解说

今天必须记住这个词：

**Blast Radius（故障/错误影响范围）**

安全设计不只是问：

> Claude 犯错概率多高？

还要问：

> **如果 Claude 犯一次错，最多能造成多大损害？**

例如：

```text
Agent error
   ↓
Can restart ONE service?
```

和：

```text
Agent error
   ↓
Can delete ALL production databases?
```

完全不同。

所以：

**A — Least Privilege**

限制能力。

**B — Containment**

隔离危险 capability。

Anthropic’s 2026 containment guidance frames agent risk partly in terms of limiting how much damage a failure can cause as agents gain broader capabilities. ([Anthropic][4])

C 只有 behavioral guidance。

D 并没有限制实际 capability。

**Key exam takeaway:**

> **Safety = reduce failure probability AND limit failure impact.**

---

## Question 6 — Context Storage vs Active Context

A long-running investigation produces **300 MB of logs**.

Claude may need to revisit small portions later.

What is the BEST design?

**A.** Keep all 300 MB permanently inside Claude’s active context.  
**B.** Store the logs externally and let Claude selectively retrieve relevant slices when needed.  
**C.** Delete the logs after the first summary.  
**D.** Repeat the complete logs after every agent handoff.  

### Correct Answer: **B**

**English Explanation:**
Durable information storage and active model context are separate concerns. Large evidence can remain externally available while only relevant portions are brought into the context window.

### 中文解说

这是一个很重要的区别：

> **Stored information ≠ Active context**

可以设计：

```text
SESSION / STORAGE
─────────────────
300 MB logs
full evidence
history

        ↓ selective retrieval

ACTIVE CONTEXT
─────────────────
Relevant log slice
Current findings
Current objective
```

Anthropic’s context-engineering guidance emphasizes external organization and just-in-time retrieval rather than loading entire data objects into context, while its newer Managed Agents architecture explicitly separates durable session state from the context presented to the model. ([Anthropic][5])

A 会造成严重 context pressure。

C 损失 evidence。

D 更糟糕。

**Key exam takeaway:**

> **Durable memory can be large; active context should remain selective.**

---

## Question 7 — Harness Assumption

Aegis’s harness automatically resets Claude’s context every 40 turns because an older model tended to terminate tasks prematurely near its context limit.

A newer model no longer shows this behavior, but the resets now cause useful working state to be lost.

What should engineers do?

**A.** Keep the reset forever because harness rules should never change.  
**B.** Re-evaluate the harness assumption and remove or modify the workaround if current evaluation shows it is no longer beneficial.  
**C.** Add resets every 20 turns instead.  
**D.** Solve the issue by increasing temperature.  

### Correct Answer: **B**

**English Explanation:**
Harnesses encode assumptions about model behavior, and those assumptions can become stale as models improve. Harness behavior should therefore be validated against current models and workloads.

### 中文解说

这是一个非常新的架构思维：

> **Harness itself can become stale.**

以前：

```text
Model behavior X
   ↓
Harness workaround Y
```

后来 Model 改进：

```text
Model no longer has X
```

但 Y 还在。

结果：

> Workaround 本身变成问题。

Anthropic’s April 2026 Managed Agents engineering discussion explicitly notes that harnesses encode assumptions that can become stale as model capabilities change, using context-reset behavior as an example. ([Anthropic][1])

所以 Harness 也要进入：

```text
Evaluation
→ Regression testing
→ Optimization
```

**Key exam takeaway:**

> **Do not preserve a workaround after the behavior it addressed has changed.**

---

## Question 8 — Multi-Agent Parallelism

A complex security investigation requires:

```text
A. Analyze authentication logs  
B. Analyze network logs  
C. Analyze endpoint telemetry  
D. Synthesize the three analyses  
```

A, B, and C are independent.

Which design BEST reduces latency while preserving correctness?

**A.** Run A → B → C → D sequentially.  
**B.** Run A, B, and C in parallel, then run D after their required results are available.  
**C.** Run all four simultaneously.  
**D.** Run D first and let it predict the missing evidence.  

### Correct Answer: **B**

**English Explanation:**
Independent investigation branches can execute in parallel, while synthesis is a dependent fan-in step.

### 中文解说

这是熟悉的：

**Fan-out / Fan-in**

```text
          ┌→ A ─┐
Start ────┼→ B ─┼→ D
          └→ C ─┘
```

注意考试陷阱：

> “Parallel 更快”

并不等于：

```text
A + B + C + D
all parallel
```

D 需要 A/B/C 的 evidence。

Anthropic’s work on parallel agent teams similarly emphasizes structuring work so multiple agents can make progress concurrently where tasks permit it. ([Anthropic][6])

**Key exam takeaway:**

> **Parallelize independent work, not dependent reasoning.**

---

## Question 9 — Agent Evaluation

### Choose TWO.

Aegis tests a new incident-remediation agent.

For each trial, engineers currently evaluate only the final natural-language response.

Which TWO additions would MOST improve the evaluation?

**A.** Check the resulting environment state to determine whether remediation actually succeeded.  
**B.** Run representative trials multiple times to measure reliability under agent nondeterminism.  
**C.** Score responses only by length.  
**D.** Treat any tool invocation as proof of successful remediation.  

### Correct Answers: **A and B**

**English Explanation:**
Agent evaluation should inspect actual outcomes and account for nondeterministic behavior across repeated trials rather than relying solely on the final transcript.

### 中文解说

Agent Eval 和普通问答 Eval 不一样。

必须考虑：

### Outcome

```text
Claude says:
"Fixed."
```

不够。

要检查：

```text
System actually fixed?
```

### Nondeterminism

同一个任务：

```text
Trial 1 → PASS
Trial 2 → PASS
Trial 3 → FAIL
Trial 4 → PASS
```

不能只跑一次就说：

> Reliability = 100%

Anthropic’s January 2026 eval guidance emphasizes evaluating environment outcomes and repeated trials because agent behavior is nondeterministic. ([Anthropic][7])

**Key exam takeaway:**

> **Evaluate real outcomes across repeated trials.**

---

## Question 10 — Failure-Layer Diagnosis

Aegis observes:

> Claude identifies the correct firewall rule to change.
> The requested change is valid.
> The change requires security-team approval.
> Claude correctly states that approval is required.
> Nevertheless, `update_firewall_rule` executes without checking whether approval exists.

Which layer MOST directly failed?

**A.** Model reasoning  
**B.** Context engineering  
**C.** Execution-time authorization enforcement  
**D.** Tool discovery  

### Correct Answer: **C**

**English Explanation:**
Claude understood the policy correctly. The failure occurred because the state-changing execution boundary did not enforce the required authorization.

### 中文解说

逐层诊断：

```text
Claude reasoning       ✓
Action identification  ✓
Validation             ✓
Policy understanding   ✓

Execution enforcement  ✗
```

这里再强调一个考试核心：

> **Claude knows a rule**

不等于：

> **System enforces the rule**

错误架构：

```text
Prompt:
"Only change firewall after approval."

Claude
   ↓
update_firewall_rule
   ↓
EXECUTE
```

更强：

```text
Claude
   ↓
update_firewall_rule
   ↓
Check approval
   ↓
Approved?
├─ Yes → execute
└─ No  → reject
```

**Key exam takeaway:**

> **High-impact constraints belong at the execution boundary, not only in model instructions.**

---

# Day 18 — New High-Value Terms

| Term                          | Exam meaning                                                                 |
| ----------------------------- | ---------------------------------------------------------------------------- |
| **Progressive disclosure**    | Load detailed tools/context only when relevant                               |
| **Tool discovery**            | Find relevant capabilities without loading the entire catalog                |
| **Programmatic tool calling** | Use code for loops, conditions, filtering, and orchestration                 |
| **Blast radius**              | Maximum damage a failure or compromised agent can cause                      |
| **Harness**                   | System surrounding the model: tools, loop, context, orchestration, execution |
| **Ground truth**              | Actual environment state used to verify progress                             |
| **Context storage**           | Durable information available for later retrieval                            |
| **Active context**            | Information currently presented to the model                                 |

## The Three-Layer Rule

For difficult CCAR-F questions, mentally separate:

```text
MODEL
What does Claude believe/decide?
        ↓
HARNESS
How are context, tools, retries,
and orchestration managed?
        ↓
ENVIRONMENT
What actually happened?
```

Then ask:

> **Which layer first became incorrect?**

That often eliminates two or three plausible distractors immediately.

## Day 18 Common Trap

If Claude says:

> “I restarted the service and the incident is resolved.”

There are actually **two claims**:

**Claim 1:** The restart occurred.

**Claim 2:** The restart solved the incident.

A successful `restart_service` result may establish Claim 1.

It does **not** automatically establish Claim 2.

You still need environmental verification.

### Eight sentences to memorize

> **Discover large tool catalogs progressively.**

> **Use code for repetitive deterministic tool orchestration.**

> **Known paths favor workflows; unknown paths may justify agents.**

> **Tool success does not prove task success.**

> **Limit blast radius with capability boundaries.**

> **Durable storage and active context are different concerns.**

> **Harness assumptions must evolve as models evolve.**

> **High-impact authorization must be enforced where the action executes.**

[1]: https://www.anthropic.com/engineering/managed-agents?utm_source=chatgpt.com "Scaling Managed Agents: Decoupling the brain from the hands \ Anthropic"
[2]: https://www.anthropic.com/engineering/advanced-tool-use?utm_source=chatgpt.com "Introducing advanced tool use on the Claude Developer Platform \ Anthropic"
[3]: https://www.anthropic.com/engineering/building-effective-agents?utm_source=chatgpt.com "Building Effective AI Agents \ Anthropic"
[4]: https://www.anthropic.com/engineering/how-we-contain-claude?utm_source=chatgpt.com "How we contain Claude across products \ Anthropic"
[5]: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents?utm_source=chatgpt.com "Effective context engineering for AI agents \ Anthropic"
[6]: https://www.anthropic.com/engineering/building-c-compiler?utm_source=chatgpt.com "Building a C compiler with a team of parallel Claudes \ Anthropic"
[7]: https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents?utm_source=chatgpt.com "Demystifying evals for AI agents \ Anthropic"
