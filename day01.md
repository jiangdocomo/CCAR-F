## CCAR-F Daily Practice — Day 1

Today’s set focuses on **Domain 1: Agentic Architecture & Orchestration**, the highest-weighted domain at **27%**. The current CCAR-F exam is English-only and uses scenario-based multiple-choice/multiple-response questions, so the questions below are intentionally written in that style. ([ClaudePrep][1])

### Scenario

**FinServe Inc. is building a Claude-based customer dispute investigation system.**

When a customer disputes a transaction, the system may need to:

* retrieve the customer profile;
* search transaction history;
* inspect merchant information;
* check previous disputes;
* ask specialized subagents to analyze fraud indicators;
* request human approval for high-value refunds.

The required investigation path is not known in advance. Claude must decide which information to retrieve based on the results of previous steps.

---

### Question 1 — Agentic Loop

Claude requests the `search_transactions` tool. The API response has:

`stop_reason = "tool_use"`

What should the application do **NEXT**?

A. End the agent loop and return Claude's current response to the customer.  
B. Execute the requested tool, append the tool result to the conversation, and call Claude again.  
C. Restart the entire conversation with the tool result as a new user message.  
D. Ask Claude whether it intended to call the tool.  

**Correct Answer: B**

**English Explanation:**
`tool_use` means Claude is requesting a tool operation and is waiting for its result. The application should execute the requested tool, return the corresponding `tool_result`, preserve the conversation state, and continue the agentic loop.

**中文解说：**
这是非常典型的 CCAR-F 判断题。

看到：

`stop_reason = tool_use`

第一反应应该是：

**Execute tool → return tool_result → call Claude again**

A 错，因为 `tool_use` 并不表示任务结束。正常完成通常看 `end_turn`。

C 错，因为不能把前面的上下文丢掉重新开始。

D 错，因为 Claude 已经通过结构化的 `tool_use` 明确表达了调用工具的意图，没有必要再问一次。

**Exam takeaway:**
**`tool_use` → continue the loop; `end_turn` → normal completion.**

---

### Question 2 — Workflow vs Agent

FinServe initially proposes this implementation:

`Retrieve profile → Retrieve transactions → Retrieve merchant → Check previous disputes → Run fraud analysis`

However, engineers discover that most disputes require only two or three of these steps, and the next investigation step depends heavily on information discovered during the previous step.

Which architecture is MOST appropriate?

A. A fixed sequential workflow executing every step.  
B. An agentic loop in which Claude selects the next action dynamically.  
C. A batch-processing pipeline.  
D. Five independent agents that always execute in parallel.  

**Correct Answer: B**

**English Explanation:**
An agentic loop is appropriate when the next action cannot be predetermined and depends on intermediate results. A deterministic workflow is preferable when the sequence is known in advance.

**中文解说：**
题眼是：

> **the next investigation step depends on information discovered during the previous step**

也就是说，下一步事先不知道。

因此：

**Dynamic decision → Agent**

A 是典型干扰项。固定 Workflow 适用于：

`A → B → C → D`

步骤提前就确定的情况。

D 也不对。虽然 Parallel Agents 是 CCAR-F 高频知识点，但这里各调查步骤可能有依赖关系，不能看到“多个任务”就全部并行。

**Exam takeaway:**

**Predetermined steps → Workflow**
**Dynamic next-step reasoning → Agent**

---

### Question 3 — Parallel Subagents

**Choose TWO.**

The coordinator needs to perform the following tasks after obtaining the complete transaction record:

1. Analyze merchant risk history.  
2. Analyze the customer's historical spending pattern.  
3. Compare the two analyses and produce a final fraud-risk assessment.  

Which TWO actions provide the best orchestration design?

A. Run tasks 1 and 2 in parallel.  
B. Run tasks 1, 2, and 3 simultaneously.  
C. Wait for tasks 1 and 2 before starting task 3.  
D. Always run all subagents sequentially to preserve deterministic ordering.  

**Correct Answers: A and C**

**English Explanation:**
Merchant-risk analysis and customer-pattern analysis are independent once the transaction record is available, so they can run concurrently. The final comparison depends on both outputs and therefore must run afterward.

**中文解说：**

这里考的是 dependency。

结构应该是：

```text
             Coordinator
              /       \
             /         \
Merchant Analysis    Customer Analysis
             \         /
              \       /
            Risk Assessment
```

1 和 2 没有相互依赖：

**Parallel**

3 需要 1 和 2 的结果：

**Sequential after both complete**

B 错在把有依赖的 Task 3 也同时执行。

D 是另一个常见陷阱：

> 为了 deterministic，所以全部 sequential。

这虽然可能运行，但浪费 latency，而且没有利用 independent tasks 可以并行的特点。

**Exam takeaway:**

**Independent → Parallel**
**Dependent → Sequential**

---

### Question 4 — Prompt vs Programmatic Enforcement

Company policy states:

> Refunds above $5,000 MUST receive human approval before the refund tool can execute.

Which solution provides the STRONGEST guarantee?

A. Add “Never issue refunds above $5,000 without approval” to the system prompt.  
B. Add three examples showing Claude that large refunds require approval.  
C. Programmatically prevent the refund tool from executing until a valid human approval is present.  
D. Ask Claude to verify its reasoning before issuing a large refund.  

**Correct Answer: C**

**English Explanation:**
A mandatory security or business constraint should be enforced programmatically. Prompts and examples can influence model behavior but cannot provide the same deterministic guarantee as an application-level control.

**中文解说：**

这是 CCAR-F 最重要的思维模式之一。

题目出现：

> **MUST**

而且涉及：

> **$5,000 refund**

属于高风险、不可逆业务操作。

所以不能仅仅：

`Tell Claude not to do it.`

必须：

`Application / Hook / Permission check → block tool execution`

A、B、D 都是在增强 Claude 的行为倾向，但都不是**强制保证**。

记住一个非常实用的考试判断：

**Preference → Prompt**

**Guarantee → Code / Hook / Permission**

题目里出现这些词时尤其警惕：

`must` / `never` / `authorization` / `security` / `payment` / `compliance`

---

### Question 5 — Coordinator Responsibilities

The fraud system uses a coordinator agent and several specialist subagents.

Which responsibility should PRIMARILY belong to the **coordinator**?

A. Give every specialist the complete conversation history.  
B. Decompose the investigation, delegate bounded tasks, and synthesize the returned results.  
C. Allow specialist agents to freely create additional agents until consensus is reached.  
D. Require every specialist to independently produce the final customer response.  

**Correct Answer: B**

**English Explanation:**
A coordinator should manage decomposition, delegation, relevant context transfer, failure handling, and synthesis. Subagents should generally receive bounded tasks and only the context necessary to complete them.

**中文解说：**

Coordinator 可以理解成：

```text
             Coordinator
          /       |       \
         ↓        ↓        ↓
      Agent A   Agent B   Agent C
          \       |       /
           \      |      /
             Results
                ↓
           Coordinator
                ↓
          Final decision
```

B 完整体现了 coordinator 的职责。

A 是高频陷阱：

> “为了保证 subagent 信息完整，把整个 history 都传过去。”

通常不是最佳设计。

应该遵循：

**Minimum sufficient context**

给 subagent **完成任务所需要的上下文**，而不是无脑复制整个 conversation。

C 会造成 uncontrolled agent proliferation。

D 则破坏 coordinator 的 synthesis 职责。

---

### Question 6 — Context Isolation

A merchant-analysis subagent only needs:

* merchant ID,
* transaction details,
* merchant history.

The main conversation contains 40 pages of unrelated customer-support history.

What should the coordinator do?

A. Pass the complete conversation so the subagent has maximum context.  
B. Pass only the information required for merchant analysis.  
C. Summarize all 40 pages and send the entire summary.  
D. Start a completely unrelated session without transaction information.  

**Correct Answer: B**

**English Explanation:**
Subagents should receive sufficient but focused context. Excessive unrelated context increases token usage and can reduce reasoning quality through context pollution.

**中文解说：**

这道题很有 CCAR-F 风格，因为 A 看起来很合理：

> 信息越多越好。

但 Agent Architecture 里通常不是这样。

应该：

**Relevant context > Maximum context**

也就是：

`merchant ID + transaction + merchant history`

已经够完成任务，就不要塞 40 页客服聊天记录。

A：context pollution。
C：虽然比 A 好一点，但仍然把大量无关内容带进去。
D：又走到了另一个极端——必要信息也没有。

**Key phrase:**
**Minimum sufficient context**

---

### Question 7 — Failure Handling

A merchant-risk subagent calls an external service and receives:

```text
error_type: rate_limit
retryable: true
retry_after: 5 seconds
```

What is the BEST initial response?

A. Immediately escalate the entire dispute to a human.  
B. Return “FAILED” to the coordinator with no additional information.  
C. Apply the defined retry policy and preserve structured error information if the retry ultimately fails.  
D. Ignore merchant-risk analysis and mark the transaction safe.  

**Correct Answer: C**

**English Explanation:**
A retryable transient error should normally be handled locally according to a bounded retry policy. If recovery fails, structured failure information should propagate to the coordinator so it can make an informed decision.

**中文解说：**

看到：

`retryable: true`

以及：

`rate_limit`

通常意味着：

**Transient failure**

所以第一选择是：

**bounded retry**

而不是马上找人工。

如果最终还是失败，应向 Coordinator 返回类似：

```text
error_type
attempted_action
retry_count
partial_result
failure_reason
```

而不是只返回：

`FAILED`

A 是典型的 **over-escalation**。

D 更严重，不能因为一个检查失败就默认交易安全。

**Exam takeaway:**
**Recover locally when possible; escalate when necessary.**

---

### Question 8 — Termination Guard

Engineers are concerned that Claude might repeatedly alternate between two investigation tools and never finish.

Which design is BEST?

A. Terminate every agent after exactly five tool calls.  
B. Use normal semantic termination such as `end_turn`, with a maximum-iteration limit as a safety guard.  
C. Remove all tools after the first tool call.  
D. Ask Claude in the system prompt to “please avoid infinite loops.”  

**Correct Answer: B**

**English Explanation:**
Normal completion should follow the model/API termination signal. A bounded iteration limit provides protection against pathological loops without prematurely terminating valid investigations.

**中文解说：**

这里同时考两个概念：

**Normal termination**

和

**Safety guard**

正常情况下：

`end_turn → finish`

异常情况下，如果 Claude 一直：

`tool A → tool B → tool A → tool B...`

则：

`max_iterations`

作为保险。

所以结构应该是：

```text
while iterations < MAX:
    call Claude

    if stop_reason == tool_use:
        execute tool
        continue

    if stop_reason == end_turn:
        break
```

A 的问题是：

**5 次不是业务完成条件。**

有些正常调查可能需要 6 次工具调用。

D 还是经典陷阱：

**Prompt ≠ deterministic enforcement**

---

## Day 1 Score Guide

**8/8** — Excellent. Your Domain 1 mental model is strong.
**6–7/8** — Good, but review the distractors you selected.
**4–5/8** — Revisit agent vs workflow, orchestration, and enforcement.
**0–3/8** — Review Domain 1 fundamentals before moving to mixed-domain scenarios.

今天最值得背下来的不是 8 道题本身，而是这 **6 个英文判断句**：

> **Dynamic decision-making → Agentic loop**
> **Predetermined sequence → Workflow**
> **Independent tasks → Parallel execution**
> **Dependent tasks → Sequential execution**
> **Hard guarantee → Programmatic enforcement**
> **Minimum sufficient context → Better subagent isolation**

尤其注意今天反复出现的一个 **CCAR-F distractor pattern**：

> **“Put it in the prompt.”**

当题目要求的是 **guarantee / must / security / authorization / deterministic behavior** 时，这往往不是最佳答案；考试更偏向选择能够在架构层真正 enforce 规则的方案。

当前蓝图中 **Agentic Architecture & Orchestration 占 27%，约对应 60 道题中的 16 道**，因此把这一套判断模式练熟是很划算的。CCAR-F 当前为 60 题、120 分钟、4 个场景（从 6 个中抽取），并包含单选和多选题。([CCAF Preparation][2])

[1]: https://claude.termidy.com/ccar-f/?utm_source=chatgpt.com "CCAR-F: Claude Certified Architect Foundations Exam Guide + Free Practice"
[2]: https://www.ccafpreparation.com/?utm_source=chatgpt.com "Claude Certified Architect Foundations (CCAR-F) Exam Prep"
