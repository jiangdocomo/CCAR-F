# CCAR-F Daily Practice — Day 21

Today’s set focuses on **agent observability and production diagnosis**: distributed tracing, correlation IDs, structured logging, metrics vs. traces, SLOs, token/latency attribution, tool-call observability, privacy-aware telemetry, and detecting silent failures.

**Exam mode:** 10 questions. **Target time: 20 minutes.** Questions 4 and 9 are **Choose TWO**.

---

## Scenario: Nexus Global Support Agent

Nexus operates a Claude-based enterprise support platform.

The system contains:

* `CoordinatorAgent` — receives and decomposes customer requests.
* `KnowledgeAgent` — retrieves product documentation.
* `DiagnosticAgent` — investigates technical problems.
* `ResolutionAgent` — performs permitted remediation.
* `ReviewAgent` — reviews high-impact actions.

Available capabilities include:

```text
search_documentation
search_logs
get_service_health
get_customer_config
restart_service
create_support_case
send_customer_message
```

A single customer request may trigger several agents and dozens of tool calls.

Nexus now wants to improve production observability because engineers can see that some requests fail, but often cannot determine **where, why, or how much each component contributed to latency and cost**.

---

## Question 1 — End-to-End Correlation

A customer request triggers:

```text
CoordinatorAgent
   ↓
KnowledgeAgent
   ↓
DiagnosticAgent
   ↓
search_logs
   ↓
get_service_health
   ↓
ResolutionAgent
```

Each component writes logs, but engineers cannot reliably determine which log entries belong to the same customer request.

What is the BEST improvement?

**A.** Add a shared correlation/trace identifier that propagates across agents and tool calls.  
**B.** Make every log message longer.  
**C.** Store all logs inside Claude's context.  
**D.** Increase model temperature.  

### Correct Answer: **A**

**English Explanation:**
A propagated correlation or trace identifier allows events from multiple agents, tools, and services to be reconstructed as one end-to-end execution.

### 中文解说

这是今天第一个重要概念：

**Correlation ID / Trace ID**

一个 Request 可能经过：

```text
request
  ↓
Agent A
  ↓
Agent B
  ↓
Tool X
  ↓
Tool Y
```

如果每个地方只记录：

```text
Tool executed.
Agent finished.
Search completed.
```

事后很难知道这些日志属于哪一次请求。

应该传播类似：

```text
trace_id = T-94821
```

于是：

```text
T-94821 Coordinator started
T-94821 KnowledgeAgent started
T-94821 search_logs called
T-94821 ResolutionAgent finished
```

B 只是让日志更长，不解决关联问题。

C 会污染模型 Context，而且 observability storage 与 active context 是不同问题。

**Key exam takeaway:**

> **Use correlation identifiers to connect distributed agent activity into one execution trace.**

---

## Question 2 — Metrics vs. Traces

Nexus knows:

```text
p95 request latency = 42 seconds
```

Engineers need to determine **which stage** is responsible for the latency.

What should they inspect NEXT?

**A.** Only the aggregate p95 metric.  
**B.** End-to-end traces containing timing information for model calls, agents, and tools.  
**C.** The number of characters in the final answer.  
**D.** Claude's confidence.  

### Correct Answer: **B**

**English Explanation:**
Metrics reveal that a performance problem exists, while traces help identify where time was spent across an individual distributed execution.

### 中文解说

要区分：

### Metric

回答：

> **有没有问题？问题多严重？**

例如：

```text
p50 = 12s
p95 = 42s
error rate = 3%
```

### Trace

回答：

> **这一次请求到底慢在哪里？**

例如：

```text
Coordinator       2s
KnowledgeAgent    5s
search_logs      25s  ← bottleneck
DiagnosticAgent   6s
Synthesis         4s
```

所以可以记：

> **Metrics detect. Traces diagnose.**

A 已经知道 42 秒，但不知道为什么。

**Key exam takeaway:**

> **Aggregate metrics tell you that something is wrong; traces help explain where it went wrong.**

---

## Question 3 — Structured Logging

A tool currently logs:

```text
Something went wrong while searching logs.
```

Possible failures include:

```text
TIMEOUT
RATE_LIMIT
ACCESS_DENIED
INVALID_QUERY
```

Which logging design is MOST useful?

**A.** Record structured fields such as error type, tool, trace ID, duration, and retry status.  
**B.** Replace the message with “ERROR!!!”.  
**C.** Hide the failure because logs increase storage costs.  
**D.** Ask Claude to remember the error for later.  

### Correct Answer: **A**

**English Explanation:**
Structured telemetry makes failures searchable, aggregatable, and attributable to specific operations and executions.

### 中文解说

以前我们从 Agent recovery 角度学过：

**Structured Error**

今天从 observability 角度再看：

**Structured Logging**

例如：

```text
trace_id: T-94821
tool: search_logs
error_type: RATE_LIMIT
duration_ms: 310
retryable: true
attempt: 2
```

这样才能统计：

> 最近一天 `search_logs` 的 RATE_LIMIT 有多少？

如果全部只是：

```text
Something went wrong.
```

机器很难可靠分析。

**Key exam takeaway:**

> **Logs should contain machine-queryable operational semantics, not only human-readable prose.**

---

## Question 4 — Useful Production Metrics

### Choose TWO.

Nexus wants to determine whether a new agent architecture improves production performance.

Which TWO measurements are MOST useful?

**A.** End-to-end task success rate on meaningful outcomes.  
**B.** Latency and token/cost attribution by major component or operation.  
**C.** Number of adjectives in Claude's responses.  
**D.** Number of source-code files in the repository.  

### Correct Answers: **A and B**

**English Explanation:**
Production architecture should be measured against outcomes and operational characteristics such as reliability, latency, and cost. Component attribution helps identify trade-offs and bottlenecks.

### 中文解说

Architecture 优化不能只看：

```text
Tokens ↓
```

也不能只看：

```text
Latency ↓
```

必须结合：

```text
Quality / task success
Latency
Cost
Reliability
```

例如：

```text
Architecture A
success = 97%
latency = 30s
cost = $0.08

Architecture B
success = 83%
latency = 10s
cost = $0.02
```

B 虽然便宜快，但不一定更好。

**Key exam takeaway:**

> **Optimize quality, latency, cost, and reliability together—not one metric in isolation.**

---

## Question 5 — SLO

Nexus defines:

> “99% of routine support requests should complete successfully within 30 seconds.”

What is this MOST naturally an example of?

**A.** SLO  
**B.** Prompt injection  
**C.** MCP Resource  
**D.** Few-shot example  

### Correct Answer: **A**

**English Explanation:**
A Service Level Objective defines a measurable reliability or performance target for a service over a specified class of operations.

### 中文解说

今天要记一个生产术语：

**SLO — Service Level Objective**

例如：

> 99% requests < 30 sec

或者：

> 99.9% availability

它是：

**明确、可测量的服务目标。**

考试里还可能看到：

**SLI — Service Level Indicator**

这是实际测量指标，例如：

```text
successful_requests / total_requests
```

可以简单记：

```text
SLI = What do we measure?
SLO = What target do we want?
```

**Key exam takeaway:**

> **An SLO turns reliability expectations into measurable operational targets.**

---

## Question 6 — Silent Failure

`ResolutionAgent` calls:

```text
restart_service("billing-api")
```

The tool returns:

```text
status = SUCCESS
```

Claude reports:

> “The incident has been resolved.”

Production monitoring later shows that `billing-api` restarted successfully but immediately began failing again.

What observability improvement MOST directly helps detect this class of failure?

**A.** Observe the intended post-action outcome, not merely tool-call completion.  
**B.** Make Claude's confirmation message longer.  
**C.** Count the number of tool calls.  
**D.** Remove service-health monitoring.  

### Correct Answer: **A**

**English Explanation:**
Operational success should be tied to the intended outcome. A successful action invocation does not guarantee that the underlying incident was resolved.

### 中文解说

这是：

**Silent Failure**

表面：

```text
restart tool = SUCCESS
Claude says = FIXED
```

实际上：

```text
service health = BAD
```

所以要观察：

```text
Action
   ↓
Immediate result
   ↓
Postcondition
```

例如：

```text
restart_service
      ↓
wait / observe
      ↓
get_service_health
      ↓
healthy?
```

再次强化：

> **Tool success ≠ Task success**

**Key exam takeaway:**

> **Observability should measure desired postconditions, not just completed operations.**

---

## Question 7 — Token Attribution

Nexus's total token usage increases by 70%.

Engineers know the overall total but cannot determine the cause.

What should they do?

**A.** Attribute token consumption to relevant model calls, agents, context construction, and major workflow stages.  
**B.** Assume the largest model is responsible.  
**C.** Remove random system instructions.  
**D.** Reduce every prompt by exactly 70%.  

### Correct Answer: **A**

**English Explanation:**
Optimization requires attribution. Breaking consumption down by component helps distinguish expensive reasoning, duplicated context, oversized tool results, and inefficient orchestration.

### 中文解说

这是：

**Attribution**

看到：

```text
Total token +70%
```

只能说明：

> 有问题。

不能说明：

> 哪里有问题。

可能是：

```text
Coordinator context duplicated
Tool result too large
Subagent handoff too large
More retries
More reasoning turns
```

所以应该拆：

```text
Coordinator: 20%
KnowledgeAgent: 15%
DiagnosticAgent: 40%
Tool results: 20%
Other: 5%
```

然后再优化。

这和 Day 14 的原则一致：

> **Measure → locate → optimize → re-evaluate**

**Key exam takeaway:**

> **You cannot optimize what you cannot attribute.**

---

## Question 8 — Privacy-Aware Telemetry

Nexus wants detailed traces for debugging.

Some tool results contain:

* customer email addresses;
* authentication tokens;
* financial account identifiers.

What is the BEST approach?

**A.** Log every raw value because observability is more important than privacy.  
**B.** Collect the minimum telemetry needed for diagnosis and redact or avoid unnecessary sensitive data.  
**C.** Put authentication tokens into trace IDs.  
**D.** Disable all observability.  

### Correct Answer: **B**

**English Explanation:**
Observability should be designed with data minimization and secret handling in mind. Useful operational telemetry does not require indiscriminately storing sensitive values.

### 中文解说

Observability 也不是：

> 越多越好。

要考虑：

**Data Minimization**

例如需要知道：

```text
tool = search_customer
status = ACCESS_DENIED
```

不一定需要记录：

```text
customer_password
API_SECRET
full bank account
```

特别是 Secret：

> 尽量根本不要进入 telemetry。

A 是典型：

> 为了 debug 什么都记。

风险很高。

D 又走向另一个极端。

**Key exam takeaway:**

> **Observability should be sufficient for diagnosis without becoming a sensitive-data collection system.**

---

## Question 9 — Debugging a Regression

### Choose TWO.

After a deployment:

```text
Task success: 96% → 95%
p95 latency: 24s → 51s
Cost/request: +55%
```

Which TWO actions are BEST first steps?

**A.** Compare pre- and post-deployment traces to identify stages with increased latency or token usage.  
**B.** Segment metrics by task type, tool, and failure category to locate concentrated regressions.  
**C.** Immediately replace the model without diagnosis.  
**D.** Remove all validation because it may consume time.  

### Correct Answers: **A and B**

**English Explanation:**
A regression should first be localized. Trace comparison and segmented metrics help determine whether the cause is model usage, context growth, retries, a slow dependency, or another workflow change.

### 中文解说

这里最重要的是：

> **先定位，不要先猜。**

现在知道：

```text
Latency ↑↑
Cost ↑↑
Success slight ↓
```

可能原因很多：

```text
More model turns?
Larger context?
Tool timeout + retries?
Slow MCP service?
Subagent duplication?
```

A：

**Trace comparison**

找到请求内部哪里变慢。

B：

**Segmentation**

找到哪些类别特别异常。

C 是 premature optimization / diagnosis。

D 可能让 reliability 更差。

**Key exam takeaway:**

> **Localize regressions before changing architecture.**

---

## Question 10 — Failure-Layer Diagnosis

Nexus observes:

> Overall error rate increased from 1% to 8%.
> Traces show Claude chooses the correct tool and supplies valid arguments.
> `search_logs` now takes 18 seconds instead of 2 seconds and frequently returns `TIMEOUT`.
> Model latency is unchanged.

Which layer should engineers investigate FIRST?

**A.** Prompt engineering  
**B.** Model reasoning  
**C.** Tool/dependency performance  
**D.** Few-shot examples  

### Correct Answer: **C**

**English Explanation:**
The trace has already localized the regression to the external tool/dependency. Prompt changes are unlikely to address a service whose latency and timeout rate have degraded.

### 中文解说

这就是 Observability 真正的价值：

如果没有 trace，可能有人会说：

> Claude 最近是不是变差了？

但数据已经显示：

```text
Tool selection    ✓
Arguments         ✓
Model latency     ✓

search_logs
2s → 18s          ✗
TIMEOUT ↑         ✗
```

所以首先查：

**Tool / dependency**

例如：

* service capacity；
* network；
* rate limiting；
* backend query；
* timeout configuration。

不要什么问题都修改 Prompt。

---

# Day 21 — Observability Decision Map

| Signal                                | Think first                         |
| ------------------------------------- | ----------------------------------- |
| Cannot connect logs from one request  | **Trace/correlation ID**            |
| Know system is slow but not where     | **Distributed trace**               |
| Errors are only free-form text        | **Structured logging**              |
| Need measurable reliability target    | **SLO**                             |
| Tool succeeds but task remains broken | **Postcondition monitoring**        |
| Total tokens suddenly increase        | **Cost/token attribution**          |
| Logs contain secrets/PII              | **Data minimization/redaction**     |
| Deployment causes latency regression  | **Trace comparison + segmentation** |
| Model looks correct, tool gets slow   | **Dependency diagnosis**            |

## SLI vs. SLO — Memorize This

> **SLI:** What are we measuring?

Example:

```text
Percentage of successful requests completed within 30 seconds
```

> **SLO:** What result do we require?

Example:

```text
99% of requests must satisfy that SLI.
```

Think:

```text
SLI = measurement
SLO = target
```

## Day 21 Common Trap

When production performance gets worse, four explanations may all sound plausible:

> Prompt problem
> Model problem
> Tool problem
> Orchestration problem

Do not guess.

Use:

```text
Metrics
   ↓
Detect

Traces
   ↓
Localize

Structured logs
   ↓
Explain

Environment state
   ↓
Verify outcome
```

Then fix the layer supported by evidence.

### Ten sentences to memorize

> **Correlation IDs connect distributed agent activity.**

> **Metrics detect; traces diagnose.**

> **Structured logs make failures queryable.**

> **SLOs define measurable reliability targets.**

> **Tool completion does not prove outcome completion.**

> **Attribute latency and cost before optimizing them.**

> **Observe postconditions for state-changing actions.**

> **Telemetry should minimize unnecessary sensitive data.**

> **Segment regressions before changing architecture.**

> **Observability helps you fix the failing layer instead of guessing.**

