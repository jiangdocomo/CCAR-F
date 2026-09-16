# CCAR-F Daily Practice — Day 14

Today’s set moves into **architecture trade-offs under real production constraints**: agent autonomy, prompt caching, context strategy, model routing, latency/cost optimization, evaluation, and failure isolation. The current certification is Anthropic’s **Claude Certified Architect, Foundations**, described by Anthropic as a technical certification for solution architects building production applications with Claude. ([Anthropic][1])

**Exam mode:** 10 questions. **Target time: 20 minutes.** Questions 5 and 9 are **Choose TWO**.

---

## Scenario: Meridian Legal Intelligence Platform

Meridian Legal operates a Claude-based platform that reviews commercial contracts for enterprise customers.

The system contains:

* `CoordinatorAgent` — manages each review.
* `ExtractionAgent` — extracts clauses and contractual facts.
* `PolicyAgent` — retrieves the customer’s legal-review policy.
* `RiskAgent` — identifies deviations and contractual risks.
* `ReviewAgent` — independently reviews high-risk findings.

Each request may include a 100–300 page contract.

The same customer-specific legal handbook, approximately 80 pages long, is used repeatedly across thousands of requests.

---

## Question 1 — Stable vs Dynamic Context

Each request contains:

**Stable content**

* system instructions;
* tool definitions;
* customer legal handbook.

**Dynamic content**

* the contract being reviewed;
* retrieved evidence;
* intermediate findings.

Meridian wants to reduce repeated processing cost without changing behavior.

Which architecture is MOST appropriate?

**A.** Randomly reorder all content on every request.  
**B.** Keep reusable stable content organized consistently so that caching can benefit repeated requests, while placing request-specific content separately.  
**C.** Duplicate the legal handbook several times inside every request.  
**D.** Replace the legal handbook with a one-sentence summary.  

### Correct Answer: **B**

**English Explanation:**
Large, repeatedly reused prompt content is a strong candidate for caching. Keeping stable reusable content separate from dynamic request-specific information improves the opportunity to reuse previously processed context.

### 中文解说

这里考一个很实用的架构判断：

**Stable Context vs Dynamic Context**

可以想成：

```text
STABLE
────────────
System instructions
Tool definitions
80-page handbook

DYNAMIC
────────────
Contract A
Current evidence
Current findings
```

每次都变化的是下面部分。

重复利用的是上面部分。

A 随机改变顺序会破坏稳定性。

C 增加 token，没有意义。

D 虽然节省 token，但可能损失 handbook 的重要细节。

**Exam takeaway:**

> **Large + repeated + stable context → think caching before deleting useful information.**

---

## Question 2 — Retrieval vs Full Context

Meridian maintains a 20,000-page archive of historical legal opinions.

For a typical contract, fewer than 10 opinions are relevant.

What is the BEST design?

**A.** Include all 20,000 pages in every request.  
**B.** Retrieve relevant opinions based on the current case and place only useful evidence into active context.  
**C.** Permanently summarize the entire archive into one paragraph.  
**D.** Ask Claude to answer from general legal knowledge instead.  

### Correct Answer: **B**

**English Explanation:**
Large corpora with sparse per-request relevance are better handled through retrieval. Active context should contain information useful to the current reasoning task rather than the entire available corpus.

### 中文解说

这里要区分 Day 14 的两个场景。

上一题：

> 80 页 handbook，每次基本都要用。

→ **Stable reusable context / caching**

这一题：

> 20,000 页 archive，每次只相关不到 10 篇。

→ **Retrieval**

判断：

```text
Almost always relevant?
→ Keep available in stable context

Huge corpus, small relevant subset?
→ Retrieve
```

A 是典型：

> “Context window 大，所以全部塞进去。”

这是错误思路。

**Exam takeaway:**

> **Available information ≠ active-context information.**

---

## Question 3 — Context Compression

During a long review, `RiskAgent` accumulates:

* 140 pages of raw tool output;
* 25 rejected hypotheses;
* 11 verified risk findings;
* 6 unresolved questions.

What should remain MOST salient in the agent's working state?

**A.** Every raw token in chronological order.  
**B.** Verified findings, unresolved questions, current objective, and references allowing important evidence to be retrieved.  
**C.** Only the most recent message.  
**D.** All rejected hypotheses repeated after every tool call.  

### Correct Answer: **B**

**English Explanation:**
Long-running agents benefit from a compact working state that preserves durable facts and unresolved work while allowing bulky evidence to remain externally retrievable.

### 中文解说

这题进一步强化：

**Working State**

长期 Agent 最重要的不是：

> “记住所有发生过的话。”

而是：

```text
What do we know?
What remains unresolved?
What are we trying to accomplish?
Where is the supporting evidence?
```

因此 working state 可以类似：

```text
Verified:
- Clause 8 violates policy P17
- Liability cap is absent

Unresolved:
- Governing-law exception
- Data-retention requirement

Evidence:
- doc:contract §8
- policy:P17
```

被证明错误的 hypothesis 不应该一直占据核心 context。

**Exam takeaway:**

> **Agent memory should represent current knowledge state, not merely conversation history.**

---

## Question 4 — Model Routing

Meridian has two classes of tasks:

**Task A:** classify thousands of straightforward clauses into known categories.

**Task B:** analyze a highly unusual indemnification structure involving conflicting provisions across multiple sections.

The company wants to balance quality, latency, and cost.

What is the BEST general strategy?

**A.** Always use the most capable and expensive configuration for every task.  
**B.** Use an appropriately capable lower-cost configuration for routine tasks and route genuinely difficult reasoning tasks to a more capable configuration when justified.  
**C.** Always use the cheapest configuration regardless of task complexity.  
**D.** Randomly select a configuration to avoid systematic bias.  

### Correct Answer: **B**

**English Explanation:**
Different tasks have different capability requirements. Routing by task complexity can improve cost and latency while reserving higher-capability reasoning for cases where it materially improves quality.

### 中文解说

这是：

**Model Routing**

不要陷入两个极端：

```text
Everything → strongest model
```

或者：

```text
Everything → cheapest model
```

更合理的是：

```text
Routine classification
→ sufficient lower-cost path

Complex ambiguous reasoning
→ stronger reasoning path
```

但是注意：

不能凭感觉就上线。

应该用 evaluation 验证：

> 哪些任务 lower-cost path 已经足够？

**Exam takeaway:**

> **Use the least costly configuration that reliably meets the task's quality requirement.**

---

## Question 5 — Optimization

### Choose TWO.

Meridian wants to reduce cost and latency while preserving quality.

Which TWO approaches are MOST defensible?

**A.** Measure token usage, latency, and quality on representative workloads before optimizing.  
**B.** Use caching or retrieval strategically where large repeated or sparsely relevant context creates unnecessary processing.  
**C.** Remove all system instructions because they consume tokens.  
**D.** Disable validation because retries cost money.  

### Correct Answers: **A and B**

**English Explanation:**
Optimization should be evidence-driven. Measure the actual workload first, then target structural sources of unnecessary cost such as repeated stable context or irrelevant active context without removing reliability controls.

### 中文解说

这里考：

**Cost Optimization 不能破坏 Reliability。**

正确思路：

```text
Measure
  ↓
Find bottleneck
  ↓
Optimize architecture
  ↓
Re-evaluate quality
```

A 是 measurement。

B 是 architecture optimization。

C、D 虽然都可能“省钱”，但代价是：

**质量和安全性下降。**

考试里如果一个答案是：

> “为了省 token，把安全验证删掉。”

通常应该高度怀疑。

**Exam takeaway:**

> **Optimize waste before removing safeguards.**

---

## Question 6 — Cascading Failure

`ExtractionAgent` incorrectly extracts:

> Liability cap = $50 million

The actual contract says:

> Liability cap = $5 million

`RiskAgent` then correctly reasons from the supplied $50 million value and produces the wrong final recommendation.

Which statement BEST describes the failure?

**A.** `RiskAgent` necessarily has a reasoning defect.  
**B.** An upstream extraction error propagated into downstream reasoning, demonstrating the need to evaluate interfaces and end-to-end behavior.  
**C.** The context window is necessarily too small.  
**D.** The system needs more parallel agents.  

### Correct Answer: **B**

**English Explanation:**
Downstream components can reason correctly from incorrect upstream state and still produce an incorrect system outcome. This is why component accuracy, interface validation, and end-to-end evaluation all matter.

### 中文解说

流程：

```text
Extraction
$5M → $50M   ✗
      ↓
RiskAgent
reasoning     ✓
      ↓
Final result  ✗
```

这里 `RiskAgent` 本身可能完全正确。

问题是：

**Garbage in → reasonable reasoning → garbage out**

所以要测试：

* Extraction accuracy；
* component interface；
* semantic validation；
* end-to-end outcome。

A 是常见误判：

> 最终答案错，所以最后一个 Agent 错。

不一定。

**Exam takeaway:**

> **Trace errors upstream before changing the downstream reasoner.**

---

## Question 7 — Deterministic Post-Processing

Claude extracts:

```json
{
  "contract_value": 12500000,
  "currency": "USD",
  "risk_level": "HIGH"
}
```

Company policy states:

> Every HIGH-risk contract worth more than $10 million must be escalated.

What should determine whether escalation is required?

**A.** A deterministic rule over the validated structured fields.  
**B.** A second Claude call asking whether escalation feels appropriate.  
**C.** Majority voting across five agents.  
**D.** The length of Claude's explanation.  

### Correct Answer: **A**

**English Explanation:**
Once the required structured values are available and the policy is exact, the decision is deterministic and should be enforced programmatically.

### 中文解说

这是非常高频的考试模式。

前半段：

> 从复杂合同里识别 value / risk

可能需要 Claude。

但后半段：

```text
risk == HIGH
AND
value > 10,000,000
```

是明确 Boolean Rule。

直接：

```text
if risk == HIGH and value > 10_000_000:
    escalate()
```

即可。

不要把已经确定的问题再次交给概率模型。

**Exam takeaway:**

> **LLM for interpretation; code for exact policy logic.**

---

## Question 8 — Latency Diagnosis

Meridian's end-to-end review takes 90 seconds.

Measurements show:

```text
Policy retrieval:        3 s
Contract extraction:    28 s
Vendor research A:      20 s
Vendor research B:      21 s
Final synthesis:        18 s
```

Research A and B are independent but currently execute sequentially.

What should engineers investigate FIRST?

**A.** Parallelizing independent research branches.  
**B.** Removing final synthesis.  
**C.** Eliminating policy retrieval.  
**D.** Increasing the number of sequential agents.  

### Correct Answer: **A**

**English Explanation:**
Independent sequential work is an obvious source of avoidable wall-clock latency. Parallel execution can reduce latency without removing necessary work.

### 中文解说

这是：

**Critical Path Thinking**

现在：

```text
Research A 20s
     ↓
Research B 21s

Total ≈ 41s
```

如果独立：

```text
      ┌─ A 20s ─┐
Start ┤         ├→ Continue
      └─ B 21s ─┘
```

理论上这一段接近：

> `max(20,21) ≈ 21s`

而不是 41s。

这比：

> 删除必要的 synthesis

更合理。

**Exam takeaway:**

> **Optimize the critical path, especially independent work accidentally serialized.**

---

## Question 9 — Evaluation Before Optimization

### Choose TWO.

Meridian plans to route routine clauses to a cheaper model configuration.

Which TWO evaluation practices are MOST important before rollout?

**A.** Compare the cheaper route against trusted expected outcomes on representative routine and edge-case examples.  
**B.** Measure end-to-end quality, not only the routed component's local accuracy.  
**C.** Deploy immediately because lower cost proves architectural improvement.  
**D.** Evaluate only token cost because quality is unchanged by model routing.  

### Correct Answers: **A and B**

**English Explanation:**
Routing changes can affect both local task performance and downstream behavior. Representative component tests plus end-to-end evaluation help establish whether the cost saving preserves acceptable quality.

### 中文解说

不能：

```text
Cheaper
= Better architecture
```

必须证明：

```text
Cost ↓
Latency ↓
Quality still acceptable ✓
```

A 测：

**component behavior**

B 测：

**system outcome**

两个都需要。

特别注意：

> Local accuracy 没明显变化

也不代表下游一定没变化。

**Exam takeaway:**

> **Cost optimization is successful only if required quality remains satisfied.**

---

## Question 10 — Failure-Layer Diagnosis

Meridian observes:

> Claude extracted the correct contract value.
> Structured output passed validation.
> The applicable policy was current.
> The rule clearly required escalation.
> Claude's narrative even stated that escalation was required.
> However, the application did not create the review case because escalation depended on parsing the narrative text for the word “escalate,” and the parser missed the phrasing.

Which improvement MOST directly addresses the defect?

**A.** Increase Claude's temperature.  
**B.** Make the narrative explanation longer.  
**C.** Represent the escalation decision as a structured field or derive it deterministically from validated data instead of parsing free-form prose.  
**D.** Add another research agent.  

### Correct Answer: **C**

**English Explanation:**
The model understood the requirement; the failure occurred at the machine interface. Programmatic behavior should depend on structured state or deterministic rules rather than fragile parsing of natural-language prose.

### 中文解说

这题非常接近真正架构题的思路。

逐层检查：

```text
Extraction       ✓
Schema           ✓
Policy freshness ✓
Reasoning        ✓
Narrative        ✓
Application      ✗
```

真正坏的是：

> **Natural language → machine action**

这层 interface。

不要设计：

```text
Claude prose
↓
search for word "escalate"
↓
business action
```

更好：

```json
{
  "risk_level": "HIGH",
  "contract_value": 12500000,
  "requires_escalation": true
}
```

或者更强：

```text
validated risk + value
        ↓
deterministic policy rule
        ↓
requires_escalation = true
```

**Exam takeaway:**

> **Do not use prose as an API when structured state is available.**

---

# Day 14 — Architecture Optimization Map

| Scenario signal                                    | Think first                                   |
| -------------------------------------------------- | --------------------------------------------- |
| Large content reused almost every request          | **Caching / stable context**                  |
| Huge corpus, tiny relevant subset                  | **Retrieval**                                 |
| Long session accumulating noise                    | **Compact working state**                     |
| Easy and hard workloads differ                     | **Model routing**                             |
| Need lower cost                                    | **Measure before optimizing**                 |
| Downstream reasoning wrong because input was wrong | **Error propagation**                         |
| Exact rule after extraction                        | **Deterministic post-processing**             |
| Independent slow branches serialized               | **Parallelize / critical path**               |
| Cheaper model proposed                             | **Evaluate quality before routing**           |
| Application parses prose to trigger action         | **Structured interface / deterministic rule** |

## Four Distinctions to Memorize

> **Caching** asks: “Am I repeatedly processing the same useful content?”

> **Retrieval** asks: “From a huge corpus, which small subset matters now?”

> **Compression/state management** asks: “Of what I already discovered, what must stay salient?”

> **Routing** asks: “How much model capability does this particular task actually require?”

These solve different problems. Do not select one merely because the question mentions “too many tokens.”

## Day 14 Common Trap

A production system is slow or expensive, and an answer proposes:

> remove validation, shorten important instructions, skip evidence, or eliminate required review.

Those options may reduce cost, but they optimize the wrong objective.

The stronger sequence is:

> **Measure → locate waste → change architecture → re-evaluate quality.**

### Eight sentences to memorize

> **Cache repeated stable context; retrieve sparsely relevant corpora.**

> **Working state should preserve knowledge, not transcript volume.**

> **Route tasks according to demonstrated capability requirements.**

> **Optimize measured bottlenecks rather than guessed bottlenecks.**

> **Downstream correctness depends on upstream state quality.**

> **Use deterministic logic when the policy itself is deterministic.**

> **Parallelize independent work on the critical path.**

> **Structured state is a stronger machine interface than free-form prose.**

[1]: https://www.anthropic.com/news/claude-partner-network?cmid=03693516-1d21-4130-ac10-3f9892c0929b&utm_source=chatgpt.com "Anthropic invests $100 million into the Claude Partner Network \ Anthropic"
