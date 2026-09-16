# CCAR-F Daily Practice — Day 17

Today’s set is a **harder evaluation-and-agent-architecture set**. It adds several terms worth knowing for current Claude agent work: **outcome-based evaluation, agent harness, grader selection, pass@k vs. pass^k, balanced eval sets, trial isolation, progressive disclosure, and regression suites**. Anthropic’s January 9, 2026 guidance emphasizes evaluating the model **together with its agent harness**, checking actual environment outcomes, combining code/model/human graders, and accounting for nondeterminism across repeated trials. ([Anthropic][1])

**Exam mode:** 10 questions. Target time: **20 minutes**. Questions 5 and 9 are **Choose TWO**.

---

## Scenario: Nova Customer Operations Agent

Nova operates a Claude-based customer-service platform.

The system contains:

* `CoordinatorAgent` — handles the customer interaction and delegates work.
* `PolicyAgent` — retrieves refund and cancellation policies.
* `OrderAgent` — investigates orders.
* `ResolutionAgent` — performs permitted resolutions.
* `ReviewAgent` — reviews unusual or high-value cases.

Available tools include:

```text
get_order
retrieve_policy
search_customer_history
calculate_refund
process_refund
cancel_order
create_escalation
send_confirmation
```

Nova is preparing a new version of the system and must determine whether it is reliable enough for production.

---

## Question 1 — Transcript vs Outcome

An evaluation asks the agent to issue a $75 refund.

The transcript ends with:

> “Your $75 refund has been successfully processed.”

However, inspection of the test environment shows:

```text
refund_status = NOT_CREATED
```

How should the evaluation score this case?

**A.** Pass, because the final response clearly states that the refund succeeded.
**B.** Pass if Claude's confidence was above 95%.
**C.** Fail, because the required real-world/environment outcome did not occur.
**D.** Pass if the agent called `process_refund` at least once.

### Correct Answer: **C**

**English Explanation:**
For an action-oriented agent, the actual environment state is stronger evidence of task completion than the agent's claim that the action succeeded.

### 中文解说

这是今天非常重要的新概念：

**Outcome-based Evaluation**

不要只看：

```text
Claude said:
"Refund completed."
```

而要检查：

```text
Actual environment:
Refund exists?
```

这里：

```text
Transcript → SUCCESS
Environment → FAILURE
```

所以应该：

**FAIL**

D 也不够。

因为：

```text
Tool called
≠
Tool succeeded
```

例如 Tool 可能返回：

```text
ACCESS_DENIED
TIMEOUT
INVALID_AMOUNT
```

**Key exam takeaway:**

> **For state-changing tasks, evaluate what actually happened—not merely what the agent said happened.**

---

## Question 2 — Agent Harness

Nova changes no model and no prompt.

Instead, engineers modify the surrounding system so that:

* tool results are summarized differently;
* retries are handled differently;
* the context passed between agents changes;
* tool calls are orchestrated differently.

Production behavior changes substantially.

What is the BEST explanation?

**A.** Only the underlying model determines agent performance.
**B.** The agent harness/scaffold is part of the system being evaluated and can materially affect behavior.
**C.** Tool orchestration cannot affect model performance.
**D.** The change proves the model itself was retrained.

### Correct Answer: **B**

**English Explanation:**
An agent is more than the underlying model. The surrounding harness controls context, tools, orchestration, state, and execution behavior, so changes to it can alter end-to-end performance.

### 中文解说

今天第二个重要术语：

**Agent Harness / Scaffold**

可以理解为：

```text
              Agent System
                   │
      ┌────────────┼────────────┐
      ↓            ↓            ↓
    Model        Tools      Orchestration
                   │
              Context/State
                   │
                Retries
```

所以：

> 同一个 Claude model

放进不同 harness，最终能力可能不同。

Anthropic 当前的 agent-eval 指南也明确区分 model 和 harness，并指出评估“agent”时实际上是在评估二者共同工作的结果。([Anthropic][1])

**Key exam takeaway:**

> **Agent performance = model behavior + harness behavior.**

---

## Question 3 — Choosing the Right Grader

Nova must evaluate this deterministic requirement:

> A refund must never exceed the amount originally paid.

The evaluation environment contains:

```text
original_payment = 80
actual_refund = 100
```

Which grader is BEST?

**A.** An LLM judge asking whether the refund “seems reasonable.”
**B.** A deterministic state/code check comparing the two numeric values.
**C.** A customer-satisfaction survey.
**D.** A second agent that votes on the refund.

### Correct Answer: **B**

**English Explanation:**
When success can be objectively verified from environment state, a deterministic grader provides a direct and reliable check.

### 中文解说

不要形成错误习惯：

> “AI Eval → LLM Judge”

Evaluator 应根据任务性质选择。

这里规则是：

```text
refund <= payment
```

完全可以：

```text
assert actual_refund <= original_payment
```

所以最强的是：

**Code-based grader / state check**

LLM Judge 更适合：

* empathy；
* writing quality；
* relevance；
* completeness；
* subjective semantic criteria。

**Key exam takeaway:**

> **Use deterministic graders whenever correctness is deterministic.**

---

## Question 4 — LLM Grader

Nova also needs to evaluate whether customer responses:

* acknowledge frustration appropriately;
* clearly explain the resolution;
* avoid unnecessary verbosity.

There is no single deterministic formula for these qualities.

What is the MOST appropriate approach?

**A.** Use a rubric-based model grader calibrated periodically against human judgment.
**B.** Check whether the response contains the word “sorry.”
**C.** Measure response length only.
**D.** Use refund amount as the communication-quality metric.

### Correct Answer: **A**

**English Explanation:**
Subjective semantic qualities are appropriate candidates for rubric-based model evaluation, particularly when the rubric is explicit and model grading is calibrated against trusted human judgments.

### 中文解说

这就是：

**Model-based Grader / LLM-as-a-Judge**

适合：

```text
Was the response empathetic?
Was it clear?
Was it concise?
Was it relevant?
```

但必须注意两个词：

**Rubric**

和：

**Calibration**

不能只问：

> “这回答好吗？”

更好：

```text
Score 1–5:
1. acknowledges frustration
2. explains resolution
3. avoids irrelevant details
...
```

然后拿部分结果和专家人工评分比较。

Anthropic 的当前 eval 指南同样建议对研究等主观质量使用 rubric-based model graders，并经常与专家人工判断校准。([Anthropic][1])

**Key exam takeaway:**

> **Subjective criterion → explicit rubric + calibrated model/human evaluation.**

---

## Question 5 — Balanced Evaluation

### Choose TWO.

Nova tests whether the agent correctly uses `search_customer_history`.

The entire evaluation suite contains only cases where history **should** be searched.

After optimization, the agent begins searching customer history for almost every request.

Which TWO changes are MOST appropriate?

**A.** Add cases where customer-history search is necessary.
**B.** Add cases where customer-history search should not occur.
**C.** Measure both under-triggering and over-triggering behavior.
**D.** Increase the number of positive-only cases from 500 to 5,000.

### Correct Answers: **B and C**

**English Explanation:**
A one-sided evaluation can optimize the agent toward always performing the tested behavior. Balanced cases should test both appropriate triggering and appropriate non-triggering.

### 中文解说

这是很容易忽略的：

**Balanced Evaluation**

现在 Eval 全是：

```text
Should search → YES
Should search → YES
Should search → YES
```

Agent 最容易学到的行为是什么？

> **Always search.**

然后测试全部通过。

但生产环境会出现：

```text
Should search → YES
Should search → NO
```

所以必须同时测试：

**Under-triggering**

> 该搜索却没搜索。

和：

**Over-triggering**

> 不该搜索却搜索了。

Anthropic 2026 eval 指南也专门强调 balanced problem sets，并以 web search 的 undertriggering / overtriggering 为例说明只测一个方向会造成 one-sided optimization。([Anthropic][1])

**Key exam takeaway:**

> **Test both when a behavior should happen and when it should not.**

---

## Question 6 — pass@k

A coding-style support agent has a **60% probability of solving a task correctly on each independent attempt**.

For this use case, Nova can generate several candidate solutions and succeeds if **at least one** works.

Which evaluation concept is MOST relevant?

**A.** pass@k
**B.** pass^k
**C.** Precision
**D.** Context utilization

### Correct Answer: **A**

**English Explanation:**
`pass@k` measures the probability of obtaining at least one successful result across `k` attempts. It is useful when multiple attempts are acceptable and one working solution is sufficient.

### 中文解说

这是今天必须记的新术语。

### pass@k

问：

> **试 k 次，至少成功一次的概率是多少？**

假设每次成功率 60%。

试 3 次：

失败三次的概率：

```text
0.4 × 0.4 × 0.4 = 0.064
```

所以至少成功一次：

```text
1 - 0.064 = 93.6%
```

因此：

```text
pass@1 = 60%
pass@3 ≈ 93.6%
```

它适合：

> “我可以试几个 solution，只要有一个能用。”

**Key exam takeaway:**

> **pass@k asks whether at least one of k attempts succeeds.**

---

## Question 7 — pass^k

Nova has another requirement:

> A customer-facing refund agent must behave correctly every time. Inconsistent behavior across repeated identical cases is unacceptable.

Which metric is MORE informative?

**A.** pass@k only
**B.** pass^k
**C.** Maximum response length
**D.** Number of available tools

### Correct Answer: **B**

**English Explanation:**
`pass^k` measures the probability that all `k` trials succeed and therefore captures consistency across repeated runs.

### 中文解说

这个符号很容易和上一题混。

### pass@k

```text
k 次里面
至少成功 1 次
```

### pass^k

```text
k 次
全部成功
```

如果单次成功率：

```text
75%
```

连续三次全部成功：

```text
0.75³ ≈ 42%
```

所以一个 Agent：

> 偶尔能做对

和：

> 每次都稳定做对

完全是不同能力。

Anthropic 当前的 eval 指南明确区分这两个指标，并指出 customer-facing agents 往往尤其需要关注一致性，因此 `pass^k` 很有意义。([Anthropic][1])

**Key exam takeaway:**

> **pass@k measures opportunity; pass^k measures consistency.**

---

## Question 8 — Trial Isolation

Nova runs 100 evaluation trials against the same test environment.

Each trial should begin with:

```text
customer_balance = $1,000
refunds = []
```

But previous trials leave refund records behind. Later trials therefore observe state created by earlier trials.

What is the MOST direct problem?

**A.** Insufficient prompt length
**B.** Evaluation trials are not isolated, so shared state can contaminate results.
**C.** Claude needs more tools.
**D.** The evaluation needs a higher temperature.

### Correct Answer: **B**

**English Explanation:**
Evaluation trials should generally begin from controlled, clean state. Shared state between trials can either create failures unrelated to the agent or artificially make later tasks easier.

### 中文解说

这是：

**Trial Isolation**

理想：

```text
Trial 1
→ clean environment

Trial 2
→ clean environment

Trial 3
→ clean environment
```

而不是：

```text
Trial 1
   ↓ leaves data
Trial 2
   ↓ sees Trial 1 data
Trial 3
   ↓ sees both
```

否则测出来的结果不再是纯粹的 Agent 能力。

甚至可能出现两种方向：

**Artificially worse**

或：

**Artificially better**

Anthropic 的 eval 指南强调每个 trial 应从 clean environment 开始，避免 leftover files、cached data 或其他 shared state 导致 correlated failures 或不公平优势。([Anthropic][1])

**Key exam takeaway:**

> **Independent trials require isolated starting state.**

---

## Question 9 — Multi-Grader Evaluation

### Choose TWO.

Nova evaluates a refund workflow with these requirements:

1. The refund must actually exist in the backend.
2. The refund amount must not exceed policy limits.
3. The customer explanation must be clear and empathetic.

Which TWO statements are correct?

**A.** Backend state and numeric policy limits are good candidates for deterministic/code-based graders.
**B.** Communication quality is a reasonable candidate for a rubric-based model or human grader.
**C.** One LLM judge should replace every other grader type.
**D.** The final transcript alone is sufficient to verify all three requirements.

### Correct Answers: **A and B**

**English Explanation:**
Agent evaluations often benefit from multiple grader types. Objective state and numeric constraints can be checked deterministically, while subjective communication quality can be evaluated with a rubric-based model or human grader.

### 中文解说

这是：

**Multi-dimensional Evaluation**

不要问：

> “哪个 grader 最好？”

而应该问：

> **这个 criterion 最适合哪个 grader？**

例如：

```text
Refund exists?
→ State check

Amount <= limit?
→ Code check

Customer response empathetic?
→ Model/human rubric
```

Anthropic 当前 eval 指南将 agent grader 大致分为 code-based、model-based 和 human，并建议根据被评估内容选择合适的 grader，而不是强行只用一种。([Anthropic][1])

**Key exam takeaway:**

> **Use the right grader for each dimension rather than forcing one grader to measure everything.**

---

## Question 10 — Failure-Layer Diagnosis

Nova observes:

> Claude correctly calls `process_refund`.
> The backend successfully creates the refund.
> Claude sends the correct confirmation.
> Yet the evaluation marks the task as failed because the grader checks whether the transcript contains the exact phrase `"REFUND_SUCCESS"`, which the task never required.

What should engineers fix FIRST?

**A.** The Claude prompt
**B.** The refund tool
**C.** The evaluation specification/grader
**D.** The context window

### Correct Answer: **C**

**English Explanation:**
The agent achieved the intended outcome, but the grader imposed an unrelated hidden requirement. Evaluation criteria should align with the task specification and actual success condition.

### 中文解说

这是 Day 17 最容易错的一题。

系统表现：

```text
Tool choice       ✓
Execution         ✓
Backend outcome   ✓
User confirmation ✓
```

但 Eval：

```text
Exact hidden string absent
→ FAIL
```

真正错误的是：

**Grader**

不是 Agent。

如果 task 根本没要求：

```text
"REFUND_SUCCESS"
```

就不能因为没出现这个词判失败。

Anthropic 当前 eval guidance 也强调：grader 检查的内容应当从 task description 中清楚可知，并建议为任务建立 reference solution，以验证任务本身可解且 grader 配置正确。([Anthropic][1])

**Key exam takeaway:**

> **A broken grader can make a correct agent look wrong.**

---

# Day 17 — Evaluation Architecture Map

| Scenario signal                                            | Think first                            |
| ---------------------------------------------------------- | -------------------------------------- |
| Agent says action succeeded                                | **Check actual outcome**               |
| Same model behaves differently after orchestration changes | **Agent harness**                      |
| Exact numeric/state condition                              | **Code-based grader**                  |
| Empathy/relevance/quality                                  | **Rubric-based model/human grader**    |
| Eval tests only “should trigger”                           | **Balanced positive + negative cases** |
| One success among several attempts matters                 | **pass@k**                             |
| Consistency across every attempt matters                   | **pass^k**                             |
| Trials inherit previous state                              | **Trial isolation**                    |
| Workflow has objective + subjective requirements           | **Multiple grader types**              |
| Agent succeeds but eval fails for hidden criterion         | **Broken grader/specification**        |

## pass@k vs. pass^k — Memorize This

Suppose per-trial success is **80%**.

For three attempts:

**pass@3** asks:

> “Did I succeed at least once?”

Approximately:

```text
1 - 0.2³ = 99.2%
```

**pass³** asks:

> “Did I succeed every time?”

```text
0.8³ = 51.2%
```

So an agent can simultaneously have:

> **Excellent pass@k**

and

> **Poor pass^k**

That is not contradictory.

It means:

> “Give it enough attempts and it will probably solve the task, but it is not consistently reliable.”

## One More Current Concept — Progressive Disclosure

Anthropic’s current Skills guidance describes Skills as containing a required `SKILL.md`, with optional scripts, references, and assets. It also emphasizes **progressive disclosure**: lightweight metadata can remain available so Claude knows when a Skill is relevant, while detailed instructions and supporting files are loaded only when needed. ([Anthropic Resources][2])

Think:

```text
Always loaded
→ enough information to discover the Skill

Skill becomes relevant
→ load SKILL.md instructions

More detail needed
→ load referenced files
```

This connects directly to the context-engineering principle you have already practiced:

> **Do not load all potentially useful information into active context all the time.**

### Eight sentences to memorize

> **Evaluate outcomes, not claims of outcomes.**

> **An agent is the model plus its harness.**

> **Use deterministic graders for deterministic conditions.**

> **Use rubric-based graders for subjective semantic quality.**

> **Balanced evals test both triggering and non-triggering.**

> **pass@k measures at-least-one success; pass^k measures consistency.**

> **Evaluation trials should start from isolated state.**

> **When the agent is correct but the score is wrong, inspect the grader.**

[1]: https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents?utm_source=chatgpt.com "Demystifying evals for AI agents \ Anthropic"
[2]: https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf?utm_source=chatgpt.com "Chapter 1

\# Fundamentals

\## What is a skill?

A"
