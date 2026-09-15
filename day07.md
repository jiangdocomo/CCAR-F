# CCAR-F Daily Practice — Day 7

Today is a **mixed-domain scenario set**. The questions deliberately avoid telling you which domain is being tested. Your first task is to identify whether the real problem is **orchestration, tool design, Claude Code configuration, prompting/structured output, or reliability**.

**Exam mode:** 10 questions. Target time: **20 minutes**. Questions 4 and 9 are **Choose TWO**.

---

## Scenario: Apex Enterprise Incident Agent

**Apex Cloud operates a Claude-based incident-response system for enterprise customers.**

When a production incident occurs, a coordinator can delegate work to:

* `LogAgent` — analyzes application and infrastructure logs.
* `CodeAgent` — investigates the relevant source code.
* `DatabaseAgent` — analyzes database behavior.
* `SecurityAgent` — investigates possible security impact.

Available tools include:

```text
search_logs
search_source_code
query_database
retrieve_runbook
restart_service
create_incident_ticket
```

The system must diagnose incidents quickly while preventing unsafe production actions.

---

## Question 1 — Parallel vs Sequential

After an incident begins, the coordinator already knows the affected service and time range.

It needs to:

1. Analyze application logs.
2. Inspect the relevant source code.
3. Compare the findings from steps 1 and 2 to identify the likely root cause.

What is the BEST execution strategy?

**A.** Execute all three tasks simultaneously.
**B.** Analyze logs first, then inspect source code, then compare them.
**C.** Run log analysis and source-code inspection in parallel, then perform the comparison after both complete.
**D.** Let the comparison agent begin first and request the other results later.

### Correct Answer: **C**

**English Explanation:**
Log analysis and source-code inspection are independent once the affected service and time range are known, so they can run concurrently. Root-cause comparison depends on both outputs and must occur afterward.

### 中文解说

先画 dependency：

```text
Log Analysis ─────┐
                  ├── Root Cause Analysis
Code Analysis ────┘
```

前两个没有相互依赖，所以：

**Parallel**

最后一个依赖前两个结果，所以：

**Sequential after both complete**

A 是常见陷阱：

> “既然 parallel 更快，那全部 parallel。”

但第三个任务没有输入就不能真正完成。

B 虽然逻辑上可以运行，但浪费 latency。

**Exam takeaway:**

> **Parallelize independent work, not dependent work.**

---

## Question 2 — Tool Description vs Prompt

Claude frequently calls:

`search_source_code`

when it should call:

`search_logs`

Both tools currently have this description:

> Search system information.

What should the engineering team do FIRST?

**A.** Add “Think carefully before selecting tools” to the system prompt.
**B.** Rewrite the tool descriptions to clearly define purpose, inputs, use cases, and non-use cases.
**C.** Add a human approval step before every search operation.
**D.** Force `search_logs` for every incident.

### Correct Answer: **B**

**English Explanation:**
The immediate problem is ambiguous tool semantics. Clear descriptions and boundaries help Claude distinguish when each capability is appropriate.

### 中文解说

这是典型的：

**Tool selection failure**

不是：

* security failure；
* authorization failure；
* workflow failure。

所以先修：

**Tool interface**

例如：

```text
search_logs

Search runtime application and infrastructure logs.

Use when:
- investigating runtime errors
- searching exceptions
- correlating events by timestamp

Do not use when:
- inspecting source implementation
```

而：

```text
search_source_code

Search repository source code.

Use when:
- locating implementation
- tracing method calls
- examining configuration in code

Do not use for:
- runtime log events
```

A 只是告诉 Claude：

> “认真一点。”

却没有消除 ambiguity。

D 又走到了另一个极端。

**Exam takeaway:**

> **Ambiguous tool choice → improve tool semantics before adding orchestration complexity.**

---

## Question 3 — Hard Safety Boundary

Apex permits Claude to recommend restarting a production service.

However, the `restart_service` tool must **never execute** unless the incident commander has approved the restart.

Which implementation is BEST?

**A.** Put “NEVER restart without approval” in the system prompt.
**B.** Include ten few-shot examples where Claude waits for approval.
**C.** Require the tool execution layer to verify a valid incident-commander approval before performing the restart.
**D.** Ask Claude to report 100% confidence before restarting.

### Correct Answer: **C**

**English Explanation:**
A mandatory safety boundary around a state-changing production action should be enforced programmatically. Prompting can guide behavior but should not be the sole control.

### 中文解说

这里出现三个非常强的信号：

> **must never**
> **production**
> **restart**

应该马上想到：

**Programmatic enforcement**

结构：

```text
Claude requests restart
        ↓
Approval validator
        ↓
Valid approval?
    /           \
  Yes            No
   ↓              ↓
Execute          Reject
```

A、B 都可以作为 defense-in-depth，但不能作为最终 guarantee。

D 的 confidence 与 permission 没关系。

考试中要特别警惕：

> **Confidence ≠ Authorization**

**Exam takeaway:**

> **Safety-critical state change → enforce outside model discretion.**

---

## Question 4 — Failure Recovery

### Choose TWO.

`query_database` returns:

```json
{
  "error_type": "connection_timeout",
  "retryable": true,
  "retry_after_seconds": 5
}
```

Which TWO actions are MOST appropriate?

**A.** Apply a bounded retry policy, respecting retry guidance where appropriate.
**B.** Immediately escalate the entire incident to a human because any tool failure invalidates agent autonomy.
**C.** If retries are exhausted, propagate structured failure context to the coordinator.
**D.** Retry indefinitely because `retryable` guarantees eventual success.

### Correct Answers: **A and C**

**English Explanation:**
Transient failures should normally be recovered locally using bounded retries. If recovery fails, the parent agent needs structured context to choose another source, continue partially, or escalate.

### 中文解说

`retryable = true` 的意思是：

> **可以尝试恢复**

不是：

> **一直重试到天荒地老**

正确流程：

```text
Transient failure
      ↓
Bounded retry
      ↓
Recovered?
 /           \
Yes           No
 ↓             ↓
Continue     Structured failure
                  ↓
             Coordinator
```

Coordinator 得到的信息最好包括：

```text
error_type
retry_count
attempted_action
partial_results
alternative_available
```

B 属于 **over-escalation**。

D 属于 **unbounded retry**。

**Exam takeaway:**

> **Retryable ≠ retry forever.**

---

## Question 5 — Structured Output vs Semantic Correctness

`LogAgent` returns:

```json
{
  "service": "payment-api",
  "error_count": 52,
  "total_requests": 40,
  "error_rate": 0.20
}
```

The response perfectly matches the JSON Schema.

What is the MOST important additional control?

**A.** No control is necessary because the schema passed.
**B.** Semantic validation of relationships among the extracted/calculated values.
**C.** A larger context window.
**D.** Convert the JSON to natural-language prose before using it.

### Correct Answer: **B**

**English Explanation:**
Schema validation confirms structure and types, not whether values are mutually consistent. Business or semantic validation should detect impossible relationships such as an error count exceeding the total request count.

### 中文解说

JSON 完全合法：

```text
service       → string ✓
error_count   → number ✓
total_requests→ number ✓
error_rate    → number ✓
```

但是：

```text
52 errors
40 total requests
```

明显不合理。

而且：

```text
52 / 40 ≠ 0.20
```

所以：

**Structure = valid**

不代表：

**Meaning = valid**

这是 CCAR-F 很重要的区分：

```text
Schema validation
        ↓
Semantic validation
        ↓
Business use
```

**Exam takeaway:**

> **Schema-valid does not mean semantically valid.**

---

## Question 6 — Long-Running Context

After three hours, the coordinator has accumulated:

* 180 pages of raw logs;
* repeated stack traces;
* obsolete hypotheses;
* multiple failed tool outputs;
* several verified root-cause facts.

Claude starts referring to hypotheses that were already disproven.

What is the BEST response?

**A.** Keep everything because deleting context always reduces accuracy.
**B.** Maintain verified findings in compact structured state while trimming or summarizing obsolete and low-value context.
**C.** Repeat every verified fact after every tool call.
**D.** Restart the investigation from zero.

### Correct Answer: **B**

**English Explanation:**
Long-running investigations benefit from separating durable verified state from transient working context. Pruning low-value material improves salience while structured state preserves important facts.

### 中文解说

这里出现一个很重要的词：

**obsolete hypotheses**

例如：

```text
Hypothesis A: Database overload
→ disproven

Hypothesis B: Memory leak
→ disproven

Verified fact:
Deployment introduced connection-pool bug
```

如果全部历史一直留着：

Claude 后面可能又被 A、B 干扰。

更好的结构：

```text
Working context
      ↓
Verify findings
      ↓
Structured durable state
      ↓
Prune obsolete information
```

所以要记：

> **Agent memory 不应该只是无限增长的 chat history。**

A 是经典的：

**More context = better**

陷阱。

**Exam takeaway:**

> **Preserve verified signal; remove obsolete noise.**

---

## Question 7 — Few-Shot vs Explicit Criteria

`SecurityAgent` is instructed:

> “Report important security issues.”

It reports naming conventions, minor formatting problems, and low-risk warnings while sometimes missing credential exposure.

What is the BEST first improvement?

**A.** Provide explicit criteria defining which security findings are important and which findings should be excluded.
**B.** Immediately add twenty few-shot examples.
**C.** Increase the number of security agents.
**D.** Increase output length.

### Correct Answer: **A**

**English Explanation:**
The core problem is an undefined decision criterion. Explicit inclusion and exclusion criteria should be established before adding examples to refine boundary behavior.

### 中文解说

注意这里和之前的 Few-shot 题区别。

现在 instruction 只有：

> **important security issues**

“important”没有定义。

所以第一步应该：

**Explicit criteria**

例如：

```text
Report:
- exposed credentials
- authentication bypass
- privilege escalation
- injection vulnerabilities

Do not report:
- naming style
- formatting
- cosmetic warnings
```

什么时候 Few-shot 更合适？

如果题目说：

> 已经有明确 criteria，但是 edge cases 仍然不稳定。

这时才优先：

**Few-shot examples**

所以判断顺序可以记成：

```text
Rules unclear?
→ Explicit criteria

Rules clear, boundaries unstable?
→ Few-shot
```

**Exam takeaway:**

> **Fix ambiguity before demonstrating edge cases.**

---

## Question 8 — Provenance

The final incident report says:

> “The outage was caused by connection-pool exhaustion introduced in deployment 8.42.”

Management asks the team to prove how the agent reached that conclusion.

Which architecture BEST supports this requirement?

**A.** Save only the final natural-language report.
**B.** Track claims together with supporting evidence, source/tool result, and relevant timestamps.
**C.** Ask Claude to regenerate an explanation after the incident.
**D.** Increase the report's confidence score.

### Correct Answer: **B**

**English Explanation:**
Provenance preserves the trace from claims back to evidence. This makes conclusions auditable and allows reviewers to verify whether the evidence actually supports them.

### 中文解说

理想结果应该类似：

```text
Claim:
Connection-pool exhaustion caused outage.

Evidence:
Database connection errors increased at 14:02.

Source:
search_logs result #17

Additional evidence:
Deployment 8.42 changed pool configuration.

Source:
search_source_code result #9

Timestamp:
14:00 deployment
14:02 errors begin
```

这就是：

**Provenance**

C 是事后让 Claude“回忆为什么”，不是可靠审计。

模型可能生成一个听起来合理但并非真实执行轨迹的解释。

**Exam takeaway:**

> **Auditability requires preserved evidence, not reconstructed explanations.**

---

## Question 9 — Human Escalation

### Choose TWO.

Which TWO situations provide the STRONGEST reasons for the incident agent to escalate?

**A.** One log-search request times out but reports `retryable=true`.
**B.** Two authoritative sources remain materially contradictory after the defined resolution procedure.
**C.** The system needs to perform an irreversible production action that policy requires a human to approve.
**D.** Claude reports 88% confidence instead of the team's arbitrary 90% target.

### Correct Answers: **B and C**

**English Explanation:**
Escalation is appropriate when material ambiguity remains unresolved or when policy explicitly requires human authorization. Transient recoverable errors should normally be handled locally, and arbitrary uncalibrated confidence thresholds are weaker escalation signals.

### 中文解说

判断 Human Escalation，不要看：

> Claude 有没有一点不确定。

要看：

```text
Policy requirement?
High-risk / irreversible?
Material unresolved ambiguity?
Recovery exhausted?
User explicitly requests human?
```

B 符合：

**material unresolved conflict**

C 符合：

**policy + irreversible action**

A 应先 retry。

D 又是：

**uncalibrated confidence trap**

**Exam takeaway:**

> **Escalation should be driven by risk, policy, and unresolved failure—not arbitrary model confidence.**

---

## Question 10 — Architecture Diagnosis

Apex observes the following failure:

> Claude correctly chooses `restart_service`, provides valid arguments, and correctly explains that restarting is necessary. However, the restart occasionally executes without the required human approval.

Which layer is MOST directly defective?

**A.** Prompt engineering
**B.** Tool-selection semantics
**C.** Programmatic authorization/enforcement
**D.** Context summarization

### Correct Answer: **C**

**English Explanation:**
Claude is already selecting and parameterizing the correct tool. The failure is that the application permits an unauthorized action to execute. Therefore the defective layer is the enforcement boundary.

### 中文解说

这是今天最重要的一题。

逐层排除：

```text
Did Claude choose wrong tool?
→ No.

Did Claude misunderstand task?
→ No.

Were arguments invalid?
→ No.

Did an unauthorized action execute?
→ Yes.
```

所以问题不在：

**Prompt**

也不在：

**Tool description**

而在：

**Enforcement**

考试时不要看到 Claude 出问题就自动选择：

> “Improve the system prompt.”

先问：

> **到底是哪一层坏了？**

---

# Day 7 — Architecture Diagnosis Cheat Sheet

| Symptom                                     | First architectural suspect     |
| ------------------------------------------- | ------------------------------- |
| Wrong tool selected                         | **Tool description / boundary** |
| Wrong output shape                          | **Schema / structured output**  |
| Correct shape, impossible values            | **Semantic validation**         |
| Mandatory sequence skipped                  | **Workflow enforcement**        |
| Unauthorized action executes                | **Programmatic authorization**  |
| Edge cases inconsistent despite clear rules | **Few-shot examples**           |
| Requirement itself is vague                 | **Explicit criteria**           |
| Early facts become unreliable               | **Context management**          |
| Cannot prove a claim                        | **Provenance**                  |
| Temporary external failure                  | **Bounded retry**               |
| Material ambiguity remains unresolved       | **Escalation**                  |

The most useful exam heuristic today is:

> **Diagnose before you optimize.**

A common CCAR-F distractor gives you a genuinely useful technique—such as few-shot prompting, a better `CLAUDE.md`, more context, or another subagent—but applies it to the **wrong architectural layer**.

Before choosing an answer, mentally classify the failure:

> **Guidance? Interface? Orchestration? Enforcement? Validation? Context? Reliability?**

Then choose the option that fixes that layer with the **smallest reliable architectural change**.
