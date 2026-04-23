# Chapter 6: Spec — Specification-Driven Development with AI

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain why specifications matter more, not less, when working with AI coding agents
- Apply core prompt engineering patterns to produce reliable AI-generated code
- Write precise, testable AI specifications using a structured template
- Use AI to generate, analyse, and validate requirements from user stories
- Recognise specification patterns for common task types and apply them correctly
- Iteratively refine a prompt through multiple design cycles

---

## 6.0 What Is Specification-Driven Development?

When a human developer receives a vague task — "add some kind of filtering to the list page" — they ask clarifying questions, draw on implicit organisational knowledge, and make reasonable assumptions. The result is still imperfect, but the developer can course-correct in real time. An AI coding agent receives the same vague task and does something different: it produces confident, syntactically correct code that satisfies a plausible interpretation of the request. That code may be entirely wrong for your system and you will not know it until you read every line.

This asymmetry is the central challenge of AI-native software engineering. AI agents are fluent but not clairvoyant. They cannot read your intent; they can only read your words. Specification-driven development (SDD) is the discipline of making those words precise enough that the gap between human intent and AI execution closes to an acceptable margin.

A **specification** in this context is a structured, unambiguous description of a unit of software behaviour — what it does, what it accepts, what it returns, what it refuses, and what invariants it must preserve. The specification is the contract. The AI agent is the contractor. A contractor who builds to an ambiguous contract will build something; it just may not be what you wanted.

The table below contrasts traditional requirements (written for human developers) with AI specifications (written to direct an AI agent):

| Dimension | Traditional Requirement | AI Specification |
|---|---|---|
| Audience | Human developer with domain context | AI agent with no implicit knowledge |
| Assumed knowledge | Organisational conventions, codebase history | Only what is explicitly provided |
| Ambiguity tolerance | Medium — developer asks questions | Zero — agent fills gaps with plausible guesses |
| Level of detail | High-level intent | Precise behaviour, types, error cases, examples |
| Validation | Code review, QA | Spec review before code is generated |
| Iteration cost | Low (developer iterates mentally) | Medium (each bad generation wastes a round-trip) |

Consider a user story written in the standard agile format:

> *As a project manager, I want to be able to mark a task as complete so that the team can track progress.*

A traditional requirement derived from this story might read: "The system shall allow project managers to mark tasks as complete." A human developer knows what "mark as complete" means in context. An AI agent does not. It does not know whether "complete" sets a boolean field or inserts an audit record, whether it triggers a notification, whether it is reversible, or whether it requires the task to already be in a specific state.

An AI specification for the same story might read:

```
Task: Implement `complete_task(task_id: int, completed_by: int) -> Task`

Context: Tasks exist in a PostgreSQL database via SQLAlchemy ORM. 
The Task model has fields: id, title, status (enum: OPEN, IN_PROGRESS, COMPLETE), 
assigned_to (FK to User), completed_by (nullable FK to User), completed_at (nullable datetime).

Behaviour:
- Load the task by task_id; raise TaskNotFoundError if absent
- Raise InvalidStateError if task.status is already COMPLETE
- Set task.status = COMPLETE, task.completed_by = completed_by, 
  task.completed_at = datetime.utcnow()
- Emit a task_completed event to the event bus
- Return the updated Task object

Error Handling:
- TaskNotFoundError: task_id does not exist
- InvalidStateError: task is already complete
- PermissionError: completed_by user does not have MANAGER role

Constraints:
- Must be idempotent-safe (calling twice raises InvalidStateError, not a silent duplicate)
- Do not commit the transaction — caller is responsible for session management
```

The AI specification is longer. That is not a flaw — it is the point. Every sentence you omit becomes a decision the AI makes for you.

---

## 6.1 What Is Prompt Engineering?

A **prompt** is the complete input you send to a language model: system instructions, context, examples, and the specific request. **Prompt engineering** is the practice of designing prompts that reliably produce outputs matching your intent.

The word "engineering" is deliberate. Prompt design is not intuition or art. It responds to systematic techniques, degrades predictably under identifiable failure modes, and can be tested and measured. A prompt is a kind of program — one that runs on a probabilistic machine rather than a deterministic one, but a program nonetheless.

In AI-native software engineering, prompt engineering appears in three distinct contexts:

**Code generation.** You describe a function, module, or algorithm and the agent writes it. The prompt must specify the interface, behaviour, and constraints clearly enough that the generated code requires no significant rewriting.

**Code review.** You provide existing code and ask the agent to identify defects, style violations, security issues, or logic errors. The prompt must constrain the scope and depth of review so the agent focuses where you need it.

**Task delegation.** You assign a multi-step engineering task — "refactor this module to use dependency injection" — and the agent executes it autonomously over several steps. The prompt must define success criteria and scope boundaries, because the agent will follow its interpretation to completion.

In all three contexts, the quality of the output is bounded by the quality of the prompt. A senior engineer who writes clear, complete specifications will consistently outperform a junior engineer using the same AI tools, because the senior engineer's prompts encode more of the relevant knowledge.

---

## 6.2 Why Prompts Fail

Understanding failure modes is prerequisite to avoiding them. Prompts fail in five recurring patterns.

### Ambiguity

An ambiguous prompt contains terms that admit multiple valid interpretations. The AI picks one interpretation silently and proceeds.

**Bad:** "Write a function that processes tasks."

"Processes" could mean parse, validate, execute, archive, or delete. "Tasks" could be database records, dictionary objects, or job queue items. The generated function will process something — just not necessarily the right thing.

**Good:** "Write a function `process_task(task: Task) -> ProcessingResult` that validates a Task object against the schema defined in `models.py`, applies the business rules in `rules.py`, and returns a `ProcessingResult` containing a status code and list of validation errors."

### Missing Constraints

Every function has implicit constraints: performance requirements, thread safety assumptions, transaction boundaries, side-effect policies. If you do not state them, the AI does not know them.

**Bad:** "Write a function to send email notifications to all users."

**Good:** "Write a function that queries users in batches of 100 to avoid memory exhaustion, sends each notification via the `EmailService.send()` method, and logs failures without raising — the caller must not be interrupted by individual send failures."

### Hallucinated APIs

Language models trained on broad code corpora have seen thousands of libraries. They will confidently use functions that do not exist, library versions that pre-date your pinned dependency, or internal APIs from similar-looking projects.

**Bad:** Asking for "SQLAlchemy code to do a bulk upsert" without specifying the SQLAlchemy version — the API for bulk upserts changed significantly between 1.4 and 2.0.

**Good:** State the exact library and version in the prompt: "Using SQLAlchemy 2.0 with the new-style `Session.execute()` API (not the legacy `session.bulk_insert_mappings()`)..."

### Overspecification

The inverse problem: providing so many constraints that they conflict, or prescribing implementation details that prevent the AI from finding a better approach.

**Bad:** "Write a binary search that uses a while loop with variables named `lo`, `hi`, and `mid`, where `mid` is computed as `(lo + hi) // 2`, and the loop condition is `lo <= hi`..." — at this point you have written the function yourself.

**Good:** Specify *what* the function must do and its observable behaviour. Leave *how* to the AI unless there is a concrete reason (e.g., performance constraint, existing pattern to match) to prescribe the implementation.

### Context Overload

Providing more context than the model can effectively use causes attention to diffuse. The model may miss the critical constraint buried in paragraph seven, or generate code that satisfies the most recent instruction but contradicts an earlier one.

**Bad:** Pasting 3,000 lines of codebase context before a simple function request.

**Good:** Curate context to the minimum necessary — the relevant data model, the function signature, and any directly called dependencies. See Section 6.5 for a systematic approach.

---

## 6.3 Core Prompt Patterns

Four patterns cover the majority of effective prompt engineering in software contexts.

### Role Prompting

Assign the AI an expert role in the system prompt. This conditions the model to apply conventions appropriate to that role — including the level of rigour, the vocabulary, and the implicit standards.

```
You are a senior Python engineer working on a financial services platform.
You follow PEP 8, write Google-style docstrings, use type hints on every
function, and treat all monetary values as Python Decimal, never float.
```

Role prompting is not magic — it does not grant capabilities the model lacks — but it shifts the probability distribution of outputs toward the conventions you need. A "senior Python engineer" prompt reliably produces type-annotated code; a bare prompt does not.

### Few-Shot Prompting

Provide two or three examples of input-output pairs before the actual request. The model infers the pattern from the examples and applies it to the new case. This is particularly effective for format-sensitive tasks like generating specifications, test cases, or structured data.

```
Example 1:
Input: "user can log in"
Output: {
  "function": "authenticate_user",
  "inputs": ["username: str", "password: str"],
  "returns": "AuthToken",
  "errors": ["InvalidCredentialsError", "AccountLockedError"]
}

Example 2:
Input: "user can reset password"
Output: {
  "function": "initiate_password_reset",
  "inputs": ["email: str"],
  "returns": "None",
  "errors": ["UserNotFoundError", "ResetCooldownError"]
}

Now generate the output for:
Input: "user can update their profile"
```

The model has learned from the examples that outputs should be structured JSON, that function names should be snake_case verbs, that inputs include type annotations, and that errors should be specific named exceptions.

### Chain-of-Thought

For tasks requiring multi-step reasoning — algorithm design, debugging, security analysis — instruct the model to reason explicitly before producing output. This reduces errors caused by the model "jumping to" an answer.

```
Think step by step about the edge cases for this function before writing it.
First list all the inputs that could cause problems. Then describe how you 
will handle each one. Then write the implementation.
```

Chain-of-thought is especially valuable when the correct answer is not the obvious answer — for instance, when handling timezone-aware datetimes, when implementing concurrent access patterns, or when reasoning about transaction isolation.

### Self-Consistency

For high-stakes generation, ask the model to produce multiple independent solutions and then reason about which is most correct. This exploits the probabilistic nature of language models: different runs produce different outputs, and the majority answer is statistically more reliable than any single answer.

```python
import anthropic

def generate_with_self_consistency(prompt: str, n: int = 3) -> str:
    client = anthropic.Anthropic()
    responses: list[str] = []

    for _ in range(n):
        message = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        responses.append(message.content[0].text)

    # Ask the model to pick the most consistent answer
    consistency_prompt = f"""
    I generated {n} solutions to a problem. Identify which solution is most
    correct and explain why. If they agree, synthesise the best version.

    Solutions:
    {chr(10).join(f'Solution {i+1}:{chr(10)}{r}' for i, r in enumerate(responses))}
    """

    final = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": consistency_prompt}]
    )
    return final.content[0].text
```

Self-consistency has a cost: it multiplies API calls by *n*. Reserve it for decisions that are expensive to reverse — core algorithm choices, security-sensitive logic, or database schema decisions.

---

## 6.4 Writing Precise AI Specifications

A specification for AI-directed code generation has a fixed structure. Each field exists because its absence causes a predictable failure.

### The Spec Template

```
## Task
[One sentence: what the function does]

## Function Signature
[Exact Python signature with type hints]

## Context
[What the function operates on: data models, dependencies, 
 calling conventions, relevant constants]

## Behaviour
[Numbered list of what the function does, in order of execution.
 Every conditional branch must appear here.]

## Error Handling
[Named exception for each failure mode, with the condition 
 that triggers it]

## Constraints
[Performance, thread safety, transaction, side-effect, 
 and style constraints]

## Examples
[2–3 input/output pairs covering normal case and at least 
 one edge case]
```

### Worked Example: `calculate_overdue_penalty`

```
## Task
Calculate the financial penalty owed on an overdue invoice.

## Function Signature
def calculate_overdue_penalty(
    invoice: Invoice,
    as_of: date,
    penalty_rate: Decimal = Decimal("0.015")
) -> Decimal:

## Context
- Invoice is a SQLAlchemy model with fields: 
    id (int), due_date (date), amount (Decimal), status (InvoiceStatus)
- InvoiceStatus is an enum: PENDING, PAID, CANCELLED
- penalty_rate is a daily rate (e.g. 0.015 = 1.5% per day overdue)
- Monetary values are always Decimal; never use float arithmetic

## Behaviour
1. If invoice.status is PAID or CANCELLED, return Decimal("0")
2. If as_of <= invoice.due_date, return Decimal("0") (not yet overdue)
3. Compute days_overdue = (as_of - invoice.due_date).days
4. Compute penalty = invoice.amount * penalty_rate * days_overdue
5. Round penalty to 2 decimal places using ROUND_HALF_UP
6. Return penalty

## Error Handling
- TypeError: if invoice.amount is not Decimal (guard at function entry)
- ValueError: if as_of is before the epoch date (date(1970, 1, 1))

## Constraints
- No database calls — all data must be passed in
- No side effects
- Pure function; safe to call multiple times with same arguments

## Examples
- Invoice(due_date=date(2024,1,1), amount=Decimal("1000"), status=PENDING),
  as_of=date(2024,1,11) → Decimal("150.00")  # 10 days * 1.5%
- Invoice(status=PAID, ...) → Decimal("0")
- Invoice(due_date=date(2024,1,1), ...), as_of=date(2024,1,1) → Decimal("0")
```

This specification is complete enough that any competent developer — human or AI — can implement the function without asking a single clarifying question. That is the test of a good spec.

---

## 6.5 Context Engineering

A prompt is not just the instruction — it is everything the AI receives. Context engineering is the discipline of deciding what to include, what to omit, and how to structure the surrounding material so the model attends to what matters.

### What to Include

**Relevant data models.** If the function operates on a `Task` object, include the `Task` class definition — its fields, types, and any relevant methods. Do not summarise it; paste the actual class.

**Directly called dependencies.** If the function calls `EmailService.send()`, include the signature and docstring for `send()`. The AI needs to know what it returns and what exceptions it raises.

**Existing tests.** If the codebase has tests for related functions, include one or two. They communicate conventions (assert style, fixture patterns, naming) more efficiently than prose descriptions.

**Relevant conventions.** If your project uses a specific error hierarchy, a particular logging pattern, or a session management convention, state it explicitly or paste a representative example.

### What to Omit

**Unrelated files.** Pasting an entire module when you need one function from it costs tokens and diffuses attention. Extract only the relevant portions.

**Verbose documentation.** Long module docstrings, changelog entries, and historical comments add noise. The AI does not benefit from knowing that a function was "refactored in Q3 2022."

**Duplicate information.** If the spec already states the type of a parameter, do not also paste the Pydantic model that defines it unless the model adds constraints the spec omits.

### Context Budget

Language models have context windows measured in tokens (roughly 0.75 words per token). Larger context windows do not mean larger is better — attention quality degrades over very long contexts, and the "lost in the middle" phenomenon means content in the centre of a long prompt is attended to less reliably than content at the start or end.

A practical budget for single-function generation tasks:

| Component | Recommended Token Budget |
|---|---|
| System prompt / role | 100–200 tokens |
| Specification | 300–500 tokens |
| Data models | 200–400 tokens |
| Dependency signatures | 100–200 tokens |
| Examples | 100–300 tokens |
| **Total** | **~1000–1600 tokens** |

Staying under 2,000 tokens for the input context of a single-function task is a reasonable guideline. For larger tasks — module refactoring, multi-function generation — the budget scales up, but the principle of ruthless curation remains.

---

## 6.6 Using AI to Generate, Analyse, and Validate Requirements

Writing specifications by hand is effective but labour-intensive. AI agents can accelerate the specification process itself — decomposing user stories into function-level specs, and then critiquing those specs for completeness, consistency, and testability. This section shows both directions: AI as spec author and AI as spec critic.

### Decomposing User Stories into Function Specifications

A user story describes intent at the business level. To direct an AI agent to implement software, that intent must be translated into one or more function-level specifications. AI can do this translation reliably when given a structured prompt.

Consider the following user story:

> *As a project manager, I want to assign tasks to team members so that responsibilities are clear and the team knows who is accountable.*

A naive prompt — "turn this user story into function specs" — produces vague output. A structured prompt produces something usable:

```
You are a senior software engineer writing specifications for a 
task management system built in Python. The system uses SQLAlchemy 
ORM, FastAPI for the HTTP layer, and raises typed exceptions 
(TaskNotFoundError, UserNotFoundError, PermissionError).

User story:
"As a project manager, I want to assign tasks to team members 
so that responsibilities are clear."

Decompose this story into 3–4 function-level specifications. 
For each function, provide:
- Function name and signature (with Python type hints)
- What it does (2–3 sentences)
- Pre-conditions (what must be true before calling it)
- Post-conditions (what is guaranteed after it returns)
- Error cases (named exceptions and the conditions that trigger them)

Format each spec as a structured block, not prose.
```

A well-prompted model produces output along these lines:

```
Function 1: assign_task
Signature: assign_task(task_id: int, assignee_id: int, assigned_by: int) -> Task
Does: Assigns an existing task to a team member. Records who made the 
      assignment and when. Emits a task_assigned event.
Pre-conditions: task exists; assignee is an active team member; 
                assigned_by holds MANAGER role
Post-conditions: task.assigned_to == assignee_id; 
                 task.assigned_at is set; event emitted
Errors: TaskNotFoundError (task_id absent), UserNotFoundError (assignee absent),
        PermissionError (assigned_by lacks MANAGER role),
        InvalidStateError (task is COMPLETE)

Function 2: get_task_assignee
Signature: get_task_assignee(task_id: int) -> User | None
Does: Returns the User currently assigned to a task, or None if unassigned.
Pre-conditions: task exists
Post-conditions: returns User object or None; no side effects
Errors: TaskNotFoundError (task_id absent)

Function 3: list_tasks_by_assignee
Signature: list_tasks_by_assignee(assignee_id: int, 
           status_filter: TaskStatus | None = None) -> list[Task]
Does: Returns all tasks assigned to a given user, optionally filtered by status.
Pre-conditions: assignee_id refers to an existing user
Post-conditions: returns list (empty if none match); no side effects
Errors: UserNotFoundError (assignee_id absent)

Function 4: reassign_task
Signature: reassign_task(task_id: int, new_assignee_id: int, 
           reassigned_by: int) -> Task
Does: Changes a task's assignee. Appends an entry to the task's 
      assignment history. Emits a task_reassigned event.
Pre-conditions: task exists; new_assignee is active; reassigned_by is MANAGER
Post-conditions: task.assigned_to == new_assignee_id; history updated; 
                 event emitted
Errors: TaskNotFoundError, UserNotFoundError, PermissionError, 
        InvalidStateError (task is COMPLETE)
```

Notice what the AI-generated output provides that the original user story omitted: the distinction between initial assignment and reassignment, the existence of an assignment history, the rule that completed tasks cannot be reassigned, and the separation of the read path (`get_task_assignee`, `list_tasks_by_assignee`) from the write path. This is the leverage: a single structured prompt turns a one-line user story into four implementation-ready specifications in seconds.

### The Spec Critic Pattern

Once a specification exists — whether written by a human or generated by AI — it must be validated before an agent acts on it. A second AI call, using a "spec critic" persona, can identify the classes of problems that cause bad code generation: missing error cases, ambiguous terms, and untestable clauses.

The spec critic pattern uses a separate prompt with a critical review role:

```
You are a software specification reviewer. Your job is to find problems
in specifications before they are sent to a code generation agent.

Review the following specification and identify:
1. Missing error cases: conditions that can occur but have no specified behaviour
2. Ambiguous terms: words or phrases that admit more than one valid interpretation
3. Untestable clauses: statements that cannot be verified by a unit or 
   integration test (e.g., "should be fast", "handles edge cases")
4. Missing constraints: performance, concurrency, or transaction requirements 
   that are likely relevant but not stated

Return your findings as JSON with keys: 
"missing_error_cases", "ambiguous_terms", "untestable_clauses", 
"missing_constraints", "overall_quality" (score 1–10).
```

The following Python function implements this pattern as a reusable utility:

```python
import json
import anthropic
from typing import TypedDict

class SpecValidationResult(TypedDict):
    missing_error_cases: list[str]
    ambiguous_terms: list[str]
    untestable_clauses: list[str]
    missing_constraints: list[str]
    overall_quality: int

CRITIC_SYSTEM_PROMPT = """
You are a software specification reviewer. Your job is to identify problems
in specifications before they are sent to a code generation agent.

For each issue you find, be specific: quote the problematic text and 
explain why it is a problem. Return only valid JSON — no markdown fences, 
no prose outside the JSON object.
""".strip()

def validate_spec(spec: str) -> SpecValidationResult:
    """
    Submit a specification to an AI critic and return a structured
    validation report.

    Args:
        spec: The full text of the function specification to validate.

    Returns:
        A SpecValidationResult dict with categorised issues and a 
        quality score from 1 (unusable) to 10 (production-ready).

    Raises:
        ValueError: If the model returns malformed JSON.
        anthropic.APIError: On API-level failures.
    """
    client = anthropic.Anthropic()

    review_prompt = f"""
Review the following function specification for problems.

Return a JSON object with exactly these keys:
- "missing_error_cases": list of strings, each describing a condition 
  that can occur but has no specified behaviour
- "ambiguous_terms": list of strings, each quoting an ambiguous phrase 
  and explaining the ambiguity
- "untestable_clauses": list of strings, each quoting a clause that 
  cannot be verified by automated testing
- "missing_constraints": list of strings, each describing a likely-relevant 
  constraint (performance, concurrency, transactions) not stated
- "overall_quality": integer 1–10

--- SPECIFICATION ---
{spec}
--- END SPECIFICATION ---
""".strip()

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system=CRITIC_SYSTEM_PROMPT,
        messages=[{"role": "user", "content": review_prompt}]
    )

    raw: str = message.content[0].text.strip()

    try:
        result = json.loads(raw)
    except json.JSONDecodeError as e:
        raise ValueError(
            f"Spec critic returned malformed JSON: {e}\nRaw output:\n{raw}"
        ) from e

    # Validate presence of expected keys
    expected_keys = {
        "missing_error_cases", "ambiguous_terms",
        "untestable_clauses", "missing_constraints", "overall_quality"
    }
    missing_keys = expected_keys - result.keys()
    if missing_keys:
        raise ValueError(
            f"Spec critic response missing keys: {missing_keys}"
        )

    return SpecValidationResult(
        missing_error_cases=result["missing_error_cases"],
        ambiguous_terms=result["ambiguous_terms"],
        untestable_clauses=result["untestable_clauses"],
        missing_constraints=result["missing_constraints"],
        overall_quality=result["overall_quality"]
    )


def validate_and_report(spec: str) -> None:
    """Print a human-readable validation report for a specification."""
    result = validate_spec(spec)
    score = result["overall_quality"]

    print(f"Specification Quality Score: {score}/10")
    print("=" * 50)

    sections = [
        ("Missing Error Cases", result["missing_error_cases"]),
        ("Ambiguous Terms", result["ambiguous_terms"]),
        ("Untestable Clauses", result["untestable_clauses"]),
        ("Missing Constraints", result["missing_constraints"]),
    ]

    for title, items in sections:
        if items:
            print(f"\n{title}:")
            for item in items:
                print(f"  - {item}")
        else:
            print(f"\n{title}: none found")
```

To illustrate the value of this pattern, consider validating a weak specification:

```python
weak_spec = """
Function: assign_task
Assigns a task to a user. Should handle errors gracefully.
The function should be fast and work correctly in all cases.
Returns True on success.
"""

validate_and_report(weak_spec)
```

A typical output from this validation call:

```
Specification Quality Score: 2/10
==================================================

Missing Error Cases:
  - No behaviour specified when task_id does not exist
  - No behaviour specified when user_id does not exist
  - No behaviour specified when the task is already assigned
  - No behaviour specified when the caller lacks permission to assign

Ambiguous Terms:
  - "handle errors gracefully" — admits any error handling behaviour; 
    does not specify whether to raise, log, return None, or swallow
  - "assign" — unclear whether this overwrites an existing assignment 
    or raises an error if one exists

Untestable Clauses:
  - "should be fast" — no measurable performance criterion given
  - "work correctly in all cases" — tautological; not a testable statement

Missing Constraints:
  - No transaction boundary specified
  - No statement on whether the operation is idempotent
  - No event or notification behaviour specified
```

The score of 2/10 and the specificity of the findings give the spec author a concrete remediation list. After revising the spec and rerunning validation, the score converges upward. This iterative loop — write, validate, revise — is faster and cheaper than discovering the same deficiencies in code review after an agent has already generated an implementation.

### Iterative Spec Refinement with AI Feedback

The validation output is itself a prompt. After `validate_spec` returns, you can feed the issues directly back to the AI along with the original spec and ask for a revised version:

```python
def refine_spec(original_spec: str, validation: SpecValidationResult) -> str:
    """
    Use AI feedback to produce a revised specification that addresses 
    the issues identified by validate_spec.
    """
    client = anthropic.Anthropic()

    issues_text = "\n".join([
        f"Missing error cases: {validation['missing_error_cases']}",
        f"Ambiguous terms: {validation['ambiguous_terms']}",
        f"Untestable clauses: {validation['untestable_clauses']}",
        f"Missing constraints: {validation['missing_constraints']}",
    ])

    refine_prompt = f"""
The following specification has quality issues. Rewrite it to fix all of them.
Keep the same function signature and intent. Add the missing detail.

Original specification:
{original_spec}

Issues to fix:
{issues_text}

Return only the revised specification — no explanation.
""".strip()

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": refine_prompt}]
    )
    return message.content[0].text.strip()
```

Running `validate_spec` and `refine_spec` in a loop with a quality threshold (e.g., stop when `overall_quality >= 8`) gives you an automated spec refinement pipeline. In practice, two to three iterations are sufficient for most function-level specifications.

---

## 6.7 Specification Patterns for Common Task Types

Different categories of software operation have different specification requirements. A function that deletes a database record has different failure modes from one that submits a background job, which differs again from one that validates an authentication token. Applying a generic spec template to all of these produces specifications that are technically structured but miss the category-specific details that matter.

This section presents four common task types, describes the specification elements that matter for each, and shows the difference between a naive spec and a complete one.

### Overview

| Task Type | Key Spec Elements Often Omitted |
|---|---|
| CRUD operations | Validation rules per field; partial-update semantics; cascade behaviour on delete; return type on create vs update |
| Authentication / authorisation | Token expiry behaviour; concurrent session handling; auth failure audit logging; rate limiting |
| Async jobs / background tasks | Job deduplication / idempotency key; status transition graph; failure retry policy; result TTL |
| REST API endpoints | All error response schemas (not just 200); query param validation; pagination contracts; rate limit headers |

### CRUD Operations

CRUD (Create, Read, Update, Delete) operations are the most common task type and the most frequently underspecified. The naive assumption is that CRUD is simple. It is not — the edge cases are numerous and consequential.

**What a naive spec includes:** "Creates a new task in the database and returns it."

**What a complete spec adds:**

- *Validation*: Which fields are required? What are the length, format, and range constraints for each? What happens when a required field is missing (raise, or return a validation error object)?
- *Uniqueness*: Are any fields unique? What happens on a duplicate (raise `DuplicateError`, or upsert)?
- *Defaults*: Which fields have server-side defaults (e.g., `created_at = now()`, `status = OPEN`)? Are any defaults caller-overridable?
- *Return type*: Does the create function return the created object (with the generated `id`), or just the `id`, or `None`?
- *Cascade on delete*: When a `Project` is deleted, what happens to its `Task` records — cascade delete, nullify, or raise `DependencyError`?

```
## Task
Create a new Task record in the database.

## Function Signature
def create_task(
    title: str,
    project_id: int,
    assigned_to: int | None = None,
    due_date: date | None = None,
    created_by: int
) -> Task:

## Validation
- title: required; 1–200 characters; stripped of leading/trailing whitespace
- project_id: must refer to an existing, non-archived Project
- assigned_to: if provided, must refer to an active User in the project
- due_date: if provided, must be >= today

## Behaviour
1. Validate all inputs; raise ValidationError with field-level detail on failure
2. Set status = OPEN, created_at = datetime.utcnow()
3. Insert and flush (do not commit — caller manages transaction)
4. Return the full Task object including generated id

## Error Handling
- ValidationError: any field fails validation (includes field name and reason)
- ProjectNotFoundError: project_id absent or archived
- UserNotFoundError: assigned_to not found or not in project
```

### Authentication and Authorisation

Auth specifications fail when they omit the security-relevant details — the exact conditions for token rejection, the behaviour on repeated failures, and the audit trail requirements.

**What a naive spec includes:** "Checks if the user is authenticated and has permission."

**What a complete spec adds:**

- *Token validation chain*: Is the token checked for format, then signature, then expiry, in that order? What specific exception does each check raise?
- *Expiry semantics*: Is expiry checked against server time or the token's `iat` claim? Is clock skew tolerated? By how much?
- *Authorisation model*: Role-based (RBAC), attribute-based (ABAC), or resource-based? What is the exact permission string checked?
- *Failure audit*: Are failed auth attempts logged? To where? With what fields (user_id if known, IP address, timestamp)?
- *Rate limiting*: Is there a lockout policy after N failures? Who enforces it — this function or a middleware layer?

```
## Task
Verify a Bearer token and confirm the caller has a specific permission.

## Function Signature
def require_permission(
    token: str,
    permission: str,
    resource_id: int | None = None
) -> AuthContext:

## Behaviour
1. Validate token format (must be non-empty string); raise MalformedTokenError if not
2. Decode and verify JWT signature using RS256; raise InvalidTokenError if invalid
3. Check token expiry (server UTC time, no clock-skew tolerance); 
   raise TokenExpiredError if expired
4. Load user from token subject claim; raise UserNotFoundError if absent or inactive
5. Check user has `permission` (optionally scoped to `resource_id`); 
   raise PermissionDeniedError if not
6. Log auth success to audit log with: user_id, permission, resource_id, timestamp
7. Return AuthContext(user_id, roles, token_expiry)

## Error Handling
- MalformedTokenError: token is empty or not a valid JWT structure
- InvalidTokenError: signature verification fails
- TokenExpiredError: token exp claim is in the past
- UserNotFoundError: subject user does not exist or is deactivated
- PermissionDeniedError: user lacks the required permission

## Constraints
- All failures must be logged to the security audit log before raising
- This function must not raise generic Exception — all failures use typed errors
- Execution time must not depend on whether the token is valid (constant-time 
  comparison for signature verification to prevent timing attacks)
```

### Async Jobs and Background Tasks

Background task specifications fail on the operational questions: what happens when a job is submitted twice? How does the caller know when it is done? What happens on failure?

**What a naive spec includes:** "Submits a report generation job and returns the job ID."

**What a complete spec adds:**

- *Idempotency key*: If the same job is submitted twice with the same parameters, does it create a new job or return the existing one?
- *Status transition graph*: What are the valid statuses (QUEUED → RUNNING → COMPLETE | FAILED)? What transitions are valid? Who triggers each?
- *Failure handling*: How many times is a failed job retried? With what backoff policy? After final failure, does it stay in FAILED state or move to DEAD?
- *Result access*: Where is the result stored? How long is it retained? What does the caller poll, and at what interval?
- *Cancellation*: Can a queued or running job be cancelled? What is the cancellation contract?

```
## Task
Submit a background report generation job.

## Function Signature
def submit_report_job(
    report_type: ReportType,
    parameters: dict[str, Any],
    requested_by: int,
    idempotency_key: str | None = None
) -> JobHandle:

## Behaviour
1. If idempotency_key is provided and a job with that key exists in 
   QUEUED or RUNNING state, return that job's JobHandle without 
   creating a new job (idempotent submission)
2. Validate parameters against ReportType schema; 
   raise ValidationError on failure
3. Create a Job record with status = QUEUED, enqueue to the task queue
4. Return JobHandle(job_id, status=QUEUED, poll_url)

## Status Transitions
QUEUED → RUNNING (worker picks up job)
RUNNING → COMPLETE (job succeeds; result stored in job.result_location)
RUNNING → FAILED (job raises exception; stored in job.error_detail)
FAILED → QUEUED (automatic retry; max 3 retries with exponential backoff)
FAILED (after 3 retries) → DEAD (no further retries; human intervention required)

## Error Handling
- ValidationError: parameters do not match ReportType schema
- QuotaExceededError: requested_by user has exceeded concurrent job limit (5)

## Constraints
- Job results retained for 72 hours after COMPLETE; then purged
- idempotency_key deduplication window: 24 hours
```

### REST API Endpoints

REST endpoint specifications are often written as prose ("accepts a POST request with task data") and omit the schema-level detail that determines whether the generated handler code is correct.

**What a naive spec includes:** "POST /tasks — creates a task."

**What a complete spec adds:**

- *Full request schema*: Every field in the request body, its type, whether it is required, and its validation constraints.
- *All response schemas*: Not just the 200/201 success schema, but the error response schema for every error code the endpoint can return.
- *Authorisation*: What role or permission is required? Where is the auth token expected (Authorization header)?
- *Idempotency*: Does the endpoint support `Idempotency-Key` header? What is the deduplication window?
- *Rate limiting*: Is this endpoint rate-limited? What headers communicate the limit and remaining quota?

```
## Endpoint
POST /api/v1/tasks

## Authorisation
Requires Bearer token with `tasks:write` permission.
Returns 401 if token absent or invalid; 403 if token valid but permission absent.

## Request
Content-Type: application/json
Body schema:
  title:       string, required, 1–200 chars
  project_id:  integer, required
  assigned_to: integer, optional
  due_date:    string (ISO 8601 date), optional; must be >= today

## Responses
201 Created: Task object
  { "id": int, "title": str, "status": "OPEN", 
    "project_id": int, "assigned_to": int|null, 
    "due_date": str|null, "created_at": str (ISO 8601) }

400 Bad Request: Validation failure
  { "error": "validation_error", 
    "fields": { "<field_name>": "<reason>", ... } }

401 Unauthorized: { "error": "unauthorized" }
403 Forbidden:    { "error": "forbidden" }
404 Not Found (project_id not found):
  { "error": "not_found", "resource": "project" }
409 Conflict (duplicate idempotency key with different params):
  { "error": "conflict", "existing_id": int }

## Headers
Idempotency-Key: optional; string; deduplication window 1 hour
X-RateLimit-Limit: 100 (requests per minute per user)
X-RateLimit-Remaining: current remaining
X-RateLimit-Reset: Unix timestamp of window reset
```

The pattern across all four task types is the same: the naive spec describes the happy path. The complete spec describes every path — including the ones that will actually be exercised in production.

---

## 6.8 Tutorial: Iterative Spec Design for the Course Project

This tutorial demonstrates the iterative spec design process using `assign_task` — a function from the course project task management system. You will see three generations of the spec, each one exposing and fixing the deficiencies of the previous version.

### Iteration 1: The First Attempt

A student new to spec-driven development writes the following prompt:

```
Write a Python function called assign_task that assigns a task to a user.
```

The AI generates a function that takes `task` and `user` as parameters, sets `task.assigned_to = user`, and returns `None`. The code is syntactically correct. It has no error handling. It accepts any object for both parameters. It has no understanding of the system's status model, permission requirements, or event system. It will need to be almost entirely rewritten.

**What went wrong:** The prompt contains no context, no types, no error cases, no constraints, and no examples. It is a sentence, not a specification.

### Iteration 2: Adding Structure

The student applies the spec template from Section 6.4:

```
## Task
Implement assign_task, which assigns an open task to a team member.

## Function Signature
def assign_task(task_id: int, assignee_id: int, assigned_by: int) -> Task:

## Context
Uses SQLAlchemy Session (passed as a global `db.session`).
Task model has: id, title, status (TaskStatus enum: OPEN, IN_PROGRESS, COMPLETE),
assigned_to (FK to User).
User model has: id, name, role (UserRole enum: MEMBER, MANAGER).

## Behaviour
1. Load task by task_id; raise TaskNotFoundError if not found
2. Load user by assignee_id; raise UserNotFoundError if not found
3. Set task.assigned_to = assignee_id
4. Return the updated task

## Error Handling
- TaskNotFoundError: task not found
- UserNotFoundError: user not found
```

The AI now generates a function with proper ORM queries and typed exceptions. However, during code review, several gaps emerge: there is no permission check (any caller can assign any task), no guard against assigning a completed task, no `assigned_at` timestamp, and the session management is incorrect (the function uses a global session rather than accepting one as a parameter).

**What went wrong:** The spec improved substantially, but omitted the permission model, the status constraint, the audit fields, and the correct session pattern.

### Iteration 3: The Complete Specification

The student adds the missing elements:

```
## Task
Assign an open task to a team member. Records who made the assignment 
and when. Emits a task_assigned event.

## Function Signature
def assign_task(
    session: Session,
    task_id: int,
    assignee_id: int,
    assigned_by: int
) -> Task:

## Context
- session: SQLAlchemy Session; do not commit inside this function
- Task model fields: id, title, status (TaskStatus: OPEN/IN_PROGRESS/COMPLETE),
  assigned_to (FK→User, nullable), assigned_at (datetime, nullable),
  last_modified_by (FK→User)
- User model fields: id, name, role (UserRole: MEMBER/MANAGER), is_active (bool)
- Event system: call event_bus.emit("task_assigned", {"task_id": ..., 
  "assignee_id": ..., "assigned_by": ...})

## Behaviour
1. Load task by task_id using session; raise TaskNotFoundError if absent
2. Raise InvalidStateError if task.status == TaskStatus.COMPLETE
3. Load assigned_by user; raise UserNotFoundError if absent or not is_active
4. Raise PermissionError if assigned_by_user.role != UserRole.MANAGER
5. Load assignee user by assignee_id; raise UserNotFoundError if absent 
   or not is_active
6. Set task.assigned_to = assignee_id
7. Set task.assigned_at = datetime.utcnow()
8. Set task.last_modified_by = assigned_by
9. session.flush() (do not commit)
10. Call event_bus.emit("task_assigned", {...})
11. Return the updated Task object

## Error Handling
- TaskNotFoundError: task_id not in database
- InvalidStateError: task.status is COMPLETE
- UserNotFoundError: assigned_by or assignee_id not found or inactive
- PermissionError: assigned_by_user.role is not MANAGER

## Constraints
- No database commit — caller is responsible for transaction
- event_bus.emit must be called after flush, not before
- All monetary and datetime fields use UTC; no naive datetimes

## Examples
- assign_task(session, task_id=1, assignee_id=5, assigned_by=2)
  → Task(id=1, assigned_to=5, status=OPEN, assigned_at=<utcnow>)
- assign_task(session, task_id=99, ...) → raises TaskNotFoundError
- assign_task(session, task_id=1, ..., assigned_by=<MEMBER user>) 
  → raises PermissionError
```

The AI generates a function that requires no modification. It handles all stated cases, uses the correct session pattern, emits the event in the correct order, and uses typed exceptions throughout.

### What Changed Across Iterations

| Element | Iteration 1 | Iteration 2 | Iteration 3 |
|---|---|---|---|
| Function signature with types | No | Yes | Yes |
| Session management | No | Incorrect (global) | Correct (parameter) |
| Permission check | No | No | Yes |
| Status guard | No | No | Yes |
| Audit fields (assigned_at) | No | No | Yes |
| Event emission | No | No | Yes |
| Error cases | 0 | 2 | 4 |
| Examples | 0 | 0 | 3 |
| Usable without rewriting | No | Partial | Yes |

The progression is not about writing more words — it is about writing the right words. Each iteration adds the elements that the previous version's generated code revealed were missing. The cost of writing the complete spec is ten minutes. The cost of discovering the missing elements in code review is proportionally higher with every element that must be found, diagnosed, and corrected.

---

## Chapter Summary

Specification-driven development is the foundational discipline of AI-native software engineering. The central argument of this chapter is simple: AI agents do not fill gaps with your intent — they fill gaps with plausible guesses. Every element of behaviour, constraint, error case, and context that you omit from a specification becomes a decision the agent makes without you.

The chapter established that effective AI specifications are different from traditional software requirements. They require zero tolerance for ambiguity, explicit statement of every constraint the agent cannot infer, and coverage of every error case — not just the happy path. The spec template (Task, Function Signature, Context, Behaviour, Error Handling, Constraints, Examples) provides a complete framework for writing specifications that produce usable generated code on the first or second attempt.

Prompt engineering supplies the techniques for communicating with AI agents effectively: role prompting to establish the relevant conventions, few-shot examples to establish output formats, chain-of-thought to improve reasoning quality on complex problems, and self-consistency to improve reliability on high-stakes decisions. Context engineering ensures that the right information reaches the model at the right level of detail, without the attention dilution that comes from providing everything.

AI can be directed inward — to generate and validate specifications themselves. The decomposition pattern turns user stories into function-level specs in seconds. The spec critic pattern systematically identifies missing error cases, ambiguous terms, untestable clauses, and missing constraints before any code is generated. The `validate_spec` function shown in Section 6.6 implements this pattern as a reusable tool that can be integrated into a specification workflow.

The specification patterns for CRUD operations, authentication, async jobs, and REST endpoints document the category-specific elements that consistently appear in naive specs as omissions. In every category, the pattern is the same: the naive spec covers the happy path; the complete spec covers every path.

---

## Review Questions

**1.** A colleague argues that writing detailed AI specifications takes more time than just prompting the AI and iterating on the output until it looks right. Construct a counter-argument using the concept of *total engineering cost*, considering the cost of code review, debugging, and rework when a generated function handles error cases incorrectly.

**2.** The following prompt is given to an AI agent: *"Write a function to update a user's email address."* Using the five failure categories from Section 6.2 (ambiguity, missing constraints, hallucinated APIs, overspecification, context overload), identify which failures this prompt is most likely to trigger and explain why.

**3.** You are building a `validate_spec` pipeline and the model returns a quality score of 6/10 after the first revision. The remaining issues are: one ambiguous term ("the function should handle concurrent calls") and one missing constraint (no statement on whether the operation is idempotent). Write the revised specification clause that resolves both issues, and explain why idempotency and concurrency safety are distinct properties that require separate statements.

**4.** Compare the specification requirements for a *synchronous* REST endpoint (`POST /tasks`) and an *asynchronous* job submission endpoint (`POST /reports/jobs`). List at least four elements that appear in the async spec but are irrelevant to the synchronous spec, and explain why each element is necessary for the async case.

**5.** The `generate_with_self_consistency` function in Section 6.3 makes *n* API calls and then a synthesis call. Describe a scenario in software engineering where self-consistency is worth the extra cost, and a scenario where it is not. For each scenario, explain what property of the task drives your decision.
