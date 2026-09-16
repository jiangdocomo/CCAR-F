# CCAR-F Daily Practice — Day 11

Today’s set is a **hard mixed-domain scenario**. New emphasis: **evaluation design, golden datasets, precision vs. recall, regression testing, fallback behavior, and model-vs-system failure diagnosis**.

**Exam mode:** 10 questions. Target time: **20 minutes**. Questions 4 and 9 are **Choose TWO**.

---

## Scenario: Northstar Claims Automation

**Northstar Insurance uses Claude to investigate and process commercial insurance claims.**

The system includes:

* `CoordinatorAgent` — manages the claim investigation.
* `DocumentAgent` — extracts facts from submitted documents.
* `PolicyAgent` — retrieves applicable policy language.
* `FraudAgent` — identifies potentially suspicious claims.
* `ReviewAgent` — independently reviews high-impact decisions.

Available tools include:

```text
get_claim
retrieve_policy
search_claim_history
extract_document
calculate_claim_amount
create_review_case
approve_claim
deny_claim
```

Northstar is preparing the system for wider production deployment. The engineering team now needs to determine whether changes to prompts, tools, models, and orchestration actually improve system quality.

---

## Question 1 — Representative Evaluation Set

Northstar tests `FraudAgent` on 100 claims. All 100 are simple claims from one product line and contain complete documentation.

The agent achieves **98% accuracy**.

Production traffic, however, contains:

* multiple product lines;
* incomplete documents;
* multilingual evidence;
* unusual fraud patterns;
* conflicting policy information.

What is the MOST important evaluation improvement?

**A.** Increase the test set to 1,000 simple claims from the same product line.
**B.** Build an evaluation set representative of important production distributions, edge cases, and failure modes.
**C.** Run the same 100 claims ten times and average the result.
**D.** Ask Claude whether 98% accuracy seems sufficient.

### Correct Answer: **B**

**English Explanation:**
Evaluation quality depends on whether the test set represents the conditions that matter in production. A high score on a narrow, easy dataset may provide little evidence about real-world reliability.

### 中文解说

这题考的是：

**Representative Evaluation Dataset**

98% 看起来很好，但测试数据全部是：

> 简单 + 同一产品 + 文档完整

这叫 evaluation distribution 太窄。

真实环境却有：

```text
multilingual
missing data
conflicting evidence
rare fraud
different products
```

所以测试集必须覆盖这些情况。

A 只是把：

**100 个简单 Case → 1000 个简单 Case**

数量增加了，但代表性没有提高。

C 可以测试一定的运行波动，却不能解决数据分布问题。

D 让模型自己判断自己的 98% 是否足够，没有客观意义。

**Exam takeaway:**

> **A large evaluation set is not necessarily a representative evaluation set.**

---

## Question 2 — Golden Dataset

Northstar wants to compare two versions of its claim-extraction prompt.

Engineers need a stable set of claims with trusted expected outputs so that both prompt versions can be measured against the same reference.

What is this MOST naturally called?

**A.** Retry budget
**B.** Golden dataset
**C.** Active context
**D.** Compensating transaction

### Correct Answer: **B**

**English Explanation:**
A golden dataset contains curated examples with trusted expected outputs or labels and provides a stable reference for evaluating changes.

### 中文解说

今天第一个重点术语：

**Golden Dataset**

可以理解成：

> 一套经过人工确认、答案可信的标准测试数据。

例如：

```text
Claim 001
Expected:
policy_number = P123
claim_amount = 5000
fraud = false

Claim 002
Expected:
...
```

然后：

```text
Prompt V1 → Golden Dataset → Score A
Prompt V2 → Golden Dataset → Score B
```

这样才能公平比较。

**Exam takeaway:**

> **Golden datasets provide stable ground truth for repeatable evaluation.**

---

## Question 3 — Precision vs. Recall

`FraudAgent` identifies claims that should receive additional human review.

Northstar is especially concerned about **missing genuinely fraudulent claims**, even if that means some legitimate claims are also sent for review.

Which metric deserves particular attention?

**A.** Recall for the fraudulent class
**B.** Precision only
**C.** Average response length
**D.** Tool-call count

### Correct Answer: **A**

**English Explanation:**
Recall measures how many actual positive cases are successfully detected. If false negatives—missed fraudulent claims—are especially costly, recall is a critical metric.

### 中文解说

这是考试可能出现的数据评价基础。

假设实际有：

```text
100 个 Fraud
```

系统抓到：

```text
90 个
```

那么 Recall 大致关注：

> **真正应该抓的，我抓到了多少？**

即：

```text
Recall = TP / (TP + FN)
```

如果最怕：

> **漏掉 Fraud**

就是怕：

**False Negative**

所以重点关注：

**Recall**

反过来，如果题目说：

> 人工审核成本极高，希望送去审核的 Case 尽可能真的是 Fraud。

这时会更加关注：

**Precision**

记忆：

> **Recall → Don't miss positives.**

> **Precision → When I say positive, be right.**

---

## Question 4 — Evaluation Dimensions

### Choose TWO.

`DocumentAgent` extracts claim amounts into structured output.

Which TWO evaluation dimensions provide the strongest direct evidence that the extraction is suitable for downstream automation?

**A.** Whether the output satisfies the required schema.
**B.** Whether the extracted amount matches the source evidence correctly.
**C.** Whether Claude uses sophisticated financial terminology.
**D.** Whether the response contains more reasoning tokens.

### Correct Answers: **A and B**

**English Explanation:**
Downstream automation requires both structural correctness and semantic correctness. A perfectly formatted but incorrect amount is unsafe, while a correct value in an unusable structure can still break the application.

### 中文解说

继续强化两个层：

```text
Structural correctness
+
Semantic correctness
```

A：

> 格式是否满足 Schema？

B：

> 值是否真的从文件里提取正确？

两个都需要。

例如：

```json
{
  "claim_amount": 900000
}
```

JSON 完全正确。

但源文件其实写：

```text
90,000
```

那么：

**Schema ✓**

**Semantic accuracy ✗**

C、D 都不直接说明系统能否安全自动化。

**Exam takeaway:**

> **Evaluate both format compliance and factual correctness.**

---

## Question 5 — Regression Testing

Northstar improves the fraud-detection prompt. Performance on difficult fraud cases increases from 72% to 86%.

After deployment, engineers discover that accuracy on simple non-fraud claims dropped from 99% to 91%.

Which practice would have MOST directly exposed this before deployment?

**A.** Regression testing across previously successful evaluation cases.
**B.** Increasing the context window.
**C.** Adding more autonomous subagents.
**D.** Increasing temperature.

### Correct Answer: **A**

**English Explanation:**
Regression testing checks whether a change improves the target behavior without degrading capabilities that previously worked.

### 中文解说

今天第二个重要概念：

**Regression（回归）**

修改目标是：

```text
Hard fraud cases
72% → 86% ✓
```

但是无意中：

```text
Simple cases
99% → 91% ✗
```

所以不能只测试：

> 我这次想改善的 Case。

还必须测试：

> **以前已经正常的 Case 有没有被改坏？**

这就是：

**Regression Suite**

理想流程：

```text
Change
  ↓
New target tests
  +
Existing regression tests
  ↓
Compare
  ↓
Deploy
```

**Exam takeaway:**

> **Improvement on one slice can hide regression on another.**

---

## Question 6 — End-to-End Evaluation

Individual evaluations show:

```text
DocumentAgent accuracy: 97%
PolicyAgent accuracy:   98%
FraudAgent accuracy:    95%
```

However, the complete claims system makes correct final decisions only **84%** of the time.

What is the BEST conclusion?

**A.** The 84% result must be a measurement error because every agent exceeds 95%.
**B.** Component-level evaluation is insufficient; the complete workflow also needs end-to-end evaluation.
**C.** The system should average 97%, 98%, and 95%.
**D.** Agent accuracy guarantees workflow accuracy.

### Correct Answer: **B**

**English Explanation:**
Errors can compound across components, and orchestration, context transfer, tool integration, synthesis, and validation may introduce failures not visible in isolated component tests.

### 中文解说

这是非常重要的系统思维：

> **Good components ≠ good system**

例如：

```text
DocumentAgent
   ↓  slight error
PolicyAgent
   ↓
Coordinator passes wrong context
   ↓
FraudAgent
   ↓
Synthesis mistake
   ↓
Final decision wrong
```

每个 Agent 单独都很优秀，组合起来仍可能出现：

* context transfer error；
* orchestration error；
* integration error；
* accumulated error；
* synthesis error。

C 直接取平均没有意义。

**Exam takeaway:**

> **Evaluate components individually and the workflow end-to-end.**

---

## Question 7 — Deterministic Evaluator

Northstar must verify this rule:

> `approved_amount` must never exceed `policy_limit`.

Both fields are already available as validated numeric values.

What is the BEST evaluator?

**A.** Ask another Claude instance whether the amount seems reasonable.
**B.** Implement a deterministic programmatic check: `approved_amount <= policy_limit`.
**C.** Ask three agents to vote.
**D.** Use a longer evaluation prompt.

### Correct Answer: **B**

**English Explanation:**
When correctness can be determined exactly with a deterministic rule, code is a stronger evaluator than probabilistic model judgment.

### 中文解说

这和前几天的：

> deterministic rule

是一脉相承的。

如果规则已经明确：

```text
approved_amount <= policy_limit
```

就直接：

```text
if approved_amount > policy_limit:
    reject
```

没有必要：

> “Claude，你觉得这个金额是不是超过 limit？”

更没有必要三个 Agent 投票。

记住：

> **LLM-as-a-judge 不是所有 evaluation 的默认答案。**

如果能 deterministic evaluate：

**优先 deterministic。**

---

## Question 8 — LLM-Based Evaluation

Northstar wants to evaluate whether claim summaries are:

* concise;
* professionally written;
* faithful to the evidence;
* focused on decision-relevant information.

There is no simple deterministic formula for overall summary quality.

Which approach is MOST reasonable?

**A.** Use a clearly defined rubric and an evaluator model, ideally calibrated against human judgments.
**B.** Count characters only.
**C.** Treat valid JSON as proof of summary quality.
**D.** Use production approval rate as the only metric.

### Correct Answer: **A**

**English Explanation:**
Subjective or semantic qualities can be evaluated with model-based graders when the rubric is explicit and the evaluator is validated against trusted human judgments.

### 中文解说

这里终于出现：

**LLM-as-a-Judge**

什么时候适合？

例如：

```text
Is summary concise?
Is it relevant?
Does it capture the important evidence?
Is writing professional?
```

这些很难用简单代码判断。

所以可以：

```text
Summary
   ↓
Explicit rubric
   ↓
Evaluator model
   ↓
Score
```

但重点是：

> **Rubric 要明确**

而且最好：

> **和 human judgment 做 calibration / validation**

否则 Judge Model 本身也可能偏。

B 只能测长度。

C 只能测结构。

**Exam takeaway:**

> **Use deterministic evaluators where possible; rubric-based model evaluators where judgment is necessary.**

---

## Question 9 — Production Monitoring

### Choose TWO.

The system performs well on the pre-deployment evaluation set.

Which TWO practices remain important after deployment?

**A.** Monitor production outcomes and failure patterns for distribution shift or new failure modes.
**B.** Feed important newly discovered failures back into the evaluation/regression suite.
**C.** Stop evaluating because pre-deployment tests already passed.
**D.** Automatically treat every production response as correct.

### Correct Answers: **A and B**

**English Explanation:**
Production traffic changes over time and exposes cases that offline evaluations may miss. Monitoring identifies new failures, and adding representative failures to the evaluation suite improves future regression coverage.

### 中文解说

Evaluation 不是：

```text
开发 → 测试 → 上线 → 结束
```

而应该形成闭环：

```text
Build
 ↓
Evaluate
 ↓
Deploy
 ↓
Monitor production
 ↓
Discover failures
 ↓
Add to eval dataset
 ↓
Improve
 ↓
Regression test
 ↓
Deploy
```

这就是非常重要的：

**Evaluation Flywheel**

生产环境可能出现：

* 新 document type；
* policy 更新；
* 用户行为变化；
* distribution shift；
* 从未测试过的 edge case。

所以离线 98% 不代表永远 98%。

**Exam takeaway:**

> **Production failures should become future evaluation cases.**

---

## Question 10 — Failure-Layer Diagnosis

Northstar changes only the `DocumentAgent` prompt.

After the change:

* extraction accuracy improves;
* schema compliance remains unchanged;
* tool behavior remains unchanged;
* final claim-decision accuracy decreases.

What should engineers do NEXT?

**A.** Assume the prompt is better because its component accuracy improved.
**B.** Compare end-to-end evaluation results and inspect how the changed extraction behavior affects downstream agents and decision logic.
**C.** Immediately increase temperature.
**D.** Remove end-to-end tests because they conflict with the component metric.

### Correct Answer: **B**

**English Explanation:**
A component improvement is not sufficient if system-level performance worsens. Engineers should evaluate the downstream effects and optimize for the actual end-to-end objective.

### 中文解说

这是 Day 11 最重要的一道综合题。

局部指标：

```text
DocumentAgent ↑
```

但最终：

```text
Business outcome ↓
```

到底应该相信谁？

如果最终目标是：

> 正确处理 Claim

那么应该优先看：

**End-to-End Metric**

然后调查：

```text
Extraction changed
      ↓
Different representation?
      ↓
Coordinator behavior changed?
      ↓
FraudAgent interpretation changed?
      ↓
Final decision degraded?
```

这也是为什么生产 AI 系统不能只优化：

> “单个 Prompt 看起来回答得更好了。”

**Exam takeaway:**

> **Optimize the system for the end objective, not merely a local metric.**

---

# Day 11 — Evaluation Map

今天新增的核心知识可以整理成：

| Question                                   | Think                          |
| ------------------------------------------ | ------------------------------ |
| Are test cases representative?             | **Evaluation distribution**    |
| Need stable trusted test cases             | **Golden dataset**             |
| Don't miss positive cases                  | **Recall**                     |
| Positive predictions should be trustworthy | **Precision**                  |
| New change breaks old behavior             | **Regression**                 |
| Agents good individually, system poor      | **End-to-end evaluation**      |
| Exact rule can determine correctness       | **Deterministic evaluator**    |
| Subjective quality needs judgment          | **Rubric / LLM evaluator**     |
| New production failure appears             | **Add it to evaluation suite** |

## Precision vs. Recall — Quick Exam Memory

Think about a fraud detector.

**Recall asks:**

> Of all the fraud that actually exists, **how much did I catch?**

**Precision asks:**

> Of everything I called fraud, **how much really was fraud?**

So:

> **Expensive false negatives → prioritize Recall**

> **Expensive false positives → prioritize Precision**

Real systems often care about both, but exam questions usually tell you which failure matters more.

## Today's Common Trap

A metric improves:

> **FraudAgent accuracy: 72% → 86%**

Do **not** immediately conclude:

> “The system improved.”

Ask:

> What happened to other slices?

> What happened to previously working behavior?

> What happened to the end-to-end result?

This gives the Day 11 rule:

> **Local improvement ≠ system improvement.**

### Seven sentences to memorize

> **Evaluation data should represent production reality.**

> **Golden datasets provide repeatable ground truth.**

> **Recall measures how many real positives you find.**

> **Precision measures how trustworthy positive predictions are.**

> **Regression tests protect previously working behavior.**

> **Use deterministic evaluators for deterministic rules.**

> **Optimize end-to-end outcomes, not isolated component scores.**
