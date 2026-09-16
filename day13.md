# CCAR-F Daily Practice — Day 13

Today’s set is a **hard mixed-domain production case**. New emphasis: **Claude Code hooks, permission boundaries, deterministic safeguards, tool-result handling, state synchronization, and choosing between prompt guidance and executable controls**.

**Exam mode:** 10 questions. Target time: **20 minutes**. Questions 4 and 9 are **Choose TWO**.

---

## Scenario: AtlasPay Engineering Platform

**AtlasPay uses Claude Code across a large payment-processing codebase.**

The repository contains:

```text
atlaspay/
├── payments/
├── fraud/
├── settlement/
├── database/
├── infrastructure/
└── tests/
```

Developers use Claude Code to:

* investigate bugs;
* modify Java and TypeScript services;
* run tests;
* review pull requests;
* generate database migrations;
* perform repetitive engineering workflows.

Because AtlasPay processes financial transactions, some development actions require deterministic safeguards.

---

## Question 1 — `CLAUDE.md` vs Executable Control

AtlasPay wants Claude Code to follow this convention:

> “When modifying Java code, prefer constructor injection over field injection.”

Where should this guidance MOST naturally be placed?

**A.** In project instructions such as `CLAUDE.md` or an applicable project rule.  
**B.** In a production authorization service.  
**C.** In a database transaction constraint.  
**D.** In an idempotency key.  

### Correct Answer: **A**

**English Explanation:**
This is persistent coding guidance rather than a hard security boundary. Project instructions are appropriate for conventions that Claude should follow while working in the repository.

### 中文解说

先判断：

> 这是 **Guidance** 还是 **Hard Enforcement**？

“Prefer constructor injection” 是：

**coding convention**

不是：

> “绝对不能执行的危险操作”。

所以适合：

**`CLAUDE.md` / project rule**

B、C、D 都是完全不同层次的机制。

考试里要避免看到：

> financial system

就把所有规则都设计成 production authorization。

**Key exam takeaway:**

> **Persistent development guidance → project instructions.**

---

## Question 2 — Hook vs Prompt

AtlasPay requires:

> Every time Claude modifies Java source files, the formatter must run before the task is considered complete.

Developers currently place this sentence in `CLAUDE.md`, but Claude occasionally finishes without running the formatter.

What is the BEST improvement?

**A.** Repeat the instruction five times in `CLAUDE.md`.  
**B.** Use an appropriate Claude Code hook or deterministic automation to run the formatter when the relevant event occurs.  
**C.** Increase temperature.  
**D.** Ask Claude at the end whether it remembers the rule.  

### Correct Answer: **B**

**English Explanation:**
If an action must happen reliably in response to a known lifecycle event, executable automation is stronger than relying only on probabilistic instruction-following.

### 中文解说

今天第一个重点：

**Hooks**

这里不是：

> “Claude 应该采用什么 coding style？”

而是：

> **发生某事件时必须执行一个动作。**

可以理解为：

```text
Claude edits Java
       ↓
Known lifecycle event
       ↓
Formatter
```

如果只写在 `CLAUDE.md`：

```text
Please remember to run formatter.
```

本质还是模型遵循 instruction。

而 hook/automation 可以把要求变成：

**deterministic execution**

A 是典型：

> instruction 不够可靠 → 把 instruction 写得更多。

不一定解决根本问题。

**Key exam takeaway:**

> **Guidance tells Claude what should happen; hooks can make known lifecycle actions happen deterministically.**

---

## Question 3 — Hook Scope

AtlasPay considers adding a hook that runs the complete 45-minute integration-test suite after **every individual file edit**.

What is the BEST assessment?

**A.** Excellent; every possible validation should run after every event.  
**B.** The hook may be attached at the wrong granularity; choose an event and validation scope appropriate to the cost and purpose of the check.  
**C.** Hooks cannot run tests.  
**D.** Replace all testing with Claude self-review.  

### Correct Answer: **B**

**English Explanation:**
Deterministic automation still needs appropriate granularity. Expensive validation attached to excessively frequent events can make the workflow inefficient without providing proportional value.

### 中文解说

这题防止另一个极端：

> Hook 好，所以越多越好。

不是。

假设 Claude 修改：

```text
File 1
→ 45 min tests

File 2
→ 45 min tests

File 3
→ 45 min tests
```

这会严重降低效率。

更合理可能是：

```text
Per edit
→ lightweight formatter/linter

After meaningful implementation
→ unit tests

Before completion / PR
→ integration suite
```

所以 Hook 也要考虑：

**event granularity**

和：

**cost**

**Key exam takeaway:**

> **Deterministic does not mean indiscriminate. Match the safeguard to the lifecycle event.**

---

## Question 4 — Hard Safeguards

### Choose TWO.

AtlasPay wants to prevent Claude Code from accidentally performing destructive database operations during routine development.

Which TWO controls provide the strongest layered protection?

**A.** Tell Claude not to run destructive commands unless explicitly required.  
**B.** Restrict the credentials/environment so routine Claude Code sessions lack unnecessary destructive database permissions.  
**C.** Give Claude full database administrator access so it can recover from mistakes.  
**D.** Treat database permissions as unnecessary because Claude normally follows instructions.  

### Correct Answers: **A and B**

**English Explanation:**
Behavioral guidance reduces inappropriate attempts, while least-privilege permissions limit what can actually execute. Together they provide defense in depth.

### 中文解说

这是：

**Guidance + Enforcement**

第一层：

```text
Instruction
→ Don't perform destructive DB operations.
```

第二层：

```text
Permission boundary
→ Routine session cannot perform them anyway.
```

如果第一层偶尔失败，第二层仍然保护系统。

这就是：

**Defense in Depth**

B 单独比 A 更强，但题目问：

> **Choose TWO / layered protection**

所以 A + B。

C 完全违反：

**Least Privilege**

**Key exam takeaway:**

> **Prompt guidance reduces bad requests; capability restrictions reduce their blast radius.**

---

## Question 5 — Validation Feedback

Claude changes payment-calculation code.

Tests return:

```text
Expected: 105.00
Actual:   100.00
Failure: tax was not applied
```

Which next step is MOST effective?

**A.** Retry the original implementation request without including the test result.  
**B.** Provide the concrete failing test result to Claude and ask it to diagnose and correct the implementation.  
**C.** Delete the failing test.  
**D.** Increase context by adding unrelated source files.  

### Correct Answer: **B**

**English Explanation:**
Specific verification feedback gives Claude actionable evidence about what failed and supports a targeted correction loop.

### 中文解说

继续强化：

**Feedback-driven iteration**

```text
Implement
   ↓
Test
   ↓
FAIL:
tax not applied
   ↓
Give exact failure to Claude
   ↓
Correct
   ↓
Retest
```

A 是：

**blind retry**

没有增加任何新 evidence。

C 属于：

> 为了让测试绿，把测试删掉。

D 只是增加 context noise。

**Key exam takeaway:**

> **Concrete execution feedback is more useful than generic retry instructions.**

---

## Question 6 — Permission Prompt vs Real Authorization

A developer configures Claude Code so that a sensitive shell command requires interactive confirmation.

Another engineer concludes:

> “This confirmation is now our production authorization mechanism.”

What is the BEST assessment?

**A.** Correct; an interactive model/tool permission prompt replaces application authorization.  
**B.** Incorrect; development-time tool permissions and production business authorization solve different problems.  
**C.** Correct if `CLAUDE.md` also documents the rule.  
**D.** Correct if the model uses Plan Mode first.  

### Correct Answer: **B**

**English Explanation:**
Claude Code permission controls can constrain development tool use, but production authorization must still be enforced by the system responsible for the protected business action.

### 中文解说

这里非常容易把两个“permission”混淆。

### Claude Code permission

主要是：

> 当前 coding agent 能不能执行某个开发环境操作？

### Production authorization

例如：

> Alice 是否有权批准 $5M payment？

这是业务系统的 security boundary。

两者不是同一个东西。

```text
Claude Code permission
≠
Business authorization
```

Plan Mode 更与 authorization 没直接关系。

**Key exam takeaway:**

> **Do not confuse agent/tool permissions with business authorization.**

---

## Question 7 — Plan Mode

AtlasPay must replace a legacy settlement component.

The change affects:

* 60 files;
* two databases;
* several API contracts;
* deployment configuration;
* backward compatibility.

Several migration strategies are possible.

What should Claude Code do FIRST?

**A.** Begin editing the first matching source file.  
**B.** Use Plan Mode to investigate dependencies and develop a migration approach before implementation.  
**C.** Increase temperature.  
**D.** Run the formatter.  

### Correct Answer: **B**

**English Explanation:**
A large, cross-cutting, ambiguous change benefits from explicit exploration and planning before edits begin.

### 中文解说

看到：

```text
60 files
databases
API contracts
deployment
backward compatibility
several strategies
```

就是非常典型的：

**Plan first**

判断标准：

```text
Small/local/obvious
→ Direct implementation may be fine

Large/cross-cutting/ambiguous
→ Plan Mode
```

注意 Plan Mode 不是因为：

> “任务很重要”。

而是因为：

> **需要先理解依赖和选择策略。**

**Key exam takeaway:**

> **Complexity + cross-cutting dependencies + strategic ambiguity → Plan Mode.**

---

## Question 8 — Subagent Context

Claude delegates a database-migration review to a specialist subagent.

The main session contains:

* UI discussions;
* old deployment logs;
* unrelated debugging history;
* the migration diff;
* database compatibility requirements;
* migration safety rules.

What context should the subagent receive?

**A.** The entire session because maximum context always produces maximum accuracy.  
**B.** Only the migration filename.  
**C.** The migration diff plus the compatibility and safety context necessary to perform the bounded review.  
**D.** No project information.  

### Correct Answer: **C**

**English Explanation:**
A specialist should receive the **minimum sufficient context** required for its task: enough to make the correct judgment without unnecessary conversational noise.

### 中文解说

重点词：

> **Minimum sufficient context**

不是：

**minimum context**

也不是：

**maximum context**

这里需要：

```text
Migration diff
+
DB compatibility requirements
+
Migration safety rules
```

不需要：

```text
UI discussion
old unrelated debugging
```

B 太少，无法判断 migration 是否安全。

A 太多，会增加：

**context pollution**

**Key exam takeaway:**

> **Give specialists relevant context, not all available context.**

---

## Question 9 — Claude Code Workflow Design

### Choose TWO.

AtlasPay performs the same code-review procedure for every release:

1. inspect changed files;
2. check security-sensitive modifications;
3. run relevant tests;
4. produce a standard review report.

The procedure is invoked on demand.

Which TWO design choices are MOST appropriate?

**A.** Package the repeatable procedure as a reusable Skill/workflow.  
**B.** Use deterministic test/lint automation where objective verification is available.  
**C.** Depend entirely on one developer remembering the procedure.  
**D.** Put every possible review result permanently into `CLAUDE.md`.  

### Correct Answers: **A and B**

**English Explanation:**
A Skill is appropriate for a reusable on-demand procedure, while deterministic checks should remain deterministic where possible. The two approaches complement each other.

### 中文解说

这题把两个概念组合：

### Skill

负责：

> **可重复调用的 procedure**

例如：

```text
/release-review
```

里面定义 review workflow。

### Deterministic automation

负责：

```text
Tests
Lint
Formatting
Static checks
```

这些不需要 Claude “凭感觉判断”。

所以：

```text
Skill
→ orchestrates procedure

Tests/hooks/scripts
→ objectively verify deterministic conditions
```

D 是常见错误：

> 什么东西都往 `CLAUDE.md` 塞。

记住：

```text
Always-relevant project guidance
→ CLAUDE.md

On-demand repeatable procedure
→ Skill

Lifecycle-triggered deterministic action
→ Hook / automation
```

这是今天最值得背的三分法。

---

## Question 10 — Failure-Layer Diagnosis

AtlasPay observes:

> Claude selected the correct migration command.
> The command itself was valid.
> Project instructions clearly said production databases must not be modified.
> Nevertheless, the development environment credentials allowed the command to modify production successfully.

Which layer MOST directly failed?

**A.** Plan Mode  
**B.** Coding style guidance  
**C.** Capability/permission boundary  
**D.** Few-shot prompting  

### Correct Answer: **C**

**English Explanation:**
The system relied on behavioral guidance while leaving the dangerous capability available. The strongest direct fix is to remove unnecessary production permissions from the development execution environment.

### 中文解说

逐层看：

```text
Instruction:
Don't modify production.
✓ 已经写了

Claude:
仍然尝试执行
✗

Environment:
竟然允许成功
✗✗
```

真正的 hard boundary 应该是：

```text
Development Claude Code
       ↓
Production DB?
       ↓
Permission denied
```

而不是：

> “Claude 一般会记住不要碰。”

这就是：

**Least Privilege + Enforcement**

---

# Day 13 — Claude Code Control Map

今天最重要的是区分这四种机制：

| Requirement                                    | Think first                           |
| ---------------------------------------------- | ------------------------------------- |
| Persistent project guidance                    | **`CLAUDE.md` / rules**               |
| Repeatable on-demand procedure                 | **Skill**                             |
| Deterministic action tied to a lifecycle event | **Hook / automation**                 |
| Dangerous capability must be impossible        | **Permission / enforcement boundary** |

可以记成：

```text
KNOW
→ CLAUDE.md

DO ON REQUEST
→ Skill

DO WHEN EVENT OCCURS
→ Hook

MUST NOT BE ABLE TO DO
→ Permission boundary
```

## Common Trap #1 — Hook vs `CLAUDE.md`

> “Always format modified files.”

如果只是作为 coding guidance：

**`CLAUDE.md`**

如果题目强调：

> **must reliably happen after the relevant event**

优先考虑：

**Hook / deterministic automation**

## Common Trap #2 — Hook vs Skill

> “Run our complete security-review workflow when I request it.”

→ **Skill**

而：

> “Whenever a file is modified, automatically run a formatter.”

→ **Hook**

Skill 是：

**on-demand procedure**

Hook 是：

**event-driven action**

## Common Trap #3 — Permission vs Instruction

> “Claude must not modify production.”

只写：

```text
DO NOT MODIFY PRODUCTION.
```

仍然只是 behavioral control。

更强的是：

```text
Development credential
→ production write permission = NONE
```

### Eight sentences to memorize

> **Project instructions guide persistent behavior.**

> **Skills package reusable on-demand workflows.**

> **Hooks automate actions around known lifecycle events.**

> **Hooks should be scoped to appropriate events and costs.**

> **Permission boundaries enforce what the agent can actually do.**

> **Minimum sufficient context is better than maximum context.**

> **Specific execution feedback improves iterative correction.**

> **Behavioral guidance and deterministic safeguards are complementary, not interchangeable.**
