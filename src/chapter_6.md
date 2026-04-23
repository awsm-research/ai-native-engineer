# Chapter 6: Prompt Engineering and Context Design

> *"A language model is a reasoning engine. Your prompt is the problem statement you hand it. Garbage in, garbage out — but more precisely: ambiguous in, plausible-sounding garbage out."*

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain what prompt engineering is and why it matters for software engineering tasks.
2. Apply core prompt patterns: role prompting, chain-of-thought, few-shot, and self-consistency.
3. Write precise, testable AI specifications for realistic software features.
4. Identify and avoid common prompt failure modes.
5. Design effective context for AI coding tasks: what to include, what to omit.
6. Apply prompt engineering techniques to produce higher-quality AI-generated code.

---

## From Requirements to AI Specifications: Clarifying the Distinction

In Chapter 2, you wrote *requirements* — descriptions of what the system must do, expressed as user stories and acceptance criteria for human readers and stakeholders. In this chapter, you will write *AI specifications* — descriptions of what a single function must do, expressed for a language model that will generate the implementation.

These two artefacts serve different purposes and are written differently:

| Dimension | Requirements (Ch. 2) | AI Specifications (Ch. 6) |
|---|---|---|
| **Audience** | Stakeholders, developers, testers | The language model generating code |
| **Scope** | A feature or user story | A single function or class |
| **Format** | Gherkin, user story, prose | Structured template: signature, behaviour, constraints, examples |
| **Level of detail** | Business-level intent | Implementation-level contract |
| **When written** | Before design begins | Immediately before generation |
| **Primary goal** | Align stakeholders on what to build | Constrain the model to generate correct code |

**The relationship:** A well-written user story from Chapter 2 provides the *intent*; an AI specification translates that intent into the precise *contract* the implementation must satisfy. One user story typically decomposes into several AI specifications — one per function or method.

**Example translation:**

*Chapter 2 user story:*
> As a project manager, I want to assign a task to a team member so that responsibilities are clear.

*Chapter 6 AI specification for one function this story requires:*
```
Function: assign_task(task_id, assignee_email, assigned_by) -> Task
Constraints: assignee must be a project member; only MANAGERs may assign;
             assignee != assigned_by; raises specific errors for each violation.
Examples: [specific input/output pairs for each scenario]
```

Neither document replaces the other. The user story stays in the product backlog; the AI specification lives in the development workflow, used when generating that function and discarded (or archived) after.

---

## 6.1 What Is Prompt Engineering?

A *prompt* is the input you provide to a language model. Prompt engineering is the discipline of designing prompts that reliably produce useful, accurate outputs for a given task.

The term "engineering" is deliberate. Getting consistently good results from a language model requires systematic thinking — about what information the model needs, how that information should be structured, and what constraints should be made explicit. It is not about finding magic words or tricks; it is about clear communication with a system that interprets natural language.

For software engineers, prompt engineering is most valuable in three contexts:

1. **Code generation**: Writing specifications that produce correct, maintainable implementations
2. **Code review**: Writing prompts that elicit substantive critique rather than superficial approval
3. **Task delegation to agents**: Writing goal descriptions that agents can execute reliably

This chapter focuses primarily on (1) and provides foundations for (2) and (3).

---

## 6.2 Why Prompts Fail

Before examining techniques that work, it is useful to understand why prompts fail. Most failures fall into one of five categories:

### 6.2.1 Ambiguity

An ambiguous prompt has multiple plausible interpretations, and the model picks one — often not the one you intended.

**Example**:
> "Write a function that processes tasks"

"Processes" is ambiguous: Does it mean validate? Transform? Filter? Save to a database? The model will choose an interpretation based on what seems most common in its training data — which may not match your intent.

**Fix**: Replace vague verbs with precise ones. "Write a function that validates a Task object's required fields and raises `ValidationError` with a descriptive message for each violated constraint" is unambiguous.

### 6.2.2 Missing Constraints

A prompt that describes what the function *should* do, but not what it *must not* do, often produces output that violates implicit constraints the author took for granted.

**Example**: You ask for a function that retrieves tasks from a database. The model generates a solution that constructs SQL by string concatenation — technically correct, but introducing a SQL injection vulnerability you never thought to prohibit.

**Fix**: Enumerate explicit constraints: "Use parameterised queries only. Do not construct SQL strings by concatenation. Raise `NotFoundError` (not `None`) when a task does not exist."

### 6.2.3 Hallucinated APIs

Language models are trained on code from a fixed point in time. When asked to use a library, they may generate plausible-looking code that calls functions or methods that do not exist in the current version of the library.

**Fix**: Provide the specific function signatures or documentation excerpts you want the model to use. Do not assume it has accurate knowledge of your library versions.

### 6.2.4 Overspecification

Prompts that are too prescriptive — specifying exactly *how* to implement something rather than *what* to implement — can produce worse results than letting the model choose an appropriate approach.

**Fix**: Specify the *contract* (inputs, outputs, constraints, examples), not the implementation. Reserve implementation guidance for cases where you have a specific reason to constrain the approach.

### 6.2.5 Context Overload

Including too much context confuses the model. If you paste 10,000 lines of codebase into a prompt alongside a specific question, the model may fail to identify what is relevant and generate a response that addresses the noise rather than the signal.

**Fix**: Be selective about context. Include only what the model actually needs: the interface definition, the relevant data structures, and any constraints specific to the task.

---

## 6.3 Core Prompt Patterns

The following patterns are broadly useful for software engineering tasks. They are not mutually exclusive — effective prompts often combine several.

### 6.3.1 Role Prompting

Assigning a role to the model primes it to approach the task with a specific perspective and apply relevant domain knowledge.

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    system="You are a senior Python software engineer with expertise in API design, "
           "testing, and security. You follow PEP 8, use type hints on all function "
           "signatures, and write minimal but correct docstrings.",
    messages=[
        {
            "role": "user",
            "content": "Implement a function that validates a task creation request...",
        }
    ],
)
```

The `system` prompt in the Anthropic API ([Anthropic, 2024](https://docs.anthropic.com/en/api/getting-started)) sets persistent context for the conversation. For code generation, the system prompt is the right place for:

- The engineer's role and expertise
- Style conventions (PEP 8, line length, type hints)
- Library preferences
- Security constraints that apply to all generated code

### 6.3.2 Few-Shot Prompting

Providing examples of the desired input-output pattern ("few-shot examples") significantly improves output quality for tasks where the format or style matters.

```python
few_shot_prompt = """
Convert each plain description into a precise function specification.

Description: "function to add two numbers"
Specification:
  Function: add(a: float, b: float) -> float
  Constraints:
    - Accepts any finite float values
    - Returns the arithmetic sum
  Examples:
    add(1.0, 2.0) == 3.0
    add(-1.0, 1.0) == 0.0

Description: "function to get a user by email"
Specification:
  Function: get_user_by_email(email: str) -> User | None
  Constraints:
    - Returns None if no user with that email exists
    - Raises ValueError if email is not a valid email address
    - Email comparison is case-insensitive
  Examples:
    get_user_by_email("Alice@Example.com") -> same result as get_user_by_email("alice@example.com")
    get_user_by_email("notfound@example.com") -> None

Description: "function to assign a task to a user"
Specification:
"""
```

By showing two worked examples, the model learns the exact format and level of detail expected before it generates the third specification.

### 6.3.3 Chain-of-Thought Prompting

Chain-of-thought (CoT) prompting encourages the model to reason step by step before producing its final answer ([Wei et al., 2022](https://arxiv.org/abs/2201.11903)). For complex code generation tasks, this reduces errors by forcing the model to plan before implementing.

```python
cot_prompt = """
Implement the following function. Before writing any code, think through:
1. What are the edge cases I need to handle?
2. What invariants must hold throughout the function?
3. What is the simplest correct implementation?

Then write the implementation.

Function to implement:
  assign_task(task_id: UUID, assignee_email: str, assigned_by: User) -> Task
  
  Constraints:
  - The task must exist; raise TaskNotFoundError if not
  - The assignee must be a member of the task's project; raise NotProjectMemberError if not
  - Only users with role MANAGER or ADMIN may assign tasks; raise PermissionError if not
  - The assignee cannot be the same as assigned_by
  - Update the task's assignee field and set assigned_at to the current UTC time
  - Return the updated Task object
"""
```

### 6.3.4 Self-Consistency

Self-consistency involves generating multiple independent responses to the same prompt and selecting the most common answer — or using the responses to identify where the model is uncertain ([Wang et al., 2022](https://arxiv.org/abs/2203.11171)).

For code generation, a practical application is generating the same function multiple times with slightly different temperatures and comparing the results:

```python
import anthropic

client = anthropic.Anthropic()


def generate_with_consistency_check(specification: str, n: int = 3) -> list[str]:
    """Generate n independent implementations and return them for comparison."""
    results = []
    for _ in range(n):
        response = client.messages.create(
            model="claude-opus-4-7",
            max_tokens=1024,
            messages=[{"role": "user", "content": specification}],
        )
        results.append(response.content[0].text)
    return results
```

If all three implementations agree on structure and logic, you can have higher confidence in the result. If they diverge significantly, that is a signal that the specification is under-constrained.

---

## 6.4 Writing Precise AI Specifications

An *AI specification* is a prompt written specifically to produce code generation. It differs from a requirements document in that it is optimised for a single, bounded task rather than a system-level description.

### 6.4.1 The Specification Template

A consistent template reduces ambiguity and ensures you cover the necessary elements:

```
## Task
[One sentence describing the function's purpose]

## Function Signature
[Exact Python type-annotated signature]

## Context
[Where does this function fit? What class or module does it belong to?
What existing types does it use?]

## Behaviour
[Bullet list of what the function must do in the normal case]

## Error Handling
[What errors should be raised, and when? Include the exact exception type.]

## Constraints
[What must the function NOT do? Style requirements? Library restrictions?]

## Examples
[3–5 input-output pairs covering normal cases and edge cases]
```

### 6.4.2 Worked Example

```
## Task
Filter a list of tasks by optional criteria, returning only the tasks that match all provided filters.

## Function Signature
def filter_tasks(
    tasks: list[Task],
    *,
    status: str | None = None,
    priority: int | None = None,
    assignee: str | None = None,
) -> list[Task]:

## Context
Part of src/task_service.py. Task is a dataclass defined as:
  @dataclass
  class Task:
      id: UUID
      title: str
      priority: int        # 1 (low) to 4 (critical)
      status: str          # "open", "in_progress", "completed", "cancelled"
      assignee: str | None  # email address or None

## Behaviour
- Returns all tasks in the input list if no filters are provided
- When a filter is provided, returns only tasks where the corresponding
  field exactly matches the filter value
- Multiple filters are ANDed: all provided filters must match

## Error Handling
- Raises TypeError with message "tasks must be a list" if tasks is not a list
- Raises ValueError with message "priority must be 1–4" if priority is
  provided and not in range(1, 5)
- Raises ValueError with message "status must be one of: ..." if status is
  provided and not one of the valid statuses

## Constraints
- Must NOT modify the input list
- Must return a new list (not a generator or iterator)
- Do not use external libraries

## Examples
tasks = [
    Task(id=uuid4(), title="T1", priority=2, status="open",       assignee="a@x.com"),
    Task(id=uuid4(), title="T2", priority=3, status="in_progress", assignee="b@x.com"),
    Task(id=uuid4(), title="T3", priority=2, status="open",       assignee=None),
]

filter_tasks(tasks)                          -> [T1, T2, T3]  (no filters)
filter_tasks(tasks, status="open")           -> [T1, T3]
filter_tasks(tasks, priority=2)              -> [T1, T3]
filter_tasks(tasks, status="open", priority=2) -> [T1, T3]
filter_tasks(tasks, assignee="a@x.com")      -> [T1]
filter_tasks([])                             -> []
filter_tasks(tasks, priority=0)              -> raises ValueError
filter_tasks("not a list")                   -> raises TypeError
```

This specification is self-contained, unambiguous, and includes sufficient examples to verify the generated output without any further clarification.

---

## 6.5 Context Engineering

Context engineering is the practice of deciding what information to include in a prompt — and what to leave out. In the Anthropic API, context includes the system prompt, conversation history, and any file contents or documentation passed in the user turn.

### 6.5.1 What to Include

**Always include:**
- The interface the function must implement (type-annotated signature)
- Definitions of any custom types the function uses or returns
- The error types it should raise (including their constructors)
- The specific constraints and edge cases that matter

**Include when relevant:**
- The module or class the function will be part of
- Related functions it will call (their signatures only, not implementations)
- Security requirements (parameterised queries, no eval, no shell injection)
- Performance requirements (must complete in O(n), must not load the full dataset into memory)

**Include sparingly:**
- Long existing implementations — truncate to signatures and docstrings
- Full file contents — only when the function must integrate tightly with existing code

### 6.5.2 What to Omit

- Unrelated modules and files
- Implementation details of functions the new function does not call
- Historical context and rationale (keep this in commit messages, not prompts)
- Redundant information (do not repeat the same constraint three ways)

### 6.5.3 The Context Budget

Every model has a maximum context window (measured in tokens). Claude claude-opus-4-7 supports up to 200,000 tokens of context ([Anthropic, 2024](https://docs.anthropic.com/en/docs/about-claude/models)), but larger contexts are slower and more expensive. More importantly, research suggests that models attend less reliably to information in the middle of very long contexts — the "lost in the middle" phenomenon ([Liu et al., 2023](https://arxiv.org/abs/2307.03172)).

Practical guideline: keep your specification prompts under 2,000 tokens for most code generation tasks. If you need more context than this, the task is probably too large for a single generation step — break it down.

---

## 6.6 Common Failure Modes and Fixes

| Failure Mode | Symptom | Fix |
|---|---|---|
| **Vague prompt** | Generated code does one plausible interpretation of many | Replace vague verbs with precise ones; add examples |
| **Missing constraints** | Generated code violates an implicit rule | Enumerate all constraints explicitly, including security |
| **Hallucinated API** | Generated code calls non-existent methods | Provide exact function signatures from your codebase |
| **Overlong context** | Generated code addresses the wrong part of the prompt | Trim context to only what is directly needed |
| **Underspecified errors** | Generated code returns `None` instead of raising | Specify exact exception types and conditions |
| **Style mismatch** | Generated code does not follow project conventions | Add style rules to system prompt |
| **Lost in the middle** | Model ignores critical constraints buried mid-prompt | Put the most important constraints first and last |

---

## 6.7 Tutorial: Iterative Prompt Design for the Course Project

This tutorial demonstrates the full Spec → Generate → Evaluate → Refine cycle on a real feature from the Task Management API: the `get_overdue_tasks` function.

### The Task

From the project backlog: *"As a project manager, I want to see all overdue tasks so that I can prioritise follow-up."*

This user story requires a function that returns tasks whose due date has passed and which are not yet completed.

### Iteration 1: First Draft (Too Vague)

```python
import anthropic

client = anthropic.Anthropic()

prompt_v1 = "Write a Python function to get overdue tasks."

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    messages=[{"role": "user", "content": prompt_v1}],
)
print(response.content[0].text)
```

The model will produce *something*, but with invented behaviour: it might query a database, return a list, raise an exception — anything is plausible. The output cannot be trusted.

### Iteration 2: Adding the Specification Template

```python
import anthropic
from datetime import date
from uuid import uuid4
from dataclasses import dataclass

client = anthropic.Anthropic()

# The Task dataclass the model needs to know about
task_definition = """
from dataclasses import dataclass
from datetime import date
from uuid import UUID

@dataclass
class Task:
    id: UUID
    title: str
    priority: int        # 1 (low) to 4 (critical)
    status: str          # "open", "in_progress", "completed", "cancelled"
    due_date: date | None
    assignee: str | None  # email address or None
"""

prompt_v2 = f"""
{task_definition}

## Task
Filter a list of tasks, returning only those that are overdue.
A task is overdue if: it has a due_date, the due_date is before today,
AND its status is not "completed" or "cancelled".

## Function Signature
def get_overdue_tasks(
    tasks: list[Task],
    today: date | None = None,
) -> list[Task]:

## Behaviour
- Returns tasks where due_date < today AND status not in ("completed", "cancelled")
- If today is None, uses date.today()
- Tasks with no due_date are never overdue
- Returns empty list if no tasks are overdue
- Does NOT modify the input list
- Result is sorted by due_date ascending (most overdue first)

## Error Handling
- Raises TypeError with message "tasks must be a list" if tasks is not a list

## Constraints
- Pure function: no I/O, no database calls, no external imports
- Python 3.11 type hints throughout

## Examples
# t1: overdue open task
t1 = Task(id=uuid4(), title="T1", priority=2, status="open",
          due_date=date(2024, 1, 1), assignee=None)
# t2: overdue but completed — should NOT appear
t2 = Task(id=uuid4(), title="T2", priority=1, status="completed",
          due_date=date(2024, 1, 1), assignee=None)
# t3: not yet due
t3 = Task(id=uuid4(), title="T3", priority=3, status="open",
          due_date=date(2099, 1, 1), assignee=None)
# t4: no due date — never overdue
t4 = Task(id=uuid4(), title="T4", priority=1, status="open",
          due_date=None, assignee=None)

today = date(2024, 6, 1)
get_overdue_tasks([t1, t2, t3, t4], today=today) == [t1]
get_overdue_tasks([], today=today) == []
get_overdue_tasks("not a list") raises TypeError("tasks must be a list")
"""

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": prompt_v2}],
)
print(response.content[0].text)
```

This produces a specification-grounded implementation. Before accepting it, evaluate each example and constraint.

### Iteration 3: Fixing a Discovered Issue

After running the evaluation suite, suppose the generated code sorts by `due_date` but crashes when two tasks have the same `due_date`. Add the constraint and regenerate:

```python
# Add to Constraints section:
# Sorting: sort by due_date ascending; break ties by priority descending
# (higher priority = lower number = should appear first in the sorted result)
# Use: sorted(tasks, key=lambda t: (t.due_date, t.priority))
```

Regenerate, re-evaluate. The cycle terminates when all examples pass and all constraints are satisfied.

Regenerate and verify the fix.





<!-- ## 2.10 AI-Assisted Requirements Engineering

AI tools are beginning to enter the requirements engineering process in meaningful ways. This section examines where they add value and where their limitations matter most.

### 2.8.1 Generating Draft Requirements

Given a brief system description, large language models can generate a first draft of requirements. This can accelerate early project stages by providing a concrete starting point for teams to critique and refine.

A capable LLM will produce a reasonable draft, but AI-generated requirements should never be accepted uncritically. Common issues include:

- **Missing domain-specific constraints**: The model cannot know about regulatory requirements, legacy integrations, or organisational policies.
- **Hallucinated specificity**: Numbers (response times, user limits, data retention periods) will be plausibly invented rather than sourced from actual stakeholders.
- **Missing edge cases**: Models generate obvious scenarios and may miss the edge cases that matter most in production.
- **Lack of traceability**: AI-generated requirements have no stakeholder source — a critical weakness when requirements are disputed.

### 2.8.2 Critiquing and Improving Requirements

A more reliable use of AI is as a *critic* rather than an *author*. Given a set of human-written requirements, an LLM can check for common quality issues:

```python
import anthropic

client = anthropic.Anthropic()

requirements_document = """
1. The system shall be fast.
2. Users can create tasks.
3. The system should be secure.
4. Tasks can be assigned to users.
5. The system shall send notifications.
"""

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": f"""Review the following requirements for quality issues.
For each requirement, identify if it is: ambiguous, incomplete, or unverifiable.
Suggest a rewritten version that addresses any issues.

Requirements:
{requirements_document}""",
        }
    ],
)

print(response.content[0].text)
```

This use of AI is lower risk because the human author retains ownership of the requirements and uses AI feedback as one input among several.

### 2.8.3 Generating User Stories from Interview Notes

Stakeholder interviews produce messy, unstructured notes. AI can help transform these into structured user stories:

```python
import anthropic

client = anthropic.Anthropic()

interview_notes = """
Spoke with Sarah (project manager, 12-person dev team).
Main frustrations: can't see at a glance who is overloaded,
tasks fall through the cracks when someone is sick,
no way to see what was completed last week for stand-ups.
Wants to reassign tasks quickly when priorities change.
Spends 30 mins every Monday just updating statuses in a spreadsheet.
"""

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": f"""You are a requirements engineer. Convert the following
stakeholder interview notes into well-structured user stories using the format:
"As a [role], I want [goal] so that [reason]."
Include 2–3 acceptance criteria per story in Gherkin format.
Only generate stories directly supported by the notes — do not invent requirements.

Interview notes:
{interview_notes}""",
        }
    ],
)

print(response.content[0].text)
```

The final instruction — "do not invent requirements" — is critical. Without it, models will helpfully generate plausible but unsupported requirements, which can mislead the team.

### 2.8.4 Where AI Cannot Replace Human Judgment

Despite these capabilities, requirements engineering remains fundamentally a human activity. AI cannot:

- **Resolve political conflicts** between stakeholders with competing interests
- **Understand organisational context** — why a constraint exists, who has authority to waive it, what previous attempts failed
- **Elicit tacit knowledge** — the workarounds, unspoken norms, and tribal knowledge that experienced users hold
- **Accept legal accountability** for requirements in regulated domains (healthcare, finance, aviation)
- **Build trust** with stakeholders — the relationship component that makes people willing to share their real problems

The engineer's role is not to be replaced by AI, but to use AI for routine structural tasks (formatting, checking, drafting) while investing their own time in the high-value, human-dependent work.

---

## 2.11 Tutorial: Requirements Review Pipeline

This tutorial demonstrates a practical requirements quality review pipeline using the Anthropic API.

### Setup

```bash
pip install anthropic python-dotenv
```

```bash
# .env
ANTHROPIC_API_KEY=your_key_here
```

### Requirements Review Pipeline

```python
# requirements_review.py
import os
import anthropic
from dotenv import load_dotenv

load_dotenv()

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])


def review_requirement(requirement: str) -> dict[str, str]:
    """Review a single requirement and return issues, improvement, and verdict."""
    response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=512,
        messages=[
            {
                "role": "user",
                "content": f"""Analyse this software requirement for quality issues.
Check for: ambiguity, lack of measurability, incompleteness, missing edge cases.

Requirement: "{requirement}"

Respond in this exact format:
ISSUES: [list specific issues, or "None" if well-formed]
IMPROVED: [rewritten version, or original if no issues]
VERDICT: [GOOD / NEEDS_IMPROVEMENT / POOR]""",
            }
        ],
    )

    text = response.content[0].text
    result: dict[str, str] = {}
    for line in text.strip().split("\n"):
        for key in ("ISSUES", "IMPROVED", "VERDICT"):
            if line.startswith(f"{key}:"):
                result[key.lower()] = line[len(key) + 1 :].strip()
    return result


def review_requirements_document(requirements: list[str]) -> None:
    """Review a list of requirements and print a quality report."""
    print("=" * 60)
    print("REQUIREMENTS QUALITY REVIEW")
    print("=" * 60)

    scores: dict[str, int] = {"GOOD": 0, "NEEDS_IMPROVEMENT": 0, "POOR": 0}

    for i, req in enumerate(requirements, 1):
        print(f"\n[{i}] {req}")
        result = review_requirement(req)
        verdict = result.get("verdict", "UNKNOWN")
        scores[verdict] = scores.get(verdict, 0) + 1

        print(f"    Verdict:  {verdict}")
        if verdict != "GOOD":
            print(f"    Issues:   {result.get('issues', 'N/A')}")
            print(f"    Improved: {result.get('improved', 'N/A')}")

    print("\n" + "=" * 60)
    print("SUMMARY")
    for label, count in scores.items():
        print(f"  {label:<22} {count}")
    print("=" * 60)


if __name__ == "__main__":
    sample_requirements = [
        "The system shall be fast.",
        "The system shall respond to 95% of API requests within 200ms "
        "under a load of 1,000 concurrent users.",
        "Users can create tasks.",
        "The system shall allow authenticated users to create tasks with a title "
        "(required, max 200 characters), description (optional, max 2,000 characters), "
        "due date (optional), and priority level (low, medium, high, critical).",
        "The system should be secure.",
        "All user passwords shall be stored as bcrypt hashes with a work factor of at least 12.",
    ]

    review_requirements_document(sample_requirements)
```

**Sample output:**

```
============================================================
REQUIREMENTS QUALITY REVIEW
============================================================

[1] The system shall be fast.
    Verdict:  POOR
    Issues:   Ambiguous and unverifiable — "fast" has no defined threshold
    Improved: The system shall respond to 95% of API requests within 200ms
              under a load of 1,000 concurrent users.

[2] The system shall respond to 95% of API requests within 200ms...
    Verdict:  GOOD

[3] Users can create tasks.
    Verdict:  NEEDS_IMPROVEMENT
    Issues:   Incomplete — missing subject (who), authentication context,
              and field constraints
    Improved: The system shall allow authenticated users to create tasks...
```

<!-- ## 3.7 AI-Assisted Design

AI tools can accelerate the design phase in several ways, but each requires critical evaluation.

### 3.7.1 Generating Architecture Diagrams from Specifications

Given a requirements document, an LLM can suggest an initial architecture:

```python
import anthropic

client = anthropic.Anthropic()

requirements = """
System: Task Management API
- Multi-tenant SaaS for software teams (10–500 users per tenant)
- REST API backend; no frontend in scope
- Tasks can be created, assigned, updated, and completed
- Email notifications on assignment
- Must support 1,000 concurrent users, 200ms p95 response time
- Data must be isolated per tenant
"""

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": f"""You are a software architect.
Based on the following requirements, suggest an appropriate
architectural pattern and explain your reasoning. Identify
the key components and their responsibilities.
Flag any requirements that represent significant architectural risk.

Requirements:
{requirements}""",
        }
    ],
)

print(response.content[0].text)
```

The output is a starting point for discussion, not a final decision. Treat it as a first draft from a knowledgeable junior architect who has not seen your organisation's constraints.

### 3.7.2 Generating Code Scaffolds

AI excels at generating boilerplate code from a class diagram or interface definition:

```python
response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=2048,
    messages=[
        {
            "role": "user",
            "content": """Generate a Python implementation of a TaskRepository
using the Repository pattern. The concrete implementation should use
a plain dictionary as an in-memory store (for testing).
Use Python 3.11 type hints throughout. Include docstrings only where
the behaviour is non-obvious. Follow PEP 8.""",
        }
    ],
)
```

Always review AI-generated scaffolds for:
- Correct use of type hints
- Adherence to the interface contract
- Missing edge cases (null handling, empty collections)
- Security issues (SQL injection if a DB implementation is generated)

---

## 3.8 Tutorial: AI-Assisted System Design

This tutorial walks through using AI to assist with the design of the course project API, then critically reviewing the output.

### Step 1: Generate a Component Design

```python
# design_assistant.py
import anthropic

client = anthropic.Anthropic()


def generate_component_design(requirements: str) -> str:
    response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=2048,
        messages=[
            {
                "role": "user",
                "content": f"""You are a software architect designing a Python
REST API. Based on the requirements below, produce:
1. A list of the main components (services, repositories, models)
2. The key interface (method signatures) for each component
3. A brief rationale for any significant design decision

Use Python 3.11 type hints in all interface definitions.
Do not generate implementation code — interfaces only.

Requirements:
{requirements}""",
            }
        ],
    )
    return response.content[0].text


requirements = """
Task Management API:
- Users can create, read, update, and delete tasks
- Tasks belong to projects; projects belong to organisations
- Tasks have: title, description, due_date, priority, status, assignee
- Project managers can assign tasks; regular users can only update their own
- Email notification sent when a task is assigned
- All endpoints require JWT authentication
"""

design = generate_component_design(requirements)
print(design)
```

### Step 2: Critically Review the Output

When reviewing AI-generated design, ask:

1. **Does each component have a single responsibility?** If a service is described as doing X, Y, *and* Z, it needs to be split.
2. **Are dependencies pointing the right direction?** High-level business logic should not depend on low-level infrastructure.
3. **Is the interface testable?** Can you write a test without a real database or email server?
4. **Are edge cases represented?** What happens when a task is assigned to a user who has left the project?
5. **Is the interface consistent?** Do all repository methods follow the same conventions?

Document your review findings and revise the design before implementing.
 -->
