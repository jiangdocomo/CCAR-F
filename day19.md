# CCAR-F Daily Practice — Day 19

Today’s set focuses on **production agent reliability under changing state**: **optimistic concurrency, versioned state, stale reads, checkpoints, resumability, escalation boundaries, tool-result validation, graceful degradation, and audit provenance**.

**Exam mode:** 10 questions. **Target time: 20 minutes.** Questions 5 and 9 are **Choose TWO**.

---

## Scenario: Orion Enterprise Access Agent

Orion Corp. uses Claude to investigate and process enterprise access requests.

The system contains:

* `CoordinatorAgent` — orchestrates each access case.
* `IdentityAgent` — investigates employee identity and employment status.
* `PolicyAgent` — retrieves access-control policies.
* `SecurityAgent` — evaluates security risk.
* `ProvisioningAgent` — performs approved access changes.
* `ReviewAgent` — reviews privileged-access requests.

Available capabilities include:

```text
get_employee
get_current_access
retrieve_access_policy
search_security_events
calculate_risk_score
create_review_case
grant_access
revoke_access
send_notification
```

Some investigations last several hours. During that time, employee status, policy versions, and existing permissions can change.

---

## Question 1 — Stale Read Before Write

At 09:00, `ProvisioningAgent` reads:

```text
employee = E-1042
current_access = NONE
```

At 09:20, an administrator independently grants the employee `READ_ONLY` access.

At 09:30, Claude attempts to grant `ADMIN` access based on the state it read at 09:00.

What is the BEST design?

**A.** Execute because Claude already retrieved the state once.  
**B.** Revalidate relevant mutable state immediately before the high-impact write.  
**C.** Ask Claude whether 30 minutes seems stale.  
**D.** Put the 09:00 result into the system prompt.  

### Correct Answer: **B**

**English Explanation:**
Mutable state used to authorize or construct a high-impact action can become stale. Relevant preconditions should be revalidated close to execution.

### 中文解说

今天第一个重点：

**Stale Read**

读取时：

```text
09:00
Access = NONE
```

不代表执行时：

```text
09:30
Access = NONE
```

中间可能发生：

```text
Admin
System
Another agent
Workflow
```

修改状态。

所以高影响 write 前应该检查：

> **我做决定时依赖的关键状态，现在还成立吗？**

A 是典型：

> “我之前查过了。”

这在 mutable system 中不够。

C 把 freshness 变成 Claude 的主观判断。

**Key exam takeaway:**

> **Read once is not enough when mutable state matters at execution time.**

---

## Question 2 — Optimistic Concurrency

Two agents simultaneously process the same employee.

Both retrieve:

```text
access_version = 17
```

Agent A updates access successfully, producing:

```text
access_version = 18
```

Agent B then submits its update based on version 17.

Which mechanism BEST prevents Agent B from silently overwriting Agent A's change?

**A.** Require Agent B to submit the expected version and reject the update if the current version no longer matches.  
**B.** Tell both agents to work carefully.  
**C.** Let the last write always win.  
**D.** Increase model context.  

### Correct Answer: **A**

**English Explanation:**
This is **optimistic concurrency control**. The write is accepted only if the state still matches the version on which the proposed change was based.

### 中文解说

今天第二个重要术语：

**Optimistic Concurrency Control**

Agent B 的逻辑建立在：

```text
version = 17
```

之上。

但执行时已经：

```text
version = 18
```

所以：

```text
update_access(
    expected_version = 17
)
```

系统发现：

```text
current_version = 18
```

于是：

```text
CONFLICT
```

而不是直接覆盖。

C 的：

**Last Write Wins**

在敏感权限系统中可能导致丢失更新。

**Key exam takeaway:**

> **Version checks prevent stale decisions from silently overwriting newer state.**

---

## Question 3 — Checkpointing

A complex security investigation runs for two hours and performs 40 successful read-only investigation steps.

Immediately before the final review, the worker process crashes.

The system currently restarts the entire investigation from step 1.

What is the BEST improvement?

**A.** Increase temperature so the restarted investigation finishes differently.  
**B.** Persist useful workflow state/checkpoints so execution can resume safely from an appropriate point.  
**C.** Disable crashes in the prompt.  
**D.** Put all 40 results into `CLAUDE.md`.  

### Correct Answer: **B**

**English Explanation:**
Long-running workflows benefit from durable checkpoints that preserve completed work and allow safe recovery without unnecessarily repeating the entire process.

### 中文解说

这是：

**Checkpointing / Resumability**

长任务：

```text
Step 1
 ↓
...
 ↓
Step 40
 ↓
CRASH
```

不应该一定：

```text
Restart → Step 1
```

可以保存：

```text
verified findings
completed stages
evidence references
workflow state
checkpoint ID
```

然后：

```text
Restart
  ↓
Load checkpoint
  ↓
Validate state
  ↓
Resume
```

注意：

**Resume ≠ blindly continue**

恢复时仍可能需要检查外部 mutable state。

**Key exam takeaway:**

> **Checkpoint durable progress; revalidate mutable assumptions when resuming.**

---

## Question 4 — Tool-Result Validation

`get_employee` is expected to return:

```json
{
  "employee_id": "E-1042",
  "status": "ACTIVE",
  "department": "Finance"
}
```

Instead, because of an upstream service bug, it returns:

```json
{
  "employee_id": null,
  "status": "ACTVE",
  "department": 472
}
```

What should the agent system do?

**A.** Let Claude infer the intended values and continue.  
**B.** Validate the tool result against the expected contract and treat invalid output as an integration/tool failure.  
**C.** Convert every value to a string.  
**D.** Assume `ACTVE` means `ACTIVE`.  

### Correct Answer: **B**

**English Explanation:**
Structured contracts should be validated at system boundaries. Malformed tool output should not silently become trusted agent state.

### 中文解说

以前练过：

> Claude → Tool 参数 validation

今天反过来：

> **Tool → Claude 的结果也要 validation**

边界：

```text
External service
      ↓
Tool result
      ↓
Schema validation
      ↓
Trusted application state
```

如果直接让 Claude猜：

```text
ACTVE probably means ACTIVE
```

可能“看起来聪明”，但生产系统风险很高。

特别是：

```text
status
amount
permission
identity
```

这类关键字段。

**Key exam takeaway:**

> **Validate both inputs to tools and outputs from tools.**

---

## Question 5 — Safe Resumption

### Choose TWO.

A workflow checkpoint says:

```text
employee_status = ACTIVE
policy_version = 31
approval = APPROVED
```

The workflow resumes 12 hours later before executing `grant_access`.

Which TWO checks are MOST important?

**A.** Verify that mutable employee status and relevant policy state are still current.  
**B.** Verify that the approval remains valid for the exact action that will execute.  
**C.** Trust the checkpoint because persisted data cannot become stale.  
**D.** Skip all validation because resumption should be fast.  

### Correct Answers: **A and B**

**English Explanation:**
A checkpoint preserves what was previously known; it does not guarantee that external mutable state remains unchanged. Execution-time assumptions and authorization should be refreshed when necessary.

### 中文解说

非常重要：

> **Checkpoint ≠ current truth**

Checkpoint 只是：

```text
12 hours ago, we knew...
```

不是：

```text
Now, this is still true.
```

例如 12 小时内：

```text
Employee terminated
Policy v32 published
Approval revoked
Target access changed
```

都有可能。

所以：

**A — State freshness**

**B — Authorization freshness**

C 是今天的典型陷阱。

**Key exam takeaway:**

> **Persisted state preserves history; revalidation establishes current truth.**

---

## Question 6 — Graceful Degradation

`SecurityAgent` normally uses three sources:

```text
identity history
security-event history
device-risk history
```

The device-risk service is temporarily unavailable.

Policy explicitly allows the agent to continue for low-risk access requests if it clearly records that device risk was unavailable, but privileged-access requests require all three sources.

What is the BEST behavior?

**A.** Fail every request.  
**B.** Ignore the missing source silently.  
**C.** Continue low-risk requests with explicit degraded-state tracking, but fail or escalate privileged requests requiring complete evidence.  
**D.** Ask Claude to invent a likely device-risk score.  

### Correct Answer: **C**

**English Explanation:**
Graceful degradation should follow explicit business policy. Missing noncritical evidence may permit bounded continuation, while required evidence for high-impact decisions must not be silently bypassed.

### 中文解说

这是：

**Graceful Degradation**

不是两个极端：

```text
一个 service 坏了
→ 全系统停止
```

也不是：

```text
一个 service 坏了
→ 当作没事
```

而是根据：

**criticality / policy**

处理。

这里题目明确：

```text
Low-risk
→ may continue

Privileged
→ all sources required
```

所以 C。

关键是：

> degraded state 必须显式记录。

例如：

```text
device_risk = UNAVAILABLE
decision_confidence_basis = PARTIAL
```

而不是伪造：

```text
device_risk = LOW
```

**Key exam takeaway:**

> **Degrade only where policy permits, and preserve the fact that evidence was incomplete.**

---

## Question 7 — Escalation Boundary

`SecurityAgent` encounters a request involving a newly created privileged role that is not covered by any current policy.

What should the agent do?

**A.** Infer a new access policy from similar previous cases and grant access.  
**B.** Recognize that the case is outside its authorized decision boundary and escalate for human review.  
**C.** Choose the most permissive interpretation.  
**D.** Retry policy retrieval until a matching policy appears.  

### Correct Answer: **B**

**English Explanation:**
Agents should have explicit boundaries for autonomous decision-making. When a high-impact case falls outside established policy, escalation is safer than inventing authority.

### 中文解说

这是：

**Escalation Boundary**

Agent autonomy 不是：

> “遇到没规则的情况自己创造规则。”

应该区分：

```text
Inside policy boundary
→ autonomous decision may be allowed

Outside policy boundary
→ escalate
```

尤其这里是：

**privileged role**

高影响操作。

A 的问题不是 Claude 推理能力够不够。

而是：

> **Claude 没有 authority 创造新的企业 access policy。**

**Key exam takeaway:**

> **Capability to reason does not imply authority to decide.**

---

## Question 8 — Audit Provenance

Six months after access was granted, an auditor asks:

> Why did E-1042 receive ADMIN access?

Which record is MOST useful?

**A.** Only the final statement: “Access granted successfully.”  
**B.** A durable record linking the decision to relevant evidence, policy version, approval, requested action, and execution result.  
**C.** Claude's current reconstruction of what probably happened.  
**D.** The total number of tokens used.  

### Correct Answer: **B**

**English Explanation:**
Auditability requires durable provenance connecting the action to the evidence, rules, authorization, and actual execution that supported it.

### 中文解说

好的 audit record 应该能回答：

```text
WHAT happened?
WHO approved?
WHY was it allowed?
WHICH policy was used?
WHAT evidence supported it?
WHAT exactly executed?
```

例如：

```text
case_id
employee_id
requested_role
evidence_ids
policy_version
approval_id
decision
tool_request_id
execution_result
timestamp
```

A 只有结果，没有 reasoning basis。

C 是事后 reconstruction。

**Key exam takeaway:**

> **Auditability requires reconstructable evidence and decision provenance, not model memory.**

---

## Question 9 — Production Evaluation

### Choose TWO.

Orion evaluates access-provisioning reliability.

Which TWO test cases are especially important in addition to ordinary successful requests?

**A.** Concurrent updates to the same employee's access.  
**B.** Workflow resumption after external state or policy has changed.  
**C.** Only cases where every dependency works perfectly.  
**D.** Only short cases that finish in one model turn.  

### Correct Answers: **A and B**

**English Explanation:**
Production reliability depends on behavior under concurrency, stale state, interruptions, and recovery—not only ideal-path correctness.

### 中文解说

今天的主题就是：

**Happy Path 不够。**

生产 Eval 应该主动测试：

```text
Concurrent writes
Stale reads
Crash + resume
Policy changes
Approval changes
Partial dependency failure
Duplicate requests
Malformed tool output
```

为什么？

因为系统最危险的问题往往不发生在：

```text
Everything works perfectly.
```

而发生在：

```text
Two things happen at once.
Something changes midway.
Something partially fails.
```

**Key exam takeaway:**

> **Evaluate recovery and concurrency paths, not only happy paths.**

---

## Question 10 — Failure-Layer Diagnosis

Orion observes:

> Claude correctly determines that ADMIN access requires approval.
> A valid approval exists.
> `grant_access` receives the correct employee and role.
> Meanwhile, another process revokes the employee's eligibility.
> `grant_access` still succeeds because it never verifies that the eligibility version matches the state used to make the decision.

Which improvement MOST directly addresses the failure?

**A.** Add more examples to the prompt.  
**B.** Increase Claude's reasoning effort.  
**C.** Add execution-time precondition/version checking to the state-changing operation.  
**D.** Make the tool description longer.  

### Correct Answer: **C**

**English Explanation:**
The model made the correct decision using the state available to it. The failure occurred because the write did not enforce that its decision preconditions were still valid at execution time.

### 中文解说

逐层排查：

```text
Reasoning       ✓
Policy          ✓
Approval        ✓
Arguments       ✓

State changed concurrently
              ↓
Write still executed ✗
```

所以根本问题是：

**Concurrency / Stale Preconditions**

可以设计：

```text
grant_access(
    employee_id = E-1042,
    role = ADMIN,
    expected_eligibility_version = 44
)
```

执行时：

```text
current_version == 44?
```

如果不是：

```text
CONFLICT
→ refresh
→ reconsider
```

这比修改 Prompt 强得多。

---

# Day 19 — New High-Value Terms

| Term                               | Meaning                                                           |
| ---------------------------------- | ----------------------------------------------------------------- |
| **Stale read**                     | A previously retrieved value no longer reflects current state     |
| **Optimistic concurrency control** | Execute a write only if the expected state/version still matches  |
| **Checkpoint**                     | Durable record of useful workflow progress                        |
| **Resumability**                   | Ability to continue safely after interruption                     |
| **Precondition**                   | State that must remain true for an operation to be valid          |
| **Graceful degradation**           | Continue with reduced capability only where explicitly acceptable |
| **Escalation boundary**            | Point beyond which autonomous decision-making is not authorized   |
| **Provenance**                     | Traceability from conclusion/action back to evidence and rules    |

## Three States You Must Not Confuse

> **Previously true**

A checkpoint or tool result tells you what was true earlier.

> **Currently true**

Fresh retrieval or version validation tells you what is true now.

> **Authorized to act**

Approval/policy tells you whether the operation may be performed.

These are separate questions.

For example:

```text
Employee was ACTIVE.       ✓ previously true
Employee is ACTIVE now.    ? must verify
ADMIN access approved.     ✓ authorization
```

One does not prove the others.

## Day 19 Common Trap

A very plausible distractor is:

> **“The agent already checked that earlier.”**

When the question involves:

* long-running tasks;
* concurrent actors;
* approval;
* policy updates;
* account balances;
* permissions;
* inventory;
* external state;

ask immediately:

> **Could this fact have changed between check and execution?**

If yes, think:

**freshness → versioning → preconditions → execution-time revalidation.**

### Eight sentences to memorize

> **Mutable state can become stale between read and write.**

> **Version checks prevent silent lost updates.**

> **Checkpoint progress, but revalidate mutable assumptions after resumption.**

> **Tool results should be validated before becoming trusted state.**

> **Persisted state is history, not proof of current truth.**

> **Graceful degradation must follow explicit policy.**

> **Agents should escalate when a case exceeds their decision authority.**

> **High-impact writes should enforce their preconditions at execution time.**
