# CCAR-F Daily Practice — Day 6

Today moves into **mixed-domain scenarios**. The main challenge is identifying **which architectural layer is actually failing**: orchestration, tool design, prompting, structured output, context management, or programmatic enforcement.

**Exam mode:** 8 questions. Target time: **16 minutes**. Some questions are **Choose TWO**.

---

## Scenario: NovaBank Lending Assistant

**NovaBank is deploying a Claude-based lending assistant for commercial loan applications.**

The system uses a coordinator agent and several specialist agents:

* `DocumentAgent` extracts financial data from uploaded documents.
* `CreditAgent` evaluates credit metrics.
* `PolicyAgent` retrieves lending policies.
* `ResearchAgent` investigates external company information.
* `CoordinatorAgent` combines the results and prepares a recommendation.

Available tools include:

```text
get_customer_profile
search_transactions
retrieve_policy
search_company
calculate_financial_ratios
submit_loan_decision
```

The production system must be auditable, reliable, and compliant with lending policies.

---

## Question 1 — Agent vs Workflow

Every loan application must perform these steps in this exact order:

```text
Identity verification
→ Sanctions screening
→ Required-document validation
```

Only after all three succeed may the AI-driven credit investigation begin.

What is the BEST architecture?

**A.** Let Claude dynamically decide whether to perform each compliance step.
**B.** Implement the mandatory compliance sequence as a deterministic workflow, then start the agentic investigation.
**C.** Ask three independent agents to perform the steps in any order.
**D.** Put the required sequence in the system prompt and allow Claude to enforce it.

### Correct Answer: **B**

**English Explanation:**
Mandatory, predetermined compliance steps are better implemented as a deterministic workflow. Agentic reasoning should begin where the next action genuinely requires dynamic judgment.

### 中文解说

这题不是问：

> Workflow 好，还是 Agent 好？

而是在考：

> **应该在哪里使用 Workflow，在哪里使用 Agent？**

前三步：

```text
Identity
→ Sanctions
→ Documents
```

顺序固定，而且属于 compliance。

所以：

**Deterministic Workflow**

完成以后，真正的贷款调查：

> 下一步查什么取决于前面发现的信息。

这部分才适合：

**Agentic Loop**

A、D 的共同问题是把必须保证的流程交给模型自行遵守。

C 又忽略了顺序要求。

**Exam takeaway:**

> **Deterministic shell + agentic core is often stronger than making everything agentic.**

---

## Question 2 — Tool Boundary

Claude frequently confuses these two tools:

```text
search_transactions
Description: Search financial information.

search_company
Description: Search financial information.
```

Which TWO changes are MOST likely to improve tool selection?

**A.** Give each tool a precise description including when to use it and when not to use it.
**B.** Clearly distinguish the input parameters and semantic boundaries of the two tools.
**C.** Increase temperature so Claude explores both tools more often.
**D.** Add “Always choose the correct financial tool” to the system prompt.

### Correct Answers: **A and B**

**English Explanation:**
Tool selection depends heavily on clear tool semantics. Distinct descriptions, use cases, non-use cases, and parameter boundaries help Claude distinguish overlapping capabilities.

### 中文解说

现在两个工具的 description 完全一样：

```text
Search financial information.
```

Claude 当然很难判断。

应该改成类似：

```text
search_transactions

Search the current customer's historical bank transactions.

Use for:
- deposits
- withdrawals
- transaction history

Do not use for:
- public company research
```

另一个：

```text
search_company

Search external information about a company.

Use for:
- company background
- public company information

Do not use for:
- customer's bank transactions
```

A 解决 description。

B 解决 **tool boundary**。

C 的 temperature 不是问题所在。

D 虽然是 instruction，但：

> “Choose correctly”

没有提供判断“正确”的依据。

**Exam takeaway:**

> **Tool confusion → clarify semantics before adding more prompting.**

---

## Question 3 — Structured Extraction

`DocumentAgent` must extract:

```text
annual_revenue
operating_income
total_debt
employee_count
```

Some documents do not report `employee_count`.

The current schema requires every field to contain a numeric value. Claude occasionally invents an employee count.

What is the BEST solution?

**A.** Require `0` when employee count is absent.
**B.** Allow `employee_count` to be null and distinguish missing information from a real numeric value.
**C.** Ask Claude to estimate employee count from annual revenue.
**D.** Remove structured output and return prose instead.

### Correct Answer: **B**

**English Explanation:**
The schema should represent legitimate absence or uncertainty. Using `null` prevents the model from being forced to invent a value or misuse a meaningful numeric value such as zero.

### 中文解说

这里特别注意：

```text
0 ≠ unknown
```

如果公司实际有：

```text
employee_count = 0
```

这是一个真正的数据。

但：

```text
employee_count = null
```

表示：

> Source document 没有提供。

所以 A 会混淆：

**zero**

和：

**unknown**

C 则直接制造 hallucination。

D 没有必要因为一个字段可能缺失，就放弃 Structured Output。

**Exam takeaway:**

> **Represent uncertainty explicitly instead of forcing fabricated precision.**

---

## Question 4 — Programmatic Enforcement

NovaBank policy states:

> Loans above $10 million require approval from two authorized human reviewers before `submit_loan_decision` may approve the loan.

Which design provides the STRONGEST guarantee?

**A.** Put the rule at the top of the system prompt in capital letters.
**B.** Add several examples showing that large loans require approval.
**C.** Make `submit_loan_decision` reject approval unless two valid reviewer authorizations are present.
**D.** Ask the coordinator to check its confidence before calling the tool.

### Correct Answer: **C**

**English Explanation:**
A mandatory authorization rule should be enforced at the application/tool boundary. Model instructions can guide behavior but should not be the sole protection for a high-risk state-changing action.

### 中文解说

看到：

> **require**
> **$10 million**
> **authorized human reviewers**
> **approve**

应该立即想到：

**Hard security/business constraint**

所以：

```text
Claude
   ↓
submit_loan_decision
   ↓
Authorization validation
   ↓
2 valid approvals?
  /            \
Yes             No
 ↓               ↓
Execute          Reject
```

A 和 B 可以作为辅助措施，但不是 guarantee。

D 的 confidence 与 authorization 完全是两回事。

这是 CCAR-F 最常见的跨 Domain 陷阱之一：

> **Prompting technique 本身是正确技术，但被放到了错误的控制层。**

**Exam takeaway:**

> **High-risk authorization belongs at the enforcement boundary.**

---

## Question 5 — Context Management

After several hours, `CoordinatorAgent` has accumulated:

* hundreds of pages of raw tool output;
* repeated company descriptions;
* multiple intermediate analyses;
* detailed logs from failed searches.

It begins contradicting verified financial facts discovered earlier.

What is the BEST improvement?

**A.** Continue appending everything because more context always improves reasoning.
**B.** Preserve verified facts in compact structured state and trim or summarize low-value context.
**C.** Restart the investigation every hour and discard all previous findings.
**D.** Repeat every important fact ten times near the end of the context.

### Correct Answer: **B**

**English Explanation:**
Long-running agents benefit from separating durable verified facts from transient conversational and tool output. Compact structured state plus context pruning improves information salience and reduces context pollution.

### 中文解说

这里同时需要两个动作：

**1. Preserve important information**

```text
Verified facts
→ structured state
```

**2. Reduce low-value information**

```text
raw logs
duplicate output
obsolete intermediate reasoning
→ trim / summarize
```

注意不是单纯：

> “做 summary 就行。”

真正重要的事实最好保存成结构化状态，例如：

```json
{
  "revenue": 4200000000,
  "debt": 1100000000,
  "policy_version": "2026-07",
  "sanctions_check": "passed"
}
```

A 是经典错误：

> More context ≠ better context.

C 又会丢失已经验证的信息。

D 会进一步消耗 context。

**Exam takeaway:**

> **Preserve signal; remove noise.**

---

## Question 6 — Provenance and Conflict

`ResearchAgent` obtains:

```text
Company filing:
Debt = $1.10B

Third-party database:
Debt = $1.35B
```

The final recommendation depends materially on the debt level.

What should the system do?

**A.** Ask Claude to choose whichever number seems more believable.
**B.** Average the values to obtain $1.225B.
**C.** Preserve both claims and their provenance, apply defined source-authority rules, and surface the conflict if it remains unresolved.
**D.** Use the larger value because conservative lending is safer.

### Correct Answer: **C**

**English Explanation:**
Material evidence conflicts should remain traceable to their sources. Explicit source-authority rules may resolve the disagreement; otherwise the uncertainty should be surfaced rather than silently hidden.

### 中文解说

这里有两个关键词：

**Provenance**

和：

**Conflict handling**

应该保存：

```text
Claim A
Debt = $1.10B
Source = Company filing

Claim B
Debt = $1.35B
Source = Third-party database
```

然后：

```text
Source authority policy
        ↓
Can resolve?
 /              \
Yes               No
 ↓                 ↓
Use authoritative  Report conflict
value              / escalate if required
```

B 的“平均”非常危险。

两个 source disagreement 并不意味着真值在中间。

D 虽然“保守”，但这不是证据处理原则。

**Exam takeaway:**

> **Do not silently collapse conflicting evidence.**

---

## Question 7 — Validation Loop

`DocumentAgent` extracts:

```text
Revenue:          $10M
Operating costs:   $8M
Operating income:  $5M
```

The output passes JSON Schema validation because all fields are valid numbers.

A business-rule validator detects an inconsistency.

What should happen NEXT?

**A.** Accept the result because schema validation passed.
**B.** Retry the identical original prompt without mentioning the problem.
**C.** Return the specific inconsistency to the extraction process, request correction, and validate again.
**D.** Remove the business-rule validator.

### Correct Answer: **C**

**English Explanation:**
Structural validity does not imply semantic correctness. Specific validation feedback provides new information that can guide correction before the result is revalidated.

### 中文解说

先区分两个层：

```text
Schema Validation
→ 类型/结构正确吗？

Semantic Validation
→ 数据关系合理吗？
```

这里：

```text
10 - 8 ≠ 5
```

所以虽然：

```text
JSON = valid
numbers = valid
```

业务上仍然错。

正确 loop：

```text
Generate
   ↓
Schema validation
   ↓
Semantic validation
   ↓
FAIL
   ↓
Specific feedback
   ↓
Correct
   ↓
Validate again
```

B 的问题是 **blind retry**。

模型没有获得新信息，很可能再次产生同样结果。

**Exam takeaway:**

> **Schema-valid ≠ semantically valid.**

---

## Question 8 — Escalation

**Choose TWO.**

Which TWO situations provide the STRONGEST reasons to escalate a NovaBank case to a human reviewer?

**A.** A temporary API timeout occurs once and the tool reports that the error is retryable.
**B.** Two authoritative sources remain materially inconsistent after the defined resolution procedure is exhausted.
**C.** Claude reports 84% confidence while an arbitrary team target is 85%.
**D.** Policy explicitly requires human approval for the requested high-risk decision.

### Correct Answers: **B and D**

**English Explanation:**
Human escalation is appropriate when material ambiguity cannot be resolved automatically or when policy explicitly requires human authorization. A transient retryable error should normally be recovered locally, and an arbitrary self-reported confidence threshold alone is a weak escalation criterion.

### 中文解说

这题考：

> **什么时候才真的需要 Human？**

B：

**重要信息冲突，而且自动 resolution 已经用尽**

→ 合理 escalation。

D：

**Policy 明确规定人工审批**

→ 必须 escalation / approval。

A：

```text
timeout
retryable = true
```

优先：

**local bounded retry**

而不是立即找人。

C 是非常重要的陷阱：

```text
confidence = 84%
threshold = 85%
```

看起来非常精确，但模型 confidence 如果没有经过 calibration，本身不能直接解释成实际正确概率。

**Exam takeaway:**

> **Escalate for policy, risk, or unresolved material ambiguity—not merely because the model feels uncertain.**

---

# Day 6 — Mixed-Domain Decision Map

从今天开始，不要看到某个关键词就机械选择答案。先判断：

**“问题到底发生在哪一层？”**

| Failure                                    | Architectural layer | Think first                        |
| ------------------------------------------ | ------------------- | ---------------------------------- |
| Mandatory fixed sequence                   | Orchestration       | **Workflow**                       |
| Dynamic next action                        | Orchestration       | **Agentic loop**                   |
| Claude selects wrong tool                  | Tool design         | **Description / boundary**         |
| Must use a specific tool                   | Tool control        | **Force specific tool**            |
| Must prevent unauthorized action           | Enforcement         | **Code / authorization check**     |
| Output shape changes                       | Structured output   | **Schema**                         |
| Fields contradict each other               | Reliability         | **Semantic validation**            |
| Important facts disappear in long sessions | Context             | **Structured state + pruning**     |
| Sources disagree                           | Reliability         | **Provenance + conflict handling** |
| Automated recovery is exhausted            | Reliability         | **Escalation**                     |

## The Most Important CCAR-F Trap

Imagine these two questions:

> **Claude keeps selecting the wrong refund tool.**

Think:

**Tool description / boundary**

But:

> **How do you guarantee an unauthorized refund can never execute?**

Think:

**Programmatic enforcement**

Both involve the same `refund` tool, but the correct architectural layer is completely different.

That leads to today's most useful exam rule:

> **Do not choose the most sophisticated-sounding answer. Choose the answer that fixes the failing layer.**

### Six sentences to memorize today

> **Fixed mandatory steps belong in deterministic workflows.**

> **Dynamic decisions belong in agentic loops.**

> **Descriptions guide tool selection; code enforces authorization.**

> **Schemas enforce structure; validators enforce business correctness.**

> **Structured state preserves important facts across long investigations.**

> **Provenance preserves the relationship between claims and evidence.**
