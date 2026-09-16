# CCAR-F Daily Practice — Day 16

Today’s set focuses on **production architecture decisions under ambiguity**: MCP capability boundaries, multi-agent handoffs, structured state, retry classification, human approval, context engineering, evaluation, and deterministic safeguards.

**Exam mode:** 10 questions. **Target time: 20 minutes.** Questions 4 and 9 are **Choose TWO**.

---

## Scenario: Helios Corporate Treasury Agent

Helios Group uses Claude to assist its corporate treasury team with cash management and payment investigations.

The system contains:

* `CoordinatorAgent` — orchestrates investigations.
* `PaymentAgent` — analyzes payment records.
* `PolicyAgent` — retrieves treasury policies.
* `CounterpartyAgent` — investigates counterparties.
* `RiskAgent` — evaluates payment risk.
* `ReviewAgent` — independently reviews high-risk cases.

Available capabilities include:

```text
search_payments
get_counterparty
retrieve_treasury_policy
calculate_exposure
create_review_case
submit_payment
cancel_payment
```

`search_payments`, `get_counterparty`, and `retrieve_treasury_policy` are read-only.

`submit_payment` and `cancel_payment` change financial state and therefore require stronger controls.

---

## Question 1 — Agent Handoff

`CoordinatorAgent` asks `RiskAgent`:

> Determine whether Payment P-4831 requires enhanced review.

The coordinator currently passes only:

```text
payment_id = P-4831
```

`RiskAgent` repeatedly performs searches that the coordinator already completed.

Which improvement is BEST?

**A.** Pass the entire conversation transcript to `RiskAgent`.  
**B.** Pass a bounded handoff containing the objective, relevant verified facts, applicable policy criteria, and evidence references.  
**C.** Tell `RiskAgent` to search harder.  
**D.** Increase temperature so `RiskAgent` explores more possibilities.  

### Correct Answer: **B**

**English Explanation:**
A good agent handoff contains the information necessary to perform the delegated task without forcing the specialist to rediscover known facts or process unrelated conversation history.

### 中文解说

这题考：

**Agent Handoff / Context Packaging**

现在 Coordinator 已经做过调查，却只传：

```text
P-4831
```

RiskAgent 只能重新搜索。

但 A 又走到另一个极端：

> 整个 transcript 全传。

更好的 handoff：

```text
Objective:
Determine whether enhanced review is required.

Verified facts:
- Amount = $2.4M
- Counterparty jurisdiction = X
- Previous payments = 3

Policy:
- Policy version 24
- Relevant rules R7, R11

Evidence:
- payment-result-17
- counterparty-result-8
```

这样减少重复 Tool Call，也降低 context noise。

**Key exam takeaway:**

> **A handoff should transfer task-relevant state, not merely an identifier or the entire transcript.**

---

## Question 2 — Read vs Write Capability

`CounterpartyAgent` only investigates counterparty identity and history.

Its MCP connection currently exposes:

```text
get_counterparty
search_payments
submit_payment
cancel_payment
```

What is the BEST security improvement?

**A.** Keep every tool because Claude may need them unexpectedly.  
**B.** Remove unnecessary state-changing capabilities from the agent's accessible tool set.  
**C.** Keep all tools but describe `submit_payment` as “dangerous.”  
**D.** Ask Claude to promise not to use write tools.  

### Correct Answer: **B**

**English Explanation:**
Least privilege reduces the blast radius of model mistakes or malicious retrieved content. An investigative agent should not possess unrelated financial-write capabilities.

### 中文解说

这是：

**Least Privilege**

CounterpartyAgent 的任务是：

> investigate

不是：

> execute payment

因此：

```text
get_counterparty   ✓
search_payments    ✓

submit_payment     ✗
cancel_payment     ✗
```

C 的 Tool Description 可以减少误调用，但不能阻止调用。

D 更只是 behavioral instruction。

**Key exam takeaway:**

> **Do not expose capabilities merely because an agent might theoretically use them.**

---

## Question 3 — Retry Classification

`get_counterparty` returns:

```json
{
  "error_type": "RATE_LIMIT",
  "retryable": true,
  "retry_after_seconds": 20
}
```

What should the system MOST reasonably do?

**A.** Retry immediately in a tight loop.  
**B.** Respect the retry guidance and use a bounded retry strategy.  
**C.** Permanently abandon the investigation.  
**D.** Change the counterparty ID before retrying.  

### Correct Answer: **B**

**English Explanation:**
A rate-limit response indicates a transient condition. The system should respect retry timing while keeping recovery bounded.

### 中文解说

这里信息非常明确：

```text
RATE_LIMIT
retryable = true
retry_after = 20
```

正确 recovery：

```text
Wait appropriately
      ↓
Retry
      ↓
Bounded attempts
```

A 会加剧 rate limit。

C 太激进。

D 是把：

**transient service condition**

误当成：

**invalid input**

注意考试里的 Error 分类：

```text
TIMEOUT / RATE_LIMIT
→ retry candidate

INVALID_ARGUMENT
→ correct input

ACCESS_DENIED
→ authorization

NOT_FOUND
→ verify identity/resource
```

**Key exam takeaway:**

> **Error semantics determine recovery strategy.**

---

## Question 4 — State-Changing Retry

### Choose TWO.

Claude calls:

```text
submit_payment(payment_id="P-4831")
```

The request reaches the payment service, but the network connection drops before a response is received.

Which TWO controls are MOST important before retrying?

**A.** Use a stable idempotency key for the logical payment operation.  
**B.** Reconcile external payment state to determine whether the original operation succeeded.  
**C.** Assume a missing response means the payment failed.  
**D.** Increase Claude's confidence threshold.  

### Correct Answers: **A and B**

**English Explanation:**
The outcome of a state-changing request is ambiguous when the response is lost. Idempotency and state reconciliation prevent a blind retry from creating duplicate financial effects.

### 中文解说

这是生产系统极高频陷阱：

```text
Client
  │
  │ submit
  ↓
Server
  │
  ├── Payment executed ✓
  │
  X response lost
```

Client 只看到：

```text
ERROR / TIMEOUT
```

但现实世界可能已经：

> 钱付出去了。

所以必须考虑：

### A — Idempotency

```text
payment_request_id = XYZ001
```

第二次调用仍代表：

> 同一笔业务操作。

### B — State Reconciliation

查询：

> P-4831 到底已经 submitted 了吗？

C 是危险假设：

> No response = no execution

错误。

**Key exam takeaway:**

> **Transport failure does not determine business-operation outcome.**

---

## Question 5 — Policy Extraction and Enforcement

Treasury policy states:

> Payments of $1 million or more to a newly created counterparty require human approval.

Claude has already extracted:

```json
{
  "amount": 1400000,
  "counterparty_status": "NEW"
}
```

Both fields have been validated.

What should determine whether approval is required?

**A.** Ask Claude to interpret the policy again.  
**B.** Implement the threshold/status rule deterministically over the validated fields.  
**C.** Ask three agents to vote.  
**D.** Use Claude's confidence score.  

### Correct Answer: **B**

**English Explanation:**
Once the required structured facts are available and the policy condition is exact, deterministic code is the strongest mechanism for applying the rule.

### 中文解说

这里要明确分工：

Claude 擅长：

```text
Unstructured document
        ↓
Extract:
amount = 1.4M
status = NEW
```

但是接下来：

```text
amount >= 1M
AND
status == NEW
```

已经是确定性 Boolean 判断。

所以用 Code：

```text
if amount >= 1_000_000 and status == "NEW":
    require_human_approval()
```

A 会把一个已经明确的问题重新变成概率判断。

C 是 **over-agentization**。

D 再次强调：

> **Confidence ≠ Authorization requirement**

**Key exam takeaway:**

> **LLM for interpretation; deterministic logic for exact policy enforcement.**

---

## Question 6 — Context Pollution

After four hours, `CoordinatorAgent` contains:

* 90 raw tool responses;
* 30 abandoned hypotheses;
* 15 repeated payment summaries;
* 8 verified findings;
* 3 unresolved questions.

The coordinator begins reconsidering hypotheses that were already disproven.

What is the BEST response?

**A.** Keep everything active because deletion always reduces reasoning quality.  
**B.** Maintain verified findings and unresolved questions in compact structured state while pruning or externalizing obsolete working context.  
**C.** Restart the entire case from scratch.  
**D.** Repeat all verified findings after every message.  

### Correct Answer: **B**

**English Explanation:**
Long-running agents should preserve durable knowledge while reducing obsolete or redundant active context. This improves information salience without discarding retrievable evidence.

### 中文解说

长期 Agent 不应该把 memory 理解成：

> 无限增长的 Chat History。

更合理：

```text
ACTIVE STATE
────────────
Verified findings
Unresolved questions
Current objective
Important references

EXTERNAL / RETRIEVABLE
────────────
Raw tool results
Old logs
Supporting documents

PRUNE
────────────
Duplicate summaries
Disproven hypotheses
```

关键词：

**Salience**

真正重要的信息应该更容易被模型“看到”。

**Key exam takeaway:**

> **Preserve knowledge state; do not preserve every reasoning artifact equally.**

---

## Question 7 — Conflicting Evidence

Two sources report:

```text
Bank confirmation:
Payment amount = $950,000

Internal ledger:
Payment amount = $1,050,000
```

The difference determines whether human approval was required.

What should the system do?

**A.** Average the values to $1,000,000.  
**B.** Use whichever source was retrieved first.  
**C.** Preserve both claims with provenance, apply explicit source-authority/reconciliation rules, and escalate unresolved material conflict when necessary.  
**D.** Ask Claude which number looks more plausible.  

### Correct Answer: **C**

**English Explanation:**
Material conflicts should remain traceable to their sources. Explicit reconciliation rules may resolve them; otherwise the uncertainty should remain visible and may require escalation.

### 中文解说

为什么 A 特别危险？

因为：

```text
950K + 1.05M
────────────
average = 1M
```

刚好落在 approval threshold。

但没有任何证据说明：

> 真值就是平均数。

这叫人为制造 **false precision**。

正确：

```text
Claim A + provenance
Claim B + provenance
       ↓
Conflict detected
       ↓
Authority/reconciliation rule
       ↓
Resolved?
├─ Yes → continue
└─ No  → surface / escalate
```

**Key exam takeaway:**

> **Never manufacture certainty by averaging conflicting evidence.**

---

## Question 8 — Independent Review

`RiskAgent` concludes:

> Payment P-4831 is low risk.

Helios wants stronger verification for high-value payments.

Which approach is BEST?

**A.** Ask `RiskAgent`, “Are you sure?” several times.  
**B.** Give an independent reviewer the necessary evidence and evaluation criteria and have it produce its own assessment before comparing results.  
**C.** Accept the answer whenever confidence exceeds 90%.  
**D.** Make the original explanation longer.  

### Correct Answer: **B**

**English Explanation:**
Independent evidence-based review reduces anchoring compared with repeatedly asking the original reasoning process to confirm itself.

### 中文解说

区别：

### Self-confirmation

```text
I think LOW.
↓
Am I sure?
↓
Yes.
```

容易产生：

**Anchoring**

### Independent review

```text
Evidence + Criteria
       ↓
Reviewer B
       ↓
Independent conclusion
```

然后再比较：

```text
Agent A: LOW
Agent B: HIGH
→ investigate disagreement
```

注意 reviewer 不是：

> 什么信息都不给。

它仍然需要：

**necessary evidence + criteria**

只是尽量避免被第一轮 conclusion 和 reasoning 锚定。

**Key exam takeaway:**

> **Independent evaluation is stronger than repeated self-confirmation.**

---

## Question 9 — Evaluation Design

### Choose TWO.

Helios tests a new risk-analysis prompt.

Results are:

```text
Overall accuracy:      90% → 94%
Routine payments:      93% → 97%
High-value payments:   91% → 78%
```

High-value payments have substantially greater financial consequences.

Which TWO conclusions are MOST appropriate?

**A.** The aggregate improvement is insufficient evidence for deployment because a critical slice regressed materially.  
**B.** Evaluation should weight or separately track operationally important failure classes.  
**C.** Deploy immediately because 94% is greater than 90%.  
**D.** Ignore high-value cases because they are less frequent.  

### Correct Answers: **A and B**

**English Explanation:**
Aggregate metrics can hide severe regressions in high-impact slices. Evaluation should reflect operational consequences, not merely example frequency.

### 中文解说

这是：

**Segmented Evaluation**

整体：

```text
90 → 94 ✓
```

但是关键 Case：

```text
High-value
91 → 78 ✗✗
```

不能只看 average。

特别要单独看：

* safety-critical；
* high-value；
* rare-but-severe；
* human-escalation cases；
* irreversible-action cases。

这也涉及：

**Cost-sensitive evaluation**

一个 $10 的错误和一个 $10M 的错误，不一定应该在 evaluation 里“价值相同”。

**Key exam takeaway:**

> **Evaluation metrics should reflect consequence, not only frequency.**

---

## Question 10 — Failure-Layer Diagnosis

Helios observes:

> Claude correctly determines that a payment requires human approval.
> Structured output contains `"requires_approval": true`.
> The application correctly reads the Boolean.
> A human approves the payment.
> Before execution, the payment amount changes substantially.
> The application executes using the old approval without revalidation.

Which layer MOST directly failed?

**A.** Prompt engineering  
**B.** Tool description  
**C.** Authorization-state freshness / execution-time revalidation  
**D.** Context compression  

### Correct Answer: **C**

**English Explanation:**
The approval was valid for an earlier state, but the protected action changed before execution. Authorization must be revalidated when material action parameters change.

### 中文解说

这是：

**TOCTOU — Time of Check to Time of Use**

过程：

```text
Amount = $1.4M
      ↓
Human approval ✓
      ↓
Amount changes to $3.2M
      ↓
Old approval reused ✗
```

问题不是 Claude。

问题是：

> Approval 和最终执行的 action state 不一致。

更安全：

```text
Proposed action
      ↓
Approval
      ↓
Action changed?
├─ No  → execute
└─ Yes → revalidate / reapprove
```

最好让 approval 与：

```text
action
target
material parameters
version/state
```

关联，而不只是：

```text
case_id
```

---

# Day 16 — High-Value Decision Map

| Scenario signal                                  | Think first                          |
| ------------------------------------------------ | ------------------------------------ |
| Specialist repeats coordinator's searches        | **Handoff/context packaging**        |
| Investigative agent has payment tools            | **Least privilege**                  |
| `RATE_LIMIT`                                     | **Wait + bounded retry**             |
| Write request loses response                     | **Idempotency + reconciliation**     |
| Exact threshold over validated data              | **Deterministic enforcement**        |
| Long session revisits disproven ideas            | **Context compression/state**        |
| Sources materially disagree                      | **Provenance + reconciliation**      |
| Agent repeatedly confirms itself                 | **Independent review**               |
| Overall eval improves but critical class worsens | **Segmented evaluation**             |
| Action changes after approval                    | **Authorization freshness / TOCTOU** |

## Four Error Classes Worth Memorizing

```text
TRANSIENT
TIMEOUT / RATE_LIMIT
→ bounded retry

INPUT
INVALID_ARGUMENT
→ correct input

AUTHORIZATION
ACCESS_DENIED
→ obtain permission / reject / escalate

AMBIGUOUS SIDE EFFECT
TIMEOUT after write request
→ reconcile state / idempotency
```

The last category is especially important.

A timeout from:

```text
search_payments
```

and a timeout from:

```text
submit_payment
```

should **not automatically receive the same recovery strategy**.

The first is generally a read retry problem.

The second may be a **duplicate financial-action problem**.

### Eight sentences to memorize

> **Agent handoffs should transfer task-relevant state.**

> **Least privilege limits the consequences of model mistakes.**

> **Error semantics determine recovery strategy.**

> **Transport failure does not prove business failure.**

> **Exact policies should be enforced deterministically.**

> **Preserve verified state while pruning obsolete reasoning artifacts.**

> **Independent review is stronger than repeated self-confirmation.**

> **Authorization must remain valid for the exact action that executes.**
