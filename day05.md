# CCAR-F Daily Practice — Day 5

Today’s set focuses primarily on **Domain 5: Context Management & Reliability**, while mixing in orchestration, tool reliability, provenance, escalation, and confidence calibration.

**Exam mode:** 8 questions. Target time: **16 minutes**. Answer all questions before reading the explanations.

---

## Scenario: Helios Research Agent

**Helios Capital operates a Claude-based research system that analyzes companies for investment analysts.**

A coordinator agent delegates work to specialist agents that:

* retrieve company filings;
* search financial databases;
* compare competing sources;
* calculate financial metrics;
* summarize management commentary;
* identify risks;
* produce a final research report.

A research session may run for several hours and generate large amounts of tool output.

After deployment, Helios observes several reliability problems:

* long-running investigations gradually become less precise;
* important facts discovered early are sometimes forgotten;
* conflicting sources are sometimes silently reconciled;
* specialist-agent failures are difficult for the coordinator to diagnose;
* analysts need to know which evidence supports important claims;
* too many cases are being escalated to humans.

---

## Question 1 — Context Degradation

A research agent correctly identifies early in a session that:

> “The company discontinued Product X in Q2.”

After dozens of tool calls and large financial-document outputs, the agent later states:

> “Product X appears to remain an important part of the company's portfolio.”

The context window has **not** reached its maximum size.

What is the MOST likely architectural issue?

**A.** Context degradation caused by excessive irrelevant or low-value information.
**B.** The model cannot reason about products.
**C.** Claude requires a larger context window whenever more than ten tools are used.
**D.** The original fact should have been repeated after every message.

### Correct Answer: **A**

**English Explanation:**
Context quality can degrade before the context window is technically full. Large amounts of tool output, duplicated information, and unrelated history can reduce the effective salience of important facts.

### 中文解说

这题的核心陷阱是：

> **Context window 没满，所以不可能是 context 问题。**

这是错的。

Context 有两个不同问题：

```text
Capacity
→ 能不能装下？

Quality
→ 重要信息还能不能保持足够显著？
```

即使容量还有剩余，大量：

* logs；
* raw tool output；
* 重复内容；
* 无关历史；

也可能让早期的重要事实越来越“不显眼”。

B 没有依据。

C 把“工具数量”机械地和 context window 联系起来。

D 也不是合理设计。每轮重复全部事实只会进一步增加 context。

**Exam takeaway:**

> **A context can become ineffective before it becomes full.**

---

## Question 2 — Structured Case Facts

During an investigation, the agent establishes several important facts:

```text
Ticker: HLC
Fiscal year: 2026
Revenue: $4.2B
Debt: $1.1B
Product X: discontinued
Primary filing: FY2026 10-K
```

These facts will be needed repeatedly during a long research session.

What is the BEST way to preserve them?

**A.** Depend entirely on the original conversation history.
**B.** Maintain a compact structured state containing the important verified facts.
**C.** Repeat every previous message before every model call.
**D.** Store only the final answer and discard intermediate facts.

### Correct Answer: **B**

**English Explanation:**
Important durable facts should be represented explicitly in a compact structured state rather than relying solely on their continued salience within an increasingly long conversation.

### 中文解说

这是 Domain 5 非常重要的模式：

> **Conversation history ≠ reliable database**

对于长期 Agent，应把关键事实从长对话里“提取”出来：

```text
Long conversation
       ↓
Verified facts
       ↓
Structured state
       ↓
Future agent calls
```

例如：

```json
{
  "ticker": "HLC",
  "fiscal_year": 2026,
  "revenue": 4200000000,
  "debt": 1100000000,
  "product_x_status": "discontinued"
}
```

A 的问题就是依赖 context salience。

C 会造成 context explosion。

D 会丢掉之后推理需要的重要依据。

**Exam takeaway:**

> **Preserve durable facts as structured state, not merely conversation text.**

---

## Question 3 — Provenance

The final report states:

> “Management expects operating margins to improve materially next year.”

An analyst asks:

> “Where did this claim come from?”

The system cannot determine whether Claude obtained it from an earnings call, an annual report, or a news article.

What architectural improvement is MOST appropriate?

**A.** Ask Claude to sound less confident.
**B.** Track provenance linking important claims to their supporting sources and evidence.
**C.** Increase the number of research subagents.
**D.** Ask Claude to regenerate the entire report.

### Correct Answer: **B**

**English Explanation:**
Provenance preserves the relationship between claims and their evidence. This enables auditing, verification, conflict resolution, and source-aware reporting.

### 中文解说

**Provenance** 是非常值得记住的考试英语词。

意思可以理解为：

> **这个结论从哪里来的？**

理想结构不是只保存：

```text
Claim:
Margins will improve.
```

而是：

```text
Claim
  ↓
Source
  ↓
Evidence
  ↓
Date / location
```

例如：

```text
Claim:
Operating margins are expected to improve.

Source:
Q2 Earnings Call

Evidence:
Management guidance...

Date:
2026-07-25
```

A 只是改变语气，不能回答来源问题。

C 增加 Agent 反而可能增加来源管理难度。

D 重新生成仍可能丢失来源关系。

**Exam takeaway:**

> **Auditable claims require provenance.**

---

## Question 4 — Conflicting Sources

Two reliable sources disagree.

**Source A — Company filing**

> Revenue: $4.20 billion

**Source B — Financial database**

> Revenue: $4.35 billion

Claude currently selects $4.35 billion because it “looks more recent” and removes the conflicting value from the report.

What is the BEST approach?

**A.** Always choose the larger value.
**B.** Preserve both sources, identify the conflict, and resolve it using defined source-authority rules or explicitly report the uncertainty.
**C.** Average the two values.
**D.** Ask Claude to choose whichever value seems more plausible.

### Correct Answer: **B**

**English Explanation:**
Conflicting evidence should not be silently collapsed. The system should preserve provenance, apply explicit source-priority rules when available, and surface unresolved uncertainty when necessary.

### 中文解说

这是典型的 reliability 题。

Agent 不应该：

> “我感觉 B 比较新，所以就用 B。”

更不能：

> `(4.20 + 4.35) / 2`

因为两个来源冲突并不代表真实值是平均数。

正确设计：

```text
Source A ── $4.20B
             │
             ├── Conflict detected
             │
Source B ── $4.35B
             ↓
Source authority rule
             ↓
Resolve OR report uncertainty
```

例如系统可能定义：

```text
Audited filing
> company press release
> third-party database
```

如果能够根据规则解决，就解决。

如果不能：

> **Preserve the disagreement.**

**Exam takeaway:**

> **Do not hide conflicting evidence.**

---

## Question 5 — Failure Propagation

A financial-analysis subagent fails while retrieving debt data.

It currently returns:

```text
FAILED
```

The coordinator therefore does not know whether it should retry, use another source, produce a partial report, or escalate.

Which response would BEST improve orchestration?

**A.**

```text
FAILED AGAIN
```

**B.**

```json
{
  "error_type": "data_source_timeout",
  "attempted_source": "DebtDB",
  "retryable": true,
  "retry_count": 2,
  "partial_results": {
    "short_term_debt": 300000000
  }
}
```

**C.**

```text
Sorry, something went wrong.
```

**D.** Return a fabricated debt value so the coordinator can continue.

### Correct Answer: **B**

**English Explanation:**
Structured failure information allows the coordinator to reason about recovery. Error type, attempted action, retryability, retry history, and partial results all support better orchestration decisions.

### 中文解说

和 Tool Error 类似，Multi-Agent 的 failure 也必须：

> **actionable**

Coordinator 真正需要知道的是：

```text
发生了什么？
尝试了什么？
能不能重试？
已经重试几次？
有没有部分结果？
有没有替代方案？
```

B 提供了这些信息。

于是 Coordinator 可以判断：

```text
Retry DebtDB?
       ↓
Use another source?
       ↓
Continue with partial result?
       ↓
Escalate?
```

A、C 信息量太低。

D 是严重错误：不能为了 workflow 继续运行而 hallucinate 数据。

**Exam takeaway:**

> **Propagate enough failure context for the parent agent to make a recovery decision.**

---

## Question 6 — Human Escalation

Helios currently escalates a research case whenever a subagent reports confidence below 90%.

Human reviewers complain that most escalated cases could have been resolved automatically.

Which design is BEST?

**A.** Escalate every case with any uncertainty.
**B.** Define explicit escalation criteria based on risk, policy requirements, unresolved ambiguity, failed recovery, and user requests.
**C.** Never escalate because Claude should remain autonomous.
**D.** Lower the confidence threshold from 90% to 89%.

### Correct Answer: **B**

**English Explanation:**
Human escalation should be governed by meaningful operational criteria rather than an arbitrary confidence threshold alone. Risk, policy, irreversibility, unresolved ambiguity, and failed automated recovery are stronger signals.

### 中文解说

这题非常容易误选“confidence threshold”。

问题是：

> Claude 说自己 confidence 89%，真的意味着有 11% 错误率吗？

不一定。

所以不能简单：

```text
confidence < 90
→ Human
```

更合理的是提前定义：

```text
Escalate if:
- user explicitly requests a human;
- policy requires human approval;
- high-risk irreversible action;
- required information remains unavailable;
- sources have unresolved material conflict;
- automated recovery is exhausted.
```

A 会造成 **over-escalation**。

C 是另一个极端。

D 从 90 改成 89 没解决架构问题。

**Exam takeaway:**

> **Escalation should be policy-driven, not anxiety-driven.**

---

## Question 7 — Confidence Calibration

A document-extraction agent reports:

```text
Predicted confidence: 95%
```

Across 10,000 labeled examples, outputs assigned approximately 95% confidence are actually correct only 78% of the time.

What does this MOST strongly indicate?

**A.** The model is well calibrated.
**B.** The model is overconfident and its confidence estimates require calibration against observed outcomes.
**C.** The model should simply report 100% confidence.
**D.** Confidence scores automatically become accurate with more context.

### Correct Answer: **B**

**English Explanation:**
Confidence is calibrated when predicted confidence corresponds reasonably to observed correctness. A 95% prediction associated with only 78% actual accuracy indicates overconfidence.

### 中文解说

**Confidence calibration** 要理解，而不是只背单词。

如果模型说：

```text
Confidence = 95%
```

长期实际结果应该大约：

```text
100 个类似预测
≈ 95 个正确
```

才可以说比较 calibrated。

现在实际只有：

```text
78 / 100 correct
```

说明：

> **Overconfident**

最重要的是：

**模型自己报的 confidence 不能直接当概率使用。**

应该拿：

```text
Predicted confidence
        ↓
Historical labeled data
        ↓
Observed accuracy
        ↓
Calibration
```

C 完全相反。

D 也没有依据。

**Exam takeaway:**

> **Self-reported confidence is not automatically calibrated probability.**

---

## Question 8 — Aggregate Metrics Trap

Helios evaluates an extraction system and obtains:

```text
Overall accuracy: 96%
```

Field-level results are:

```text
Company name:      99.8%
Ticker:            99.7%
Revenue:           97.5%
Debt:              92.0%
Risk classification: 71.0%
```

The risk classification field determines whether a case receives enhanced human review.

What is the BEST conclusion?

**A.** The system is production-ready because overall accuracy exceeds 95%.
**B.** Overall accuracy hides a critical weak category; evaluate performance by field and operational risk before automating the decision.
**C.** Delete the risk-classification field so overall accuracy increases.
**D.** Average only company name and ticker accuracy.

### Correct Answer: **B**

**English Explanation:**
Aggregate metrics can conceal poor performance on rare or high-impact categories. Evaluation should be segmented by field, class, document type, or risk level when those distinctions matter operationally.

### 中文解说

这是非常实用的 reliability 思维。

96% 看起来很好，但：

```text
Risk classification = 71%
```

而偏偏这个字段：

> **决定是否进入加强人工审查**

也就是说它的业务影响很大。

Overall accuracy 可能被大量简单字段“冲高”。

例如：

```text
Company name       → 很容易 → 数据很多
Ticker             → 很容易 → 数据很多
Risk classification→ 很难   → 数据较少
```

最终：

```text
Overall = 96%
```

并不能说明最关键的任务可靠。

C 是典型的“为了 metric 好看修改 measurement”。

**Exam takeaway:**

> **Aggregate performance can hide operationally critical failures.**

---

# Day 5 — Quick Review

今天建议重点记这 8 个英文判断：

| Exam signal                       | Preferred pattern                           |
| --------------------------------- | ------------------------------------------- |
| Long context becomes less precise | **Context degradation**                     |
| Important facts needed repeatedly | **Structured state / case facts**           |
| “Where did this claim come from?” | **Provenance**                              |
| Sources disagree                  | **Preserve conflict + authority rules**     |
| Subagent fails                    | **Structured failure propagation**          |
| Too many human handoffs           | **Explicit escalation criteria**            |
| Model says “95% confident”        | **Validate calibration empirically**        |
| Overall metric looks excellent    | **Check segmented / high-risk performance** |

## The Three Most Important Traps Today

**Trap 1 — “The context window isn't full.”**

That does **not** mean context management is healthy.

Think:

> **Context capacity ≠ context quality**

**Trap 2 — “Claude is 95% confident.”**

That does **not** mean:

> **95% probability of being correct.**

Think:

> **Confidence must be calibrated against labeled outcomes.**

**Trap 3 — “Overall accuracy is 96%.”**

Before selecting an answer that says the system is ready, look for:

* rare classes;
* high-risk fields;
* specific document types;
* failure categories.

Think:

> **Aggregate metrics can hide critical weaknesses.**

### Five sentences to memorize today

> **Preserve important facts outside the conversational noise.**

> **Claims should remain traceable to evidence.**

> **Conflicts should be surfaced, not silently erased.**

> **Escalation should follow explicit criteria.**

> **Reliability must be measured where failure actually matters.**

This completes a first focused pass across the five major CCAR-F domains from Days 1–5. The next practice set should therefore shift toward **mixed-domain exam scenarios**, where the harder part is deciding whether a problem belongs to **prompting, tool design, orchestration, programmatic enforcement, structured output, or reliability** rather than being told the domain in advance.
