# CCAR-F Daily Practice — Day 15

Today’s set emphasizes **multi-agent orchestration, context engineering, MCP/tool design, structured outputs, deterministic enforcement, reliability, and evaluation**. The distractors are intentionally plausible: identify the **failure layer** before choosing a solution.

**Exam mode:** 10 questions. Target time: **20 minutes**. Questions 4 and 9 are **Choose TWO**.

---

## Scenario: Falcon Financial Research Platform

Falcon Capital uses Claude to prepare institutional investment-research reports.

The system contains:

* `CoordinatorAgent` — decomposes research tasks and synthesizes results.
* `FilingAgent` — analyzes regulatory filings.
* `MarketAgent` — retrieves market information.
* `FinancialAgent` — calculates financial metrics.
* `RiskAgent` — identifies material investment risks.
* `ReviewAgent` — independently reviews important conclusions.

Available capabilities include:

```text
retrieve_filing
search_market_data
get_company_fundamentals
calculate_metrics
retrieve_research_policy
create_research_report
publish_report
```

Research sessions can run for several hours and process hundreds of documents.

---

## Question 1 — Orchestrator Context

`CoordinatorAgent` delegates:

> Determine whether the company's declining operating margin represents a material investment risk.

The coordinator currently sends `RiskAgent` the entire research transcript: 160 pages of filings, market searches, discarded hypotheses, logs, and unrelated company history.

`RiskAgent` becomes slower and sometimes focuses on irrelevant evidence.

What is the BEST improvement?

**A.** Give `RiskAgent` even more context so nothing can possibly be missing.
**B.** Provide a task-specific context package containing the objective, relevant verified facts, applicable criteria, and references to supporting evidence.
**C.** Give `RiskAgent` only the sentence “Analyze risk.”
**D.** Remove all context and rely on the model's general financial knowledge.

### Correct Answer: **B**

**English Explanation:**
A specialist agent should receive **minimum sufficient context**: enough information to complete its bounded task reliably without unnecessary conversational noise.

### 中文解说

这题不是简单地考：

> context 越少越好。

正确原则是：

> **Minimum sufficient context**

应该给：

```text
Objective
Relevant verified facts
Evaluation criteria
Evidence references
```

而不是把 160 页 transcript 全部塞进去。

A 是经典陷阱：

> **More context = better reasoning**

实际上过多无关信息可能导致 **context pollution**。

C 又太少，没有证据和标准。

D 更危险，投资结论必须基于 case-specific evidence，而不是模型的一般知识。

**Key exam takeaway:**

> **Specialist agents need relevant context, not maximum context.**

---

## Question 2 — Agent Delegation Boundary

Falcon needs to calculate:

```text
debt_to_equity = total_debt / shareholder_equity
```

Both input values have already been validated.

Which architecture is BEST?

**A.** Delegate the calculation to `FinancialAgent` and ask it to reason step by step.
**B.** Ask three financial agents to calculate the ratio and use majority voting.
**C.** Calculate the ratio deterministically in code or a calculation tool.
**D.** Ask `CoordinatorAgent` to estimate the ratio from prose.

### Correct Answer: **C**

**English Explanation:**
Once validated numeric inputs and an exact formula are available, deterministic computation is more reliable and efficient than probabilistic agent reasoning.

### 中文解说

这是一个非常重要的 CCAR-F 判断：

> **不是所有任务都应该交给 Agent。**

如果任务是：

```text
Known inputs
+
Exact formula
```

直接代码计算。

Claude/Agent 更适合：

* 从非结构化 filing 中识别数字；
* 判断某条 disclosure 是否重要；
* 分析 conflicting evidence。

但：

```text
100 / 50 = 2
```

没有必要使用 LLM。

A、B 都属于：

**over-agentization**

D 则把精确计算变成估算。

**Key exam takeaway:**

> **Use agents for judgment; use code for exact computation.**

---

## Question 3 — MCP Tool Boundary

Falcon exposes:

```text
market_operation(
    operation,
    ticker,
    query,
    start_date,
    end_date,
    metric,
    report_type,
    output_mode
)
```

The tool performs market search, fundamental lookup, metric calculation, and report creation depending on `operation`.

Claude frequently selects the wrong operation and supplies irrelevant arguments.

What is the BEST redesign?

**A.** Add more examples telling Claude how to populate `operation`.
**B.** Split the overloaded capability into focused tools with distinct semantic responsibilities and narrow schemas.
**C.** Increase the model's temperature.
**D.** Add more optional parameters.

### Correct Answer: **B**

**English Explanation:**
An overloaded tool creates ambiguous capability boundaries and unnecessarily complex schemas. Focused tools make selection and argument generation more reliable.

### 中文解说

看到：

> 一个 Tool 什么都能干

要想到：

**Tool Granularity / Semantic Boundary**

更好的设计：

```text
search_market_data(...)
get_company_fundamentals(...)
calculate_metrics(...)
create_research_report(...)
```

而不是：

```text
operation="maybe_search_or_calculate_or_create"
```

A 的 few-shot 可能有所帮助，但没有修复 API 本身的根本问题。

C、D 甚至可能让问题更严重。

**Key exam takeaway:**

> **Fix ambiguous interfaces before compensating with more prompting.**

---

## Question 4 — Parallel Research

### Choose TWO.

The coordinator needs to:

1. analyze the annual filing;
2. analyze the latest earnings call;
3. retrieve recent market information;
4. synthesize all three into a risk assessment.

Tasks 1–3 are independent.

Which TWO statements are correct?

**A.** Tasks 1–3 are good candidates for parallel execution.
**B.** Task 4 should begin before any upstream result is available.
**C.** Task 4 should execute after the evidence required from tasks 1–3 is available.
**D.** All four tasks should execute independently with no synchronization.

### Correct Answers: **A and C**

**English Explanation:**
Independent evidence-gathering branches can execute concurrently. Synthesis is a dependent **fan-in** step and should wait for the necessary upstream results.

### 中文解说

结构：

```text
Filing ──────────┐
                 │
Earnings Call ───┼──→ Synthesis
                 │
Market Data ─────┘
```

这是典型：

> **Fan-out → Fan-in**

1–3：

**parallel**

4：

**dependent**

B、D 都忽略 dependency。

考试中不要机械地认为：

> Multi-agent → 所有任务并行。

**Key exam takeaway:**

> **Parallelism follows dependency structure.**

---

## Question 5 — Structured Output Boundary

`FilingAgent` must return:

```json
{
  "revenue": 0,
  "operating_income": 0,
  "total_debt": 0
}
```

A filing does not disclose `total_debt`.

Because the schema requires a number, Claude returns:

```json
"total_debt": 0
```

What is the BEST fix?

**A.** Keep zero because JSON requires numeric fields.
**B.** Allow an explicit missing/null state when the source does not provide the value.
**C.** Ask Claude to estimate debt from revenue.
**D.** Replace structured output with unrestricted prose.

### Correct Answer: **B**

**English Explanation:**
The schema should represent legitimate missing information. Zero is a real numeric value and should not be overloaded to mean “unknown.”

### 中文解说

必须记住：

```text
0 ≠ unknown
```

例如：

```text
total_debt = 0
```

真正含义是：

> 公司没有债务。

而：

```text
total_debt = null
```

才可以表达：

> Source 没有提供 / 无法确定。

如果 Schema 不允许 unknown，模型有时会被迫制造 **false precision**。

C 是 hallucination。

D 没必要为了 nullable field 放弃 structured output。

**Key exam takeaway:**

> **Schema design should represent uncertainty honestly.**

---

## Question 6 — Source Conflict

`FilingAgent` reports:

```text
Revenue = $8.2B
Source: audited annual filing
```

`MarketAgent` reports:

```text
Revenue = $8.7B
Source: third-party financial database
```

The difference materially changes a valuation metric.

What should the coordinator do?

**A.** Average the values.
**B.** Choose the larger value because it is more conservative.
**C.** Preserve both values and provenance, apply explicit source-authority rules, and surface the conflict if it cannot be resolved.
**D.** Ask Claude which number looks more realistic.

### Correct Answer: **C**

**English Explanation:**
Material evidence conflicts should remain traceable. Explicit source-authority rules may resolve them; otherwise the disagreement should be surfaced rather than silently collapsed.

### 中文解说

错误答案都很“像人在做判断”：

A：

> 取平均。

没有证据说明真实 Revenue 在中间。

B：

> 选保守的。

“保守”不是 provenance resolution rule。

D：

> Claude 感觉哪个合理。

也不是可靠 source policy。

正确流程：

```text
Value A + Source A
Value B + Source B
        ↓
Conflict detection
        ↓
Source-authority rule
        ↓
Resolve OR preserve uncertainty
```

**Key exam takeaway:**

> **Do not silently reconcile material evidence conflicts.**

---

## Question 7 — Tool Timeout with Side Effect

`publish_report` sends an approved report to external clients.

Claude calls it and receives a network timeout.

The system does not know whether publication succeeded.

What should happen NEXT?

**A.** Immediately call `publish_report` again.
**B.** Treat the timeout as proof that publication failed.
**C.** Reconcile publication state or use idempotent publication semantics before retrying.
**D.** Ask Claude to estimate the probability that publication succeeded.

### Correct Answer: **C**

**English Explanation:**
A timeout after a state-changing operation leaves the outcome ambiguous. Blind retry may duplicate the side effect, so the system should verify state or rely on idempotency.

### 中文解说

这是高频生产可靠性题。

执行过程可能是：

```text
publish_report
      ↓
Server publishes ✓
      ↓
Response lost
      ↓
TIMEOUT
```

所以：

> **Timeout ≠ failure**

如果马上 retry：

```text
Client receives report #1
Client receives report #2
```

因此要考虑：

**Idempotency**

或者：

**State reconciliation**

A 是典型的 **blind retry**。

**Key exam takeaway:**

> **Ambiguous timeout + side effect → never assume failure blindly.**

---

## Question 8 — Evaluation Failure

Falcon evaluates a new `RiskAgent` prompt on 500 examples.

Overall accuracy improves:

```text
Old: 91%
New: 94%
```

But segmented results show:

```text
Routine cases:      96% → 98%
High-risk cases:    89% → 76%
```

High-risk cases determine whether reports require human review.

Should Falcon deploy the new prompt?

**A.** Yes, because overall accuracy increased.
**B.** Yes, because routine cases are more common.
**C.** Not based on the aggregate improvement alone; the critical high-risk regression must be addressed or explicitly accepted against operational requirements.
**D.** Yes, if the new prompt is shorter.

### Correct Answer: **C**

**English Explanation:**
Aggregate metrics can hide regressions in operationally important slices. Evaluation should reflect the cost and importance of failures, not merely overall accuracy.

### 中文解说

这是考试非常喜欢的：

**Aggregate Metric Trap**

表面：

```text
91 → 94
```

很好。

但关键分类：

```text
High Risk
89 → 76
```

严重退化。

而这个类别恰好控制：

> Human Review

所以不能只看 overall。

需要：

**Segmented Evaluation**

尤其关注：

* high-risk class；
* rare class；
* safety-critical class；
* expensive failure class。

**Key exam takeaway:**

> **Overall improvement can conceal critical regression.**

---

## Question 9 — High-Impact Publication

### Choose TWO.

Falcon allows Claude to prepare investment reports autonomously, but publishing a report externally requires human approval.

Which TWO controls provide the strongest layered design?

**A.** Clearly instruct Claude that external publication requires approval.
**B.** Require `publish_report` to verify valid approval at execution time.
**C.** Allow publication whenever Claude reports confidence above 95%.
**D.** Let retrieved documents redefine the approval policy.

### Correct Answers: **A and B**

**English Explanation:**
Model guidance reduces inappropriate publication attempts, while programmatic authorization prevents execution without valid approval. Together they provide defense in depth.

### 中文解说

如果题目只问：

> **STRONGEST guarantee**

答案通常是：

**B**

但这题问：

> **Choose TWO / layered design**

所以：

```text
Layer 1
Instruction
→ Reduce inappropriate requests

Layer 2
Execution-time authorization
→ Prevent unauthorized execution
```

A + B = **Defense in Depth**

C 再次出现：

> **Confidence ≠ Authorization**

D 则违反 trust boundary。

---

## Question 10 — Failure-Layer Diagnosis

Falcon observes:

> `RiskAgent` correctly concludes that a company requires human review.
> Its structured output contains `"requires_review": true`.
> The applicable policy is current.
> The application receives the structured output successfully.
> However, a coding bug checks `requires_review == "true"` as a string instead of the Boolean value `true`, so no review case is created.

Which layer MOST directly failed?

**A.** Prompt engineering
**B.** Model reasoning
**C.** Application integration/type handling
**D.** Context management

### Correct Answer: **C**

**English Explanation:**
Claude produced the correct decision in the required structured representation. The defect occurs when application code incorrectly interprets that representation.

### 中文解说

这题非常适合训练 **failure-layer diagnosis**。

逐层看：

```text
Reasoning          ✓
Policy             ✓
Structured output  ✓
Transport          ✓

Application logic  ✗
```

Claude 给的是：

```json
"requires_review": true
```

程序却判断：

```text
requires_review == "true"
```

一个是：

**Boolean**

另一个是：

**String**

所以问题在：

**Application Integration / Type Handling**

不要因为这是 AI 系统，就默认：

> “修改 Prompt。”

Prompt 根本没错。

---

# Day 15 — Failure-Layer Decision Map

今天建议把下面这张表真正背下来：

| Symptom                                               | First suspect                        |
| ----------------------------------------------------- | ------------------------------------ |
| Specialist distracted by irrelevant history           | **Context packaging**                |
| Exact numeric calculation                             | **Deterministic code/tool**          |
| Generic tool has many operation modes                 | **Tool boundary**                    |
| Independent evidence gathering                        | **Parallel fan-out**                 |
| Missing value becomes zero                            | **Schema/uncertainty modeling**      |
| Authoritative sources disagree                        | **Provenance + conflict policy**     |
| Write operation times out                             | **Idempotency/state reconciliation** |
| Overall eval improves, critical class worsens         | **Segmented regression**             |
| Protected external action                             | **Execution-time authorization**     |
| Model output correct, application interprets it wrong | **Integration layer**                |

## Today's Most Important Exam Technique

When a system produces the wrong final outcome, walk backward:

```text
Final action
    ↑
Application integration
    ↑
Validation / enforcement
    ↑
Structured output
    ↑
Agent reasoning
    ↑
Context / evidence
    ↑
Tool / source
```

Ask:

> **Where was the first point at which the state became wrong?**

Fix **that layer**.

Do not automatically modify the prompt simply because Claude participates in the system.

### Eight sentences to memorize

> **Minimum sufficient context beats maximum context.**

> **Exact computation belongs in deterministic code.**

> **Narrow tools reduce semantic ambiguity.**

> **Independent work can fan out; synthesis must fan in.**

> **Unknown is not zero.**

> **Material source conflicts require provenance-aware resolution.**

> **Timeout does not prove a side-effecting action failed.**

> **When Claude is correct but the system is wrong, inspect the integration layer.**
