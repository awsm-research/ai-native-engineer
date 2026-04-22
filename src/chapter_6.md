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

---

## 6.8 Project Milestone: AI-Ready Specifications

### This Week's Deliverables

Using the specification template from Section 6.4.1, write AI-ready specifications for **3 core features** of your Task Management API. For each specification:

1. **Full specification**: All seven sections (Task, Signature, Context, Behaviour, Error Handling, Constraints, Examples)
2. **Generation run**: Generate an implementation using the specification
3. **Evaluation log**: Record which examples and constraints the generated code satisfies and which it violates
4. **Refinement**: Make at least one refinement to the specification and regenerate; document what changed and why

Submit the specification documents alongside your evaluation logs.

---

> **Common Mistakes**
>
> **Writing behaviour as prose without examples.** "The function should handle edge cases appropriately" communicates nothing actionable. Every constraint needs a concrete example: input → expected output. Ambiguity in the spec becomes hallucination in the output.
>
> **Including the solution in the prompt.** "Write a function that loops through tasks and filters by `status == 'overdue'"` over-constrains the implementation and prevents the AI from using a more idiomatic approach. Specify *what*, not *how*.
>
> **Overloading a single prompt.** Asking one prompt to implement, document, and write tests for a function produces mediocre results for all three. Separate prompts with separate specifications produce better outputs for each.
>
> **Re-prompting instead of re-specifying.** When the AI produces wrong output, the instinct is to ask again with "try harder" or "be more careful." This rarely helps. The right response is to identify which part of the specification was under-constrained and fix it.

---

## Summary

Prompt engineering is the discipline of designing inputs that reliably produce useful outputs from language models. Key takeaways:

- Most prompt failures stem from ambiguity, missing constraints, hallucinated APIs, context overload, or over-specification.
- Core patterns — role prompting, few-shot examples, chain-of-thought, and self-consistency — each address different failure modes.
- A precise AI specification includes: the function signature, context, behaviour, error handling, explicit constraints, and worked examples.
- Context engineering decides what information to include in a prompt. Prefer lean, focused context over exhaustive context.
- The most important constraint is usually the one you forgot to write.
- Prompt refinement is an iterative process: generate, evaluate, identify the gap, add the missing constraint, regenerate.

---

## Review Questions

1. What are the five common prompt failure modes described in this chapter? For each, give a concrete example in a software engineering context.
2. Explain chain-of-thought prompting. Why does it reduce errors in code generation tasks?
3. Using the specification template from Section 6.4.1, write a complete AI specification for a function `paginate_tasks(tasks, page, page_size) -> dict` that returns a paginated subset of a task list.
4. What is the "lost in the middle" problem? How does it affect context design for code generation?
5. A colleague writes the following specification: "Write a function to get all tasks for a user." List 5 pieces of information that are missing from this specification that would affect the generated output.
6. When would you use self-consistency (generating multiple outputs) rather than a single generation? What does significant divergence between outputs tell you?

---

## References

- Anthropic. (2024). Claude API Documentation. [https://docs.anthropic.com/en/api/getting-started](https://docs.anthropic.com/en/api/getting-started)
- Anthropic. (2024). Claude Models Overview. [https://docs.anthropic.com/en/docs/about-claude/models](https://docs.anthropic.com/en/docs/about-claude/models)
- Brown, T., et al. (2020). Language Models are Few-Shot Learners. *arXiv*. [https://arxiv.org/abs/2005.14165](https://arxiv.org/abs/2005.14165)
- Liu, N. F., et al. (2023). Lost in the Middle: How Language Models Use Long Contexts. *arXiv*. [https://arxiv.org/abs/2307.03172](https://arxiv.org/abs/2307.03172)
- Wang, X., et al. (2022). Self-Consistency Improves Chain of Thought Reasoning in Language Models. *arXiv*. [https://arxiv.org/abs/2203.11171](https://arxiv.org/abs/2203.11171)
- Wei, J., et al. (2022). Chain-of-Thought Prompting Elicits Reasoning in Large Language Models. *arXiv*. [https://arxiv.org/abs/2201.11903](https://arxiv.org/abs/2201.11903)
- White, J., et al. (2023). A Prompt Pattern Catalog to Enhance Prompt Engineering with ChatGPT. *arXiv*. [https://arxiv.org/abs/2302.11382](https://arxiv.org/abs/2302.11382)
