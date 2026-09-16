# CCAR-F Daily Practice — Day 12

Today’s set is a **hard mixed-domain set** with extra emphasis on **MCP architecture, tool/resource boundaries, permissions, structured errors, orchestration, and production safety**. Several questions intentionally contain more than one reasonable-looking answer; choose the **BEST** answer for the layer actually failing.

**Exam mode:** 10 questions. Target time: **20 minutes**. Questions 4 and 8 are **Choose TWO**.

---

## Scenario: Horizon Enterprise Knowledge Agent

**Horizon Technologies is building a Claude-based enterprise assistant used by engineering, finance, and support teams.**

Claude accesses internal systems through MCP servers and application tools.

The environment exposes:

```text
Engineering MCP Server
├── source repositories
├── technical documentation
├── search_code
└── create_bug

Finance MCP Server
├── financial policies
├── expense records
├── search_expenses
└── submit_reimbursement

Support MCP Server
├── customer documentation
├── customer records
├── search_tickets
└── update_ticket
```

The assistant must answer questions, investigate problems, and perform authorized actions while preventing accidental or unauthorized changes.

---

## Question 1 — MCP Resource vs Tool

The Engineering MCP server must expose:

1. the company's coding standards document;
2. an operation that creates a new bug in the issue tracker.

How should these MOST naturally be represented?

**A.** Both as Resources
**B.** Both as Tools
**C.** Coding standards → Resource; `create_bug` → Tool
**D.** Coding standards → Tool; `create_bug` → Resource

### Correct Answer: **C**

**English Explanation:**
Resources naturally expose information or content for consumption, while tools represent callable operations. A coding-standards document is reference information; creating a bug is an action.

### 中文解说

这是 MCP 的基础判断，但真实考试往往会包装成 Scenario。

可以先问：

> **Claude 是“读取东西”，还是“执行动作”？**

如果是：

```text
Policy
Documentation
Reference data
```

首先想到：

**Resource**

如果是：

```text
Create
Update
Search operation
Send
Execute
```

首先想到：

**Tool**

所以：

```text
Coding standards
→ Resource

create_bug
→ Tool
```

A 的问题是 `create_bug` 会改变外部状态。

D 正好反了。

**Key exam takeaway:**

> **Resource = information to consume.**
> **Tool = operation to invoke.**

---

## Question 2 — MCP Does Not Replace Authorization

Horizon connects Claude to the Finance system through MCP.

An engineer argues:

> “Because MCP provides a standardized interface, we no longer need separate authorization checks for `submit_reimbursement`.”

What is the BEST response?

**A.** Correct; MCP automatically guarantees authorization for every exposed operation.
**B.** Incorrect; MCP standardizes integration, but application/system authorization must still protect sensitive operations.
**C.** Correct, provided the MCP server uses detailed tool descriptions.
**D.** Correct if Claude's confidence exceeds 95%.

### Correct Answer: **B**

**English Explanation:**
MCP addresses standardized connectivity between AI applications and external capabilities. It does not eliminate the need for authentication, authorization, validation, and other security controls.

### 中文解说

这里一定要防止把 MCP “万能化”。

MCP 主要解决：

> **AI application 如何用标准方式连接外部 capability？**

但它不等于：

```text
MCP = Authentication
MCP = Authorization
MCP = Business validation
MCP = Safety guarantee
```

例如：

```text
Claude
  ↓
MCP
  ↓
submit_reimbursement
  ↓
Authorization Check
  ↓
Execute / Reject
```

Authorization 仍然必须存在。

C 又把：

**Tool description**

和：

**Security enforcement**

混在一起。

D 的 confidence 与权限没有关系。

**Key exam takeaway:**

> **Integration standardization does not replace security enforcement.**

---

## Question 3 — Tool Schema Design

Horizon exposes this tool:

```text
search_expenses(
    query: string,
    employee_id: string,
    date_range: string,
    minimum_amount: string,
    maximum_amount: string,
    output_mode: string,
    operation_type: string
)
```

Most searches need only an employee ID and optional date range. Claude frequently supplies malformed amounts and irrelevant `operation_type` values.

What is the BEST improvement?

**A.** Add more parameters so Claude has greater flexibility.
**B.** Simplify the schema, use appropriate types, and remove parameters unrelated to the tool's actual responsibility.
**C.** Ask Claude to ignore unnecessary parameters.
**D.** Increase temperature.

### Correct Answer: **B**

**English Explanation:**
Tool schemas should be focused, semantically clear, and typed appropriately. Unnecessary or ambiguous parameters increase opportunities for incorrect tool calls.

### 中文解说

Tool Design 不只是：

> description 写得好不好。

还包括：

**Schema 本身是否合理。**

现在的问题：

```text
minimum_amount: string
maximum_amount: string
operation_type: string
```

比如金额本来应该是 numeric type，却设计成任意 string。

而且 `operation_type` 如果这个 Tool 只是 search expense，很可能根本不需要。

更合理：

```text
search_expenses(
    employee_id,
    start_date?,
    end_date?
)
```

如果确实需要 amount filter，再使用明确的 numeric type。

A 会进一步增加复杂度。

C 又是在用 Prompt 补偿 API Design 问题。

**Key exam takeaway:**

> **Make invalid tool calls difficult by designing narrow, well-typed schemas.**

---

## Question 4 — MCP Security

### Choose TWO.

`SupportAgent` only needs to read customer records and search support tickets.

Which TWO design choices BEST follow secure capability design?

**A.** Give the agent only the read/search capabilities needed for its task.
**B.** Also expose `update_ticket` because Claude may find it convenient later.
**C.** Treat retrieved customer text as potentially untrusted content rather than authoritative agent instructions.
**D.** Allow customer records to override system instructions if they contain the phrase “SYSTEM MESSAGE.”

### Correct Answers: **A and C**

**English Explanation:**
Least privilege limits the consequences of mistakes, while trust-boundary handling prevents retrieved data from gaining inappropriate instruction authority.

### 中文解说

这里组合了两个高频安全原则：

### 1. Least Privilege

SupportAgent 只需要：

```text
read
search
```

就不要顺便给：

```text
update
delete
admin
```

### 2. Trust Boundary

客户记录可能写：

> Ignore all previous instructions...

这仍然只是：

**customer data**

不是：

**trusted system instruction**

A 限制 capability。

C 限制 instruction authority。

两层结合就是：

**Defense in Depth**

B 的：

> “以后可能方便”

不是授予危险权限的理由。

D 是典型 indirect prompt-injection vulnerability。

**Key exam takeaway:**

> **Limit both what an agent can believe as instruction and what it can actually do.**

---

## Question 5 — Structured Tool Errors

`search_tickets` currently returns:

```text
Search failed.
```

Possible underlying causes include:

```text
rate_limit
invalid_customer_id
service_timeout
access_denied
```

What is the BEST improvement?

**A.** Return a longer natural-language apology.
**B.** Return structured error information that distinguishes failure type and recovery-relevant properties.
**C.** Automatically retry every error ten times.
**D.** Hide all errors from Claude.

### Correct Answer: **B**

**English Explanation:**
Different failures require different recovery strategies. Structured errors allow the agent or coordinator to distinguish retryable failures from invalid input or authorization failures.

### 中文解说

四种 error 的 recovery 完全不同：

```text
rate_limit
→ wait / retry

service_timeout
→ bounded retry

invalid_customer_id
→ correct input

access_denied
→ permission / authorization
```

如果全部变成：

> Search failed.

Agent 就无法选择 recovery。

这就是为什么 Tool Error 最好包含：

```text
error_type
retryable
retry_after
message
relevant context
```

C 是典型：

> **所有 error 都 retry**

错误。

比如 `access_denied` retry 100 次也不会变成 authorized。

**Key exam takeaway:**

> **Error semantics should support recovery decisions.**

---

## Question 6 — Resource Freshness

Claude retrieves:

> `finance-policy-v17`

from an MCP resource and uses it to approve an expense.

Minutes earlier, Finance published:

> `finance-policy-v18`

which changed the approval threshold.

What is the MOST direct architectural problem?

**A.** Tool description ambiguity
**B.** Resource freshness/version management
**C.** Few-shot prompting
**D.** Agent parallelization

### Correct Answer: **B**

**English Explanation:**
The model reasoned over stale authoritative information. Policies that affect decisions need freshness, versioning, cache invalidation, or explicit retrieval of the currently effective version.

### 中文解说

这里 Claude reasoning 可能完全没错。

真正错误是：

```text
Correct reasoning
+
Old policy
=
Wrong decision
```

所以不能看到错误 decision 就自动：

> Improve prompt.

应该检查：

**Data freshness**

可以保存：

```text
policy_version
effective_date
retrieved_at
```

并在执行高风险决策前确认：

> 当前还是 effective version 吗？

**Key exam takeaway:**

> **Model correctness cannot compensate for stale authoritative data.**

---

## Question 7 — Fixed Workflow vs Agent Decision

Every reimbursement must perform:

```text
1. Validate employee identity
2. Check receipt presence
3. Check reimbursement limit
4. Submit reimbursement
```

Steps 1–3 have deterministic rules.

Which design is BEST?

**A.** Ask Claude to reason through all four steps and decide whether each one is necessary.
**B.** Enforce steps 1–3 deterministically and permit submission only after their required conditions succeed.
**C.** Ask four agents to vote on whether the reimbursement should proceed.
**D.** Put the rules only in the tool description.

### Correct Answer: **B**

**English Explanation:**
Known deterministic requirements should be implemented deterministically rather than delegated to probabilistic model judgment.

### 中文解说

再强化一个核心 CCAR-F 原则：

> **不要把所有东西都 Agent 化。**

例如：

```text
receipt_present == true
amount <= limit
employee_active == true
```

这些规则已经完全明确。

直接 Code：

```text
if conditions_met:
    allow_submission
else:
    reject
```

Claude 更适合处理：

* 非结构化 receipt；
* ambiguous text；
* dynamic investigation；
* summarization；
* judgment where rules are not exact。

**Key exam takeaway:**

> **Use models for ambiguity; use code for exact rules.**

---

## Question 8 — Tool Execution Safety

### Choose TWO.

`submit_reimbursement` changes financial state.

A network timeout occurs immediately after the request is sent.

Which TWO concerns are MOST important before retrying?

**A.** Whether the operation supports idempotency or a stable request key.
**B.** Whether the resulting external state can be checked to determine whether the first request succeeded.
**C.** Whether Claude can generate a longer explanation.
**D.** Whether the tool description contains at least 200 words.

### Correct Answers: **A and B**

**English Explanation:**
A timeout does not prove that a state-changing operation failed. Idempotency and state reconciliation help prevent duplicate side effects when the outcome of the first request is uncertain.

### 中文解说

这是高频生产系统陷阱：

```text
Request
   ↓
Server executes successfully
   ↓
Response lost
   ↓
Client sees TIMEOUT
```

所以：

> **Timeout ≠ Operation failed**

如果直接 retry：

```text
submit reimbursement × 2
```

可能造成重复付款。

两个重要解决方案：

### Idempotency

```text
request_id = ABC123
```

重复请求仍然代表同一业务操作。

### State Reconciliation

先查询：

> reimbursement 已经创建了吗？

再决定是否 retry。

**Key exam takeaway:**

> **Ambiguous failure + side effect → reconcile state or use idempotency before retrying.**

---

## Question 9 — MCP Server Boundary

Horizon considers creating one enormous MCP server exposing:

* engineering repositories;
* payroll records;
* customer data;
* finance actions;
* production administration;
* HR records.

Every agent would connect to the same server and rely on prompts to avoid inappropriate capabilities.

What is the STRONGEST concern?

**A.** MCP servers cannot expose more than five tools.
**B.** The design creates overly broad trust and capability boundaries, making least-privilege enforcement more difficult.
**C.** Claude cannot use tools from different business domains.
**D.** MCP requires one server per individual tool.

### Correct Answer: **B**

**English Explanation:**
Capability boundaries should reflect security and organizational requirements. An overly broad server can make permission scoping, trust separation, and blast-radius control harder.

### 中文解说

这里不是说：

> “一个 MCP Server 只能放一个业务。”

而是在考：

**Security Boundary**

如果：

```text
SupportAgent
```

连接后天然就能看到：

```text
payroll
production admin
finance actions
HR
```

风险非常大。

更合理的 boundary 可能按照：

* trust domain；
* permission；
* sensitivity；
* business responsibility；

进行划分。

A、D 都是人为编出来的绝对限制。

**Key exam takeaway:**

> **Design MCP boundaries around trust and capability, not convenience alone.**

---

## Question 10 — Failure-Layer Diagnosis

Horizon observes:

> Claude selected the correct `submit_reimbursement` tool.
> The arguments matched the schema.
> The employee was authorized.
> The amount was within the limit.
> The operation executed successfully.
> However, the final report says the reimbursement was based on policy v18 when the decision actually used policy v17.

Which layer MOST directly failed?

**A.** Tool selection
**B.** Authorization
**C.** Provenance/audit-state tracking
**D.** Tool schema design

### Correct Answer: **C**

**English Explanation:**
Execution itself was valid, but the system recorded the wrong policy basis for the decision. Provenance and audit-state tracking should preserve which evidence and policy version actually supported the action.

### 中文解说

继续使用 failure-layer 排除法：

```text
Tool selection       ✓
Arguments            ✓
Authorization        ✓
Validation           ✓
Execution            ✓

Recorded evidence
/policy provenance   ✗
```

所以问题是：

**Provenance / Audit State**

系统应该记录类似：

```text
decision_id
policy_version_used
evidence_ids
approval_id
tool_request_id
execution_result
timestamp
```

而不是事后让 Claude：

> “你觉得刚才用了哪个 policy？”

---

# Day 12 — MCP Decision Map

今天把 MCP 相关概念集中整理一下：

| Scenario                                 | Think first                              |
| ---------------------------------------- | ---------------------------------------- |
| Static/reference information             | **Resource**                             |
| Callable operation                       | **Tool**                                 |
| Standardized AI-system integration       | **MCP**                                  |
| Sensitive operation                      | **Authorization still required**         |
| Agent sees capabilities it does not need | **Least privilege**                      |
| Huge generic tool                        | **Narrower tool boundary/schema**        |
| External retrieved instructions          | **Untrusted content / prompt injection** |
| Old policy returned                      | **Freshness/versioning**                 |
| Different errors need different recovery | **Structured error semantics**           |
| Financial write timed out                | **Idempotency/state reconciliation**     |
| One MCP boundary exposes everything      | **Trust/capability segmentation**        |

## Today's Important Distinction

Three concepts are easy to confuse:

> **Authentication** — Who are you?

> **Authorization** — Are you allowed to do this?

> **Validation** — Is this requested action/data acceptable?

For example:

```text
Alice is logged in.
→ Authentication ✓

Alice may submit reimbursement.
→ Authorization ✓

Requested amount = -$500.
→ Validation ✗
```

Being authenticated does not automatically mean being authorized.

Being authorized does not mean every requested parameter is valid.

## Day 12 Common Trap

> **“The data came through MCP, therefore it is trusted.”**

Wrong.

MCP is the transport/integration mechanism. The underlying content may still be:

* customer text;
* external webpages;
* uploaded documents;
* stale policy;
* malicious prompt injection.

Always separate:

> **How did the information arrive?**

from:

> **What authority should that information have?**

### Eight sentences to memorize

> **Resources expose information; tools expose operations.**

> **MCP standardizes integration but does not replace authorization.**

> **Narrow, typed tool schemas reduce invalid actions.**

> **Least privilege limits blast radius.**

> **Error semantics determine recovery behavior.**

> **Authoritative information still needs freshness controls.**

> **Timeout does not prove a state-changing action failed.**

> **Trust boundaries should follow authority and capability, not merely connectivity.**
