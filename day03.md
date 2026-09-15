# CCAR-F Daily Practice — Day 3

Today’s primary focus is **Domain 3: Claude Code Configuration & Workflows**, with cross-domain questions on orchestration and reliability. Current Anthropic material continues to emphasize `CLAUDE.md`, Plan Mode, skills, subagents, and structured workflows as core Claude Code concepts. ([Anthropic][1])

**Exam mode:** 8 questions, target **16 minutes**. Try answering each question before reading its explanation.

---

## Scenario: Atlas Payments Platform

**Atlas Financial is migrating a large payment platform to Claude Code.**

The repository contains:

```text
atlas-payments/
├── frontend/
│   └── React / TypeScript
├── backend/
│   └── Java / Spring Boot
├── database/
│   └── SQL migrations
├── tests/
└── docs/
```

The engineering team wants Claude Code to follow organization-wide conventions while also respecting different rules for frontend, backend, and database code.

Developers also use Claude Code for large refactorings, CI automation, debugging, and repeated code-review workflows.

---

## Question 1 — `CLAUDE.md`

Every developer working in the repository should have Claude Code follow these instructions:

* Run tests before completing a change.
* Never commit generated build artifacts.
* Use the repository's standard build commands.
* Follow the project's architectural conventions.

Where should these instructions MOST appropriately be maintained?

**A.** Repeat them in every developer prompt.
**B.** Store them in the project's `CLAUDE.md`.
**C.** Put them only in one developer's personal configuration.
**D.** Add them to every source file as comments.

### Correct Answer: **B**

**English Explanation:**
`CLAUDE.md` is appropriate for persistent project-level instructions that should guide Claude Code across sessions and team members. It provides project context without requiring developers to repeat the same instructions for every task.

### 中文解说

题眼是：

> **Every developer**
> **repository**
> **follow these instructions**

也就是说，这是：

**项目级、长期有效、反复需要的规则。**

因此选：

**Project `CLAUDE.md`**

A 的问题是重复、容易遗漏。

C 只对一个开发者有效，不适合作为团队项目规范。

D 把 Agent instruction 混进业务代码，也不是合理的管理方式。

可以这样记：

> **Persistent project knowledge → `CLAUDE.md`**

Anthropic 近期的 Claude Code Foundations 材料也把 `claude.md` 描述为给 Claude Code 提供项目 briefing 的核心机制。 ([Anthropic][2])

---

## Question 2 — Path-Specific Rules

Atlas has different requirements:

```text
frontend/**
→ Use React conventions.

backend/**
→ Follow Spring Boot service-layer conventions.

database/**
→ Never modify an applied migration.
```

The team does **not** want all three sets of instructions loaded indiscriminately for every file.

What is the BEST design?

**A.** Put every rule into one extremely large `CLAUDE.md`.
**B.** Use path-specific rules associated with the relevant areas of the repository.
**C.** Ask developers to manually paste the correct rules before each task.
**D.** Create three separate repositories.

### Correct Answer: **B**

**English Explanation:**
Path-specific rules allow instructions to apply where they are relevant. This avoids unnecessarily loading unrelated frontend, backend, or database guidance for every task.

### 中文解说

这是：

**Global rule vs conditional rule**

的区别。

如果规则是：

> “所有代码修改完成后都要运行测试”

适合项目级规则。

但：

> “只有修改 database migration 时才执行这个规则”

则应该使用：

**Path-specific rules**

A 是 CCAR-F 很喜欢的陷阱：

> “既然 `CLAUDE.md` 很好，那就什么都塞进去。”

并不是。

**考试速记：**

> **Always relevant → `CLAUDE.md`**
> **Path dependent → path-specific rules**

---

## Question 3 — Plan Mode

Atlas needs to replace its legacy authentication architecture.

The change will affect:

* 47 source files,
* authentication middleware,
* database schema,
* API contracts,
* integration tests,
* deployment configuration.

Several implementation strategies are possible.

What should the developer do FIRST?

**A.** Ask Claude Code to immediately modify all affected files.
**B.** Use Plan Mode to explore the codebase and develop an implementation strategy before making changes.
**C.** Ask Claude to change one random authentication file and infer the architecture afterward.
**D.** Increase the model temperature before implementation.

### Correct Answer: **B**

**English Explanation:**
Plan Mode is well suited to complex, cross-cutting changes where understanding dependencies and agreeing on an approach before execution reduces risk.

### 中文解说

看到这些词：

> **47 files**
> **architecture**
> **database schema**
> **API contracts**
> **several implementation strategies**

基本就应该想到：

**Plan first → Execute later**

Anthropic 当前 Claude Code Foundations 也明确描述了 Plan Mode 的用途：在执行之前先审查策略。 ([Anthropic][2])

A 是典型的：

**Act before understanding**

C 更明显不合理。

D 与架构分析没有直接关系。

**考试速记：**

> **Large / architectural / cross-file / ambiguous → Plan Mode**

而：

```text
Rename one local variable.
```

通常没必要先做完整 Plan。

---

## Question 4 — Reusable Workflow

Every pull request requires the same review process:

1. Inspect changed files.
2. Check security-sensitive code.
3. Run relevant tests.
4. Verify coding standards.
5. Produce a standardized review summary.

The team wants developers to invoke this process repeatedly without rewriting the full instructions.

Which approach is BEST?

**A.** Create a reusable Skill/workflow containing the review procedure.
**B.** Add the entire procedure to every developer's prompt manually.
**C.** Put the procedure in comments inside every changed file.
**D.** Depend on Claude to remember the procedure from previous unrelated sessions.

### Correct Answer: **A**

**English Explanation:**
A reusable skill is appropriate for an on-demand, repeatable workflow. It packages procedural knowledge so developers can invoke the same standardized process consistently.

### 中文解说

这里和 Question 1 有一点微妙区别。

Question 1 是：

> Claude **一直都应该知道/遵守**什么。

所以：

**CLAUDE.md**

这里是：

> 有一个特定流程，需要时重复执行。

所以：

**Skill / reusable workflow**

记住：

```text
Always relevant
→ CLAUDE.md

Path dependent
→ Rules

On-demand repeatable procedure
→ Skill
```

这三个非常容易放在一起考。

Anthropic 当前 Claude Code 基础材料也把 skills、plugins、subagents 与团队标准化实践放在同一个配置体系中。 ([Anthropic][2])

---

## Question 5 — Iterative Refinement

Claude Code implements a backend change. The code compiles, but three unit tests fail.

What is the BEST next step?

**A.** Discard the entire session and start over without showing Claude the failures.
**B.** Provide the failing test output and ask Claude to diagnose and refine the implementation.
**C.** Tell Claude only that “something is wrong.”
**D.** Disable the failing tests so the implementation can be accepted.

### Correct Answer: **B**

**English Explanation:**
Concrete verification feedback gives Claude evidence it can use to diagnose and improve its implementation. Iterative coding workflows are strongest when the agent can observe test, compiler, or runtime results and refine its work.

### 中文解说

这里考的是：

**Act → Observe → Refine**

正确循环：

```text
Implement
   ↓
Run tests
   ↓
Observe failures
   ↓
Give concrete feedback
   ↓
Fix
   ↓
Run tests again
```

A 把非常有价值的失败信息丢掉。

C 的反馈：

> something is wrong

信息量太低。

D 属于典型的：

**让验证通过，而不是让代码正确。**

Anthropic 对 Claude Code 的介绍强调其可以探索代码、修改、运行测试、调试并持续迭代；近期对真实使用情况的研究也把通过测试等可验证证据视为成功的重要信号。 ([Anthropic][1])

**考试速记：**

> **Specific feedback > vague retry**

---

## Question 6 — CI/CD

Atlas wants its CI pipeline to automatically ask Claude Code to review a pull request and then terminate without entering an interactive terminal session.

Which execution style is MOST appropriate?

**A.** Launch a normal interactive Claude Code session and wait for a human.
**B.** Use Claude Code's non-interactive/headless execution mode suitable for automation.
**C.** Store the review request in `CLAUDE.md` and assume CI will automatically execute it.
**D.** Use Plan Mode only, because CI systems cannot run Claude actions.

### Correct Answer: **B**

**English Explanation:**
CI/CD requires non-interactive execution: the task should be supplied programmatically, produce an output or status, and terminate so the surrounding automation can continue.

### 中文解说

题眼：

> **CI pipeline**
> **automatically**
> **without interactive terminal**

所以一定是：

**non-interactive / headless execution**

不要把：

`CLAUDE.md`

理解成“任务调度器”。

它负责提供 context/instructions，不会因为写了一条规则就自动执行 CI job。

同样：

Plan Mode

是一种工作模式，也不是 CI 的替代品。

**考试速记：**

> **Human terminal session → Interactive**
> **CI / script / automation → Non-interactive**

---

## Question 7 — Subagent Context

Atlas creates a security-review subagent.

The main Claude Code session contains:

* UI design discussions,
* deployment logs,
* database migration history,
* 25 previous debugging conversations,
* the authentication diff that must be reviewed.

What context should the security subagent receive?

**A.** The complete session because more context always improves accuracy.
**B.** Only the authentication diff, even if supporting security rules are required.
**C.** The authentication diff plus the relevant security policies and architectural context needed for the review.
**D.** No repository context; the subagent should rely entirely on general security knowledge.

### Correct Answer: **C**

**English Explanation:**
A subagent should receive the minimum sufficient context for its bounded task: enough information to perform the review correctly, without unrelated history that increases noise and context consumption.

### 中文解说

这题故意让 B 和 C 都看起来合理。

原则不是：

> **Minimum context**

而更准确地说是：

> **Minimum sufficient context**

B 太少。

只有 diff，没有项目 security policy，可能无法判断：

> 这个修改是否违反 Atlas 自己的安全规则。

A 又太多。

25 次 debugging conversation 和 UI design 对 security review 没有必要。

所以：

```text
Relevant code
+
Relevant policy
+
Necessary architecture context
=
Minimum sufficient context
```

**考试陷阱：**

> “More context is always better.”

通常是错的。

---

## Question 8 — Long-Running Coding Loop

Atlas runs an autonomous Claude Code task to migrate hundreds of API calls.

The team discovers that the task can continue making changes for a long time without clearly determining whether migration is complete.

Which design BEST improves reliability?

**A.** Tell Claude, “Keep working until everything looks good.”
**B.** Define explicit completion criteria, a bounded budget, verification checkpoints, and a stopping condition.
**C.** Remove all verification steps because they consume tokens.
**D.** Allow unlimited iterations because difficult migrations cannot have predefined safeguards.

### Correct Answer: **B**

**English Explanation:**
Long-running agentic coding needs an explicit definition of done, bounded execution, checkpoints, and verification. These controls make progress observable and prevent uncontrolled loops.

### 中文解说

这是非常值得注意的一道当前实践题。

一个好的长时间 Agent loop 应该知道：

```text
What does DONE mean?
        +
What is the budget?
        +
Where do we verify?
        +
When must we stop?
```

而不是：

> Keep working until it looks good.

Anthropic 在 2026 年 7 月关于 Claude Code loops 的材料中也明确强调了：**define what done means, set a budget, add a checkpoint**，以及处理无法停止的 loop。 ([Anthropic][3])

A 的问题是完成标准主观。

C 去掉 verification 会降低可靠性。

D 把“复杂任务”错误地等同于“无限执行”。

**考试速记：**

> **Autonomy does not mean unbounded execution.**

---

# Day 3 — Exam Memory Sheet

今天最重要的是把下面四组概念区分清楚：

| Requirement                                 | Best fit                                                      |
| ------------------------------------------- | ------------------------------------------------------------- |
| Persistent project-wide guidance            | **`CLAUDE.md`**                                               |
| Instructions relevant only to certain files | **Path-specific rules**                                       |
| Repeatable on-demand procedure              | **Skill / reusable workflow**                                 |
| Complex cross-file change                   | **Plan Mode first**                                           |
| Test/compiler failure                       | **Feed concrete feedback back into the loop**                 |
| CI/CD automation                            | **Non-interactive execution**                                 |
| Specialist subagent                         | **Minimum sufficient context**                                |
| Long autonomous task                        | **Definition of done + budget + checkpoint + stop condition** |

### Today's Most Important Trap

Suppose the exam asks:

> The team has a standard security review procedure that developers run **when requested**.

Do **not** automatically choose `CLAUDE.md` just because it is a team standard.

Ask:

> **Should Claude always carry this information, or is it an on-demand procedure?**

If it is always relevant:

**`CLAUDE.md`**

If it applies only to certain paths:

**Path-specific rule**

If it is a reusable procedure invoked when needed:

**Skill**

If it is a complicated one-time architectural change:

**Plan Mode**

That distinction is exactly the kind of scenario reasoning CCAR-F is designed to test, rather than simple terminology memorization. Anthropic’s current Claude Code training materials continue to frame the product around the agentic cycle of **read → plan → act → observe**, with project instructions, Plan Mode, skills, and subagents supporting that workflow. ([Anthropic][2])

[1]: https://www.anthropic.com/webinars/claude-code-in-an-hour-a-developers-intro?utm_source=chatgpt.com "Claude Code in an Hour: A Developer's Intro | Webinars \ Anthropic"
[2]: https://www.anthropic.com/webinars/claude-code-foundations?utm_source=chatgpt.com "Claude Code: Foundations | Webinars \ Anthropic"
[3]: https://www.anthropic.com/webinars/startup-builds-getting-started-with-loops?utm_source=chatgpt.com "Startup Builds: Getting Started with Loops | Webinars \ Anthropic"
