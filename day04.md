# CCAR-F Daily Practice — Day 4

Today’s primary focus is **Domain 4: Prompt Engineering & Structured Output**, with cross-domain questions on validation, batch processing, and independent review.

**Exam mode:** 8 questions. Target time: **16 minutes**. Several distractors are intentionally “partly correct”; choose the **BEST architectural answer**.

---

## Scenario: Meridian Document Intelligence

**Meridian Insurance is building a Claude-based document-processing platform.**

The platform processes insurance claims, invoices, medical receipts, and policy documents. Claude extracts structured fields, classifies documents, identifies inconsistencies, and produces summaries for downstream systems.

The engineering team has observed several problems:

* extraction formats are inconsistent;
* Claude sometimes invents values for missing fields;
* classification of unusual documents is unreliable;
* validation failures are repeatedly retried without improvement;
* millions of historical documents must be processed economically;
* high-risk claim decisions require stronger review.

---

## Question 1 — Explicit Evaluation Criteria

Meridian asks Claude:

> “Review this claim and report serious problems.”

Different runs identify very different issues. Some responses report formatting mistakes while others ignore potentially fraudulent inconsistencies.

What is the **BEST first improvement**?

**A.** Increase the context window.  
**B.** Define explicit criteria for what counts as a serious problem and what should be ignored.  
**C.** Increase temperature to encourage broader analysis.  
**D.** Ask Claude to make the response longer.  

### Correct Answer: **B**

**English Explanation:**
Vague qualitative instructions produce inconsistent interpretations. Explicit criteria define the decision boundary and help Claude distinguish relevant findings from irrelevant ones.

A stronger instruction might specify:

> Report evidence of fraud, contradictory claim amounts, impossible dates, or missing mandatory authorization. Do not report formatting preferences or stylistic issues.

### 中文解说

题眼是：

> **serious problems**

“serious”到底是什么意思，没有定义。

所以不同运行结果会出现不同判断。

最好的第一步不是让 Claude：

* 想更多；
* 写更多；
* 获得更多 context；

而是把 **evaluation criteria** 定义清楚。

例如：

```text
Report:
- suspected fraud
- contradictory monetary values
- impossible dates
- missing required authorization

Do not report:
- formatting preferences
- wording style
- cosmetic issues
```

A 的 context window 并没有解决判断标准模糊的问题。

C 增加 temperature 反而可能让结果更不稳定。

D 的“写得更长”也不等于“判断更准确”。

**Exam takeaway:**

> **Inconsistent judgment caused by vague requirements → make the criteria explicit.**

---

## Question 2 — Few-Shot Examples

Meridian already provides detailed classification rules for:

* invoice,
* receipt,
* claim form,
* supporting evidence.

However, Claude still inconsistently classifies borderline documents that contain characteristics of two categories.

What is the MOST useful next improvement?

**A.** Repeat the same rules three times.  
**B.** Add representative few-shot examples, especially examples near category boundaries.  
**C.** Remove category definitions and let Claude infer them.  
**D.** Require longer chain-of-thought output.  

### Correct Answer: **B**

**English Explanation:**
When instructions are already explicit but behavior remains inconsistent around nuanced boundaries, well-chosen examples can demonstrate the intended decision pattern. Boundary examples are particularly valuable.

### 中文解说

这题要区分：

**规则不清楚**

和：

**规则已经写清楚，但边界案例仍不稳定**

如果题目是第一种，优先：

**Explicit instructions**

但这里明确说：

> already provides detailed classification rules

所以继续堆 instructions 的边际收益已经很低。

此时：

**Few-shot examples**

特别适合展示：

> “遇到这种很像 Invoice、又有 Receipt 特征的文件，我们到底希望怎么分类？”

A 只是重复，没有提供新的信息。

C 会让分类边界更模糊。

D 是常见干扰项：题目并不需要更长的 reasoning 输出，而是需要更稳定的行为示范。

**Exam takeaway:**

> **Clear instructions + inconsistent edge cases → Few-shot examples.**

---

## Question 3 — Structured Output

A downstream payment system requires this exact structure:

```json
{
  "invoice_id": "string",
  "supplier": "string",
  "amount": 0.0,
  "currency": "string"
}
```

Claude occasionally returns:

```text
Here is the extracted invoice:

Invoice ID: INV-100
Supplier: Atlas Ltd.
Amount: $500
```

What is the BEST architecture?

**A.** Add “Please use JSON” to the prompt.  
**B.** Define a structured schema/tool interface for the required fields and validate the returned structure.  
**C.** Parse any natural-language response using regular expressions.  
**D.** Ask Claude to produce three responses and select the shortest one.  

### Correct Answer: **B**

**English Explanation:**
When software depends on a predictable machine-readable contract, the output structure should be constrained explicitly and validated rather than relying only on natural-language formatting instructions.

### 中文解说

看到：

> **downstream payment system**

就应该立刻想到：

**这是机器消费，不是人消费。**

所以：

```text
Prompt
  ↓
Schema-constrained output
  ↓
Validation
  ↓
Payment system
```

比：

> “Please use JSON.”

可靠得多。

A 可以提高格式遵守率，但不能提供同等级别的结构保证。

C 用正则去猜自由文本结构，非常脆弱。

D 和结构正确性没有直接关系。

这里再次出现 CCAR-F 很重要的思维：

> **Prompt guidance ≠ output contract**

**Exam takeaway:**

> **Machine-readable contract → Schema + validation.**

---

## Question 4 — Missing Values

An invoice schema currently requires:

```text
purchase_order_number: string
```

Some legitimate invoices do not contain a purchase-order number.

Claude has started generating plausible-looking purchase-order numbers when none appears in the source document.

What is the BEST solution?

**A.** Require Claude to always produce a string because downstream systems dislike missing fields.  
**B.** Allow the field to be nullable and explicitly instruct Claude not to infer unsupported values.  
**C.** Increase temperature so generated purchase-order numbers vary more naturally.  
**D.** Remove the field from every invoice.  

### Correct Answer: **B**

**English Explanation:**
The schema should represent legitimate uncertainty or absence. If a value may genuinely be missing, allowing `null` prevents the model from being forced into inventing a value merely to satisfy the schema.

### 中文解说

这是很容易出现在实际考试里的 hallucination 场景。

现在 schema 实际上在告诉 Claude：

> 这个字段必须存在，而且必须是 string。

但是源文件里没有。

模型面临冲突：

```text
Source: no value
Schema: value required
```

于是可能“补”一个看起来合理的值。

正确做法：

```text
purchase_order_number:
    string | null
```

同时明确：

> Do not infer a value that is not supported by the source.

A 正是在制造 hallucination 压力。

C 更明显错误。

D 又过头了，因为有些 Invoice 确实有 PO number。

**Exam takeaway:**

> **Unknown or absent data should be representable as unknown or absent.**

尤其记住：

> **Unknown ≠ Guess**

---

## Question 5 — Validation and Retry

Claude extracts:

```text
subtotal = 100
tax = 10
total = 150
```

Meridian's validator detects:

```text
subtotal + tax != total
```

The current application simply calls Claude again with the original prompt. Claude frequently returns the same incorrect values.

What should the application do?

**A.** Retry repeatedly with the identical input until the answer changes.  
**B.** Return the specific validation failure to Claude and request a corrected extraction.  
**C.** Accept the result because all fields are valid numbers.  
**D.** Remove arithmetic validation.  

### Correct Answer: **B**

**English Explanation:**
A retry is more effective when the model receives actionable feedback about what failed. Specific validation feedback enables targeted self-correction instead of simply repeating the same attempt.

### 中文解说

这是今天最值得背的一题之一。

现在系统做的是：

```text
Wrong answer
    ↓
Retry same prompt
    ↓
Wrong answer
    ↓
Retry same prompt
```

Claude 没有获得任何新信息。

更好的流程：

```text
Extraction
    ↓
Validation
    ↓
FAIL
    ↓
"subtotal + tax does not equal total"
    ↓
Claude correction
    ↓
Validate again
```

A 可以叫：

**blind retry**

而 B 是：

**feedback-driven retry**

C 只做了 syntax/type validation，却忽略了 **semantic validation**。

三个字段都是 number，并不代表它们在业务上互相一致。

D 更不是解决方案。

**Exam takeaway:**

> **Retry + specific failure feedback > blind retry.**

---

## Question 6 — Syntactic vs Semantic Validation

Claude returns:

```json
{
  "claim_date": "2026-08-30",
  "incident_date": "2026-09-15"
}
```

The JSON is syntactically valid and matches the schema, but company policy requires:

> The incident must occur on or before the claim date.

What additional control is needed?

**A.** No additional control; schema validation is sufficient.  
**B.** Semantic/business-rule validation after structural validation.  
**C.** A larger context window.  
**D.** Higher temperature.  

### Correct Answer: **B**

**English Explanation:**
Schema validation can verify structure and basic types, but business relationships between fields often require semantic validation.

### 中文解说

这题非常关键，因为很多人会把：

**Structured Output**

理解成：

> 有 Schema 就全部解决了。

不是。

这里：

```text
claim_date    = valid date
incident_date = valid date
```

每一个字段单独看都符合 Schema。

但是两者关系：

```text
incident_date <= claim_date
```

不满足业务规则。

因此验证至少可以分成：

```text
1. Structural / syntactic validation
          ↓
2. Semantic / business validation
```

Schema 负责：

* 字段有没有；
* 类型对不对；
* 基本格式对不对。

Business validator 负责：

* 金额之间是否合理；
* 日期关系是否合理；
* 状态组合是否合法；
* 业务约束是否满足。

**Exam takeaway:**

> **Schema-valid does not necessarily mean business-valid.**

---

## Question 7 — Batch Processing

Meridian must process **4 million historical documents**.

Requirements:

* processing can complete overnight or over several hours;
* users are not waiting interactively;
* reducing processing cost is important.

Which approach is MOST appropriate?

**A.** Process every document synchronously through an interactive request.  
**B.** Use asynchronous batch processing designed for high-volume, latency-tolerant workloads.  
**C.** Create one four-million-document prompt.  
**D.** Launch four million autonomous agents simultaneously.  

### Correct Answer: **B**

**English Explanation:**
Batch processing is appropriate for large volumes of independent work when immediate response latency is not required. It separates high-throughput offline processing from latency-sensitive interactive workloads.

### 中文解说

题眼有三个：

> **4 million**

> **overnight**

> **users are not waiting**

这就是非常标准的：

**Batch workload**

判断方法：

```text
Does the user need the answer now?
        │
   ┌────┴────┐
  Yes        No
   ↓          ↓
Sync       Large volume?
              │
             Yes
              ↓
            Batch
```

A 的 synchronous 方式更适合在线用户等待的请求。

C 受到 context、可靠性、故障恢复等很多限制。

D 是典型的“过度 agent 化”答案。

**Exam takeaway:**

> **Latency-sensitive → synchronous**
> **High-volume + latency-tolerant → batch**

---

## Question 8 — Independent Review

Claude produces a recommendation to approve a high-value insurance claim.

Because the financial impact is significant, Meridian wants a second AI review that is less likely to simply reinforce the assumptions of the first analysis.

Which design is BEST?

**A.** Ask the same generation context, “Are you sure?”  
**B.** Repeat the original recommendation three times and use majority voting.  
**C.** Perform an independent review with fresh context containing the evidence, decision criteria, and first-pass output as needed for critique.  
**D.** Increase the first model's confidence threshold from 80% to 95%.  

### Correct Answer: **C**

**English Explanation:**
An independent review should minimize anchoring on the original reasoning while still providing the evidence and criteria needed to evaluate the result. A separate review pass can identify errors that self-confirming continuation may overlook.

### 中文解说

题目关键词：

> **less likely to simply reinforce the assumptions**

这在问：

**如何减少自我确认 / anchoring？**

如果刚刚 Claude 已经花很多 token 得出：

> APPROVE

然后马上问：

> Are you sure?

它仍然处于同一套 reasoning/context 里，很容易继续证明自己是对的。

更强的设计：

```text
Pass 1
Evidence
  ↓
Analysis
  ↓
Recommendation

             Fresh review context
                    ↓
            Evidence + Criteria
                    ↓
             Independent Review
                    ↓
            Compare / adjudicate
```

A 是 **self-review**，有价值，但独立性较弱。

B 只是重复同一过程，不一定获得真正独立判断。

D 更是一个典型陷阱：

> 模型自己说 95% confidence，并不会自动让结果变得更正确。

**Exam takeaway:**

> **High-stakes output → independent verification can be stronger than self-confirmation.**

---

# Day 4 — Quick Memory Sheet

今天这 8 组关系建议直接记成英文，因为考试时看到关键词会更快：

| Scenario signal                       | Think first                          |
| ------------------------------------- | ------------------------------------ |
| Vague judgment                        | **Explicit criteria**                |
| Clear rules but unstable edge cases   | **Few-shot examples**                |
| Machine-consumed response             | **Schema / structured output**       |
| Legitimately missing field            | **Nullable / represent uncertainty** |
| Validation failure                    | **Specific feedback + retry**        |
| Schema passes but business rule fails | **Semantic validation**              |
| Huge offline workload                 | **Batch processing**                 |
| High-stakes generated result          | **Independent review**               |

## Common Trap #1

> **“Make the prompt longer.”**

More instructions are not automatically better.

First identify the failure:

```text
Ambiguous requirement
→ explicit criteria

Boundary behavior
→ examples

Output structure
→ schema

Business correctness
→ validation
```

## Common Trap #2

> **“The JSON is valid, therefore the answer is valid.”**

Wrong.

Always separate:

```text
Syntactic validity
        ≠
Semantic correctness
```

## Common Trap #3

> **“Retry”**

Ask:

> **What new information will the model receive on the retry?**

If the answer is:

> none

then it may simply repeat the same failure.

A much stronger pattern is:

```text
Generate
   ↓
Validate
   ↓
Specific error
   ↓
Correct
   ↓
Revalidate
```

### Five sentences to memorize today

> **Explicit criteria reduce ambiguity.**

> **Few-shot examples clarify decision boundaries.**

> **Schemas constrain structure; validators enforce correctness.**

> **Unknown values should not be guessed.**

> **Retries should include actionable feedback.**
