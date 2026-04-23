# Chapter 7: Generate — Using AI Coding Agents to Build Software

*"Generation is a commodity. Knowing what to ask for, how to break it down, and when to intervene — that is the skill."*

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain why effective prompting, not raw generation, is the bottleneck in AI-native development.
2. Translate a specification into an effective prompt for code generation.
3. Decompose large features into agent-sized tasks using dependency-ordered breakdowns.
4. Work with AI coding agents hands-on: read and steer agent plans, and decide when to intervene.
5. Manage multi-file and cross-cutting changes using agents without losing coherence.
6. Generate test stubs, documentation, OpenAPI schemas, and scaffolding from specifications.
7. Recognise and avoid common generation anti-patterns.

---

## 7.1 Generation is a Commodity — Prompting is the Skill

In 2023, generating syntactically correct Python for a CRUD endpoint was a novelty. By 2025, it is routine. Every major IDE ships with an AI assistant. Every cloud provider offers a code-generation API. The ability to produce code on demand is no longer a differentiator — it is a baseline expectation, like having a compiler.

What this shift does not change is the harder problem: knowing *what* code to write, ensuring it satisfies requirements, and verifying that it integrates correctly with the rest of the system. These tasks remain squarely in the engineer's domain. The bottleneck has moved upstream, from implementation to specification and verification.

Consider the analogy of a highly capable contractor. You can hire a contractor who is faster and more precise than any previous generation — someone who can frame a wall in minutes rather than hours. But if you hand that contractor a vague instruction like "build me a room," you will get a room. It may be the wrong size. The doors may open the wrong way. The electrical outlets may be on the wrong wall. The contractor is not at fault; the specification was incomplete. The more capable the contractor, the more important the instructions become, because a capable contractor will confidently execute a bad plan with great speed and precision.

AI coding agents behave the same way. Claude, GPT-4, and Gemini will generate code immediately, fluently, and with apparent confidence regardless of whether the prompt is good. A vague prompt produces plausible-looking code that fails at the edges. A precise prompt, grounded in a concrete specification, produces code that matches the intended behaviour. The difference is not the model — it is the engineer operating it.

This is what "generation is a commodity" means in practice. The raw act of producing code tokens is cheap and abundant. The scarce resource is the judgment to decompose a feature correctly, write a specification that captures edge cases, construct a prompt that communicates the spec unambiguously, and verify that the output actually satisfies the requirements. These are engineering skills, not AI skills. AI-native software engineering does not replace engineering judgment; it moves that judgment to the front of the workflow, where it can do the most good.

The practical implication: spend more time on your specification and your prompt than you spend reviewing the output. A well-specified prompt will produce output that requires minimal correction. A vague prompt will produce output that requires as much work to fix as it would have taken to write from scratch — and fixing AI-generated code that almost works is often harder than writing correct code from a blank file, because you are reasoning about someone else's decisions rather than your own.

---

## 7.2 How to Prompt for Code Generation

### From Specification to Prompt

The specification is the source of truth for what the code must do. The prompt is the vehicle that delivers the specification to the model, wrapped with instructions about how to interpret it and what to produce. These are two distinct artefacts, and conflating them causes most prompting failures.

A specification answers: *what* should the function do, *what* are its inputs and outputs, *what* are the constraints, *what* are the error cases, and *what* are the edge cases? A prompt answers: what role should the model take, what format should the output be in, what conventions should it follow, and here is the specification to implement.

The translation step from spec to prompt is explicit: take each requirement from the specification and embed it in the user message. Do not paraphrase. Do not summarise. If the spec says "raise `ValueError` if `title` is empty or None," the prompt should say exactly that. Models are literal; they implement what you say, not what you meant.

### One-Shot vs. Iterative Generation

One-shot generation — a single prompt that produces the final output — works well when the task is well-specified, self-contained, and small enough that the model can hold the entire context in its working attention. Leaf functions (functions with no dependencies on other functions you are simultaneously writing), data models, utility functions, and schema definitions are good one-shot candidates.

Iterative generation — a sequence of prompts that progressively refine the output — is appropriate for complex logic with multiple interacting constraints, exploratory design where the right structure is not yet known, or any case where the first output reveals a misunderstanding that must be corrected before proceeding. Do not iterate by saying "fix it." Iterate by diagnosing the gap between the output and the specification, updating either the prompt or the spec, and regenerating with the correction explicit.

### Steering with Examples and Negative Constraints

Two of the most effective steering techniques are positive examples and negative constraints.

A positive example shows the model the pattern you expect: "The function signature should look like `def assign_task(task_id: int, user_id: int, db: Session) -> Task:`." This is more reliable than describing the pattern in prose because it removes ambiguity about naming conventions, type annotation style, and parameter order.

A negative constraint explicitly rules out patterns you do not want: "Do NOT use `Optional[X]`; use `X | None` instead." "Do NOT catch bare `Exception`; catch only the specific exceptions listed." "Prefer early returns over nested conditionals." Negative constraints are particularly important when the model has a strong prior toward a pattern that is wrong for your project — for example, if your codebase uses a particular ORM pattern that differs from what the model sees most often in training data.

### System Prompt vs. User Prompt

The system prompt establishes the model's role, the project's conventions, and the invariants that apply to every generation in this session. It does not change per task. The user prompt carries the specific task: the function to implement, the spec, the constraints specific to this generation.

A well-structured system prompt for a code generation session might include: the language and version, the project's style guide, the libraries in use, the error handling conventions, the type annotation style, and any project-specific patterns. This front-loads context that would otherwise need to be repeated in every user prompt.

### Before and After: A Concrete Example

Consider the task of implementing `assign_task`. Here is a vague prompt:

> Write a function that assigns a task to a user.

Here is what the same task looks like as a well-structured prompt:

**System prompt:**
```
You are a Python backend engineer implementing functions for a task management API.
Project conventions:
- Python 3.12; use X | None instead of Optional[X]
- SQLAlchemy ORM with a Session object passed as `db`
- Raise ValueError for invalid business logic; raise LookupError if an entity is not found
- Type-annotate all parameters and return values
- No print statements; no logging in library functions
- Return the modified ORM object, not a dict
```

**User prompt:**
```
Implement `assign_task` according to the following specification.

Function signature: assign_task(task_id: int, user_id: int, db: Session) -> Task

Behaviour:
1. Fetch the task by task_id from the database. If not found, raise LookupError("Task {task_id} not found").
2. Fetch the user by user_id from the database. If not found, raise LookupError("User {user_id} not found").
3. If the task status is "completed", raise ValueError("Cannot assign a completed task").
4. Set task.assignee_id = user_id and task.updated_at = datetime.utcnow().
5. Commit the session and refresh the task object.
6. Return the updated task.

Do NOT fetch any additional related objects. Do NOT send notifications. Do NOT use try/except.
```

The improvements: role and conventions are in the system prompt (not repeated per call); the function signature is exact; each behaviour is numbered and unambiguous; error messages are literal; negative constraints rule out common failure modes. The model has no decisions left to make except translating this into Python.

Here is the corresponding SDK call:

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM_PROMPT = """You are a Python backend engineer implementing functions for a task management API.
Project conventions:
- Python 3.12; use X | None instead of Optional[X]
- SQLAlchemy ORM with a Session object passed as `db`
- Raise ValueError for invalid business logic; raise LookupError if an entity is not found
- Type-annotate all parameters and return values
- No print statements; no logging in library functions
- Return the modified ORM object, not a dict
- Output only the function implementation, no explanation, no markdown fences"""

USER_PROMPT = """Implement `assign_task` according to the following specification.

Function signature: assign_task(task_id: int, user_id: int, db: Session) -> Task

Behaviour:
1. Fetch the task by task_id. If not found, raise LookupError(f"Task {task_id} not found").
2. Fetch the user by user_id. If not found, raise LookupError(f"User {user_id} not found").
3. If task.status == "completed", raise ValueError("Cannot assign a completed task").
4. Set task.assignee_id = user_id and task.updated_at = datetime.utcnow().
5. Commit the session and refresh the task object.
6. Return the updated task.

Do NOT fetch additional related objects. Do NOT use try/except."""


def generate_function(system: str, user: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system=system,
        messages=[{"role": "user", "content": user}],
    )
    return response.content[0].text


if __name__ == "__main__":
    code = generate_function(SYSTEM_PROMPT, USER_PROMPT)
    print(code)
```

---

## 7.3 Working with AI Coding Agents: Hands-On

### What an Agent Session Looks Like

When you work with an AI coding agent — whether Claude Code in the terminal, Cursor in an IDE, or Copilot Workspace in a browser — the interaction is qualitatively different from a single API call. An agent session involves multiple steps: the agent reads your instruction, forms a plan, executes steps that may include reading files, writing files, running commands, and querying search tools, and then reports its results. The engineer's role in this loop is not passive.

In Claude Code, a typical session begins with a natural-language instruction. The agent announces its plan before executing it. You see something like:

```
I'll implement email validation for the user registration endpoint. Here's my plan:

1. Read the current User model in models/user.py to understand the schema.
2. Read the registration endpoint in api/routes/auth.py.
3. Add an `email` field validator using the `email-validator` library.
4. Update the Pydantic schema in schemas/user.py to include email format validation.
5. Add a unit test in tests/test_auth.py covering valid and invalid email inputs.
6. Run the existing test suite to confirm nothing is broken.

Shall I proceed?
```

This plan is the critical intervention point. Read it carefully before saying yes.

### Reading an Agent's Plan

A good plan is specific about files, specific about what changes in each file, and ordered so that dependencies are respected. Warning signs in a plan include:

- **Vague steps** ("update the relevant files") — the agent does not know which files are relevant, which means it will guess.
- **Missing dependencies** — if the plan updates a Pydantic schema after it has already updated the endpoint that uses that schema, the endpoint change may reference fields that do not yet exist.
- **Overreach** — the plan touches files you did not ask it to touch. This is often a sign the agent is solving a problem it invented rather than the problem you specified.
- **Missing verification** — a good plan ends with running tests. A plan that does not include verification will produce unverified code.

### The Three Intervention Points

**Before execution starts:** This is your highest-leverage moment. If the plan is wrong, the execution will be wrong. Read every step. Ask yourself: does this step correspond to something in my specification? Does the order respect dependencies? Are there steps missing? If yes to any of these, correct the plan before proceeding. In Claude Code, you can reply with corrections directly: "Before step 3, read tests/test_user.py so you understand the existing test structure. Do not add the `email-validator` library — use the stdlib `re` module with the pattern in the spec."

**During execution:** Watch the agent's output as it runs. Modern agents stream their progress. If an agent writes a file that does not match your expectations, interrupt immediately. The longer a wrong direction runs, the more files it touches, and the harder it is to unwind. In Claude Code, Ctrl-C interrupts the current step and returns control. You can then redirect the agent with a specific correction that references what you observed.

**After execution:** Verify the output against the specification. This is not optional. Read the generated code. Run the tests. Check every requirement in the spec against the implementation. A useful checklist after any agent generation: (1) Does the function signature match the spec? (2) Does each error case in the spec have a corresponding branch in the code? (3) Do the tests cover the error cases, not just the happy path? (4) Does the code respect the project's style conventions?

### Walkthrough: "Add email validation to the user registration endpoint"

You give the agent this instruction. The agent's plan lists five steps, including reading the current auth route and adding validation. You notice the plan does not mention the Pydantic schema for the registration request body — the agent plans to add validation directly in the route handler rather than in the schema layer. This is a design deviation. Before execution starts, you correct it: "Step 3 should add an `EmailStr` field to `schemas/user.py:UserCreate` rather than validating inline in the route handler. The route handler should not contain validation logic."

During execution, the agent reads `schemas/user.py` and adds `email: EmailStr`. You confirm this is correct. The agent then modifies the route handler — you check that it references `user_data.email` from the schema rather than accessing `request.body` directly. Correct.

After execution, you run `pytest tests/test_auth.py -v`. Three tests pass, but the test for a malformed email is not present — the agent wrote a test for a missing email field but not for a syntactically invalid email. You note the gap in coverage and add it to the next task.

---

## 7.4 Task Decomposition Strategies

### Why Decomposition Matters

Agent reliability degrades non-linearly with task complexity. A task that touches one function in one file is almost always completed correctly. A task that touches five functions across three files has multiple opportunities for the agent to introduce an inconsistency — using a field name that does not match the model, calling a function with the wrong signature, or writing a test that imports a path that no longer exists. Research on LLM coding agents consistently shows that task success rates drop sharply as the number of files and functions involved increases ([SWE-bench, 2024](https://www.swebench.com/)).

The practical response is to decompose every feature before you begin generation. Decomposition is not overhead — it is the work that makes generation reliable.

### The Feature → File → Function Breakdown

Start at the feature level: what is the user-visible behaviour being added? Then identify which files change to implement that behaviour. For each file, identify which functions are added or modified. This three-level breakdown gives you a list of atomic generation tasks, each of which is small enough for an agent to handle correctly.

**Dependency ordering** is critical. Generate leaf functions before callers. Generate data models before the services that use them. Generate services before the endpoints that call them. Generate tests after the functions they test, but keep them in the same generation session so the agent knows what it just implemented.

### Example: Decomposing "Add Task Commenting Feature"

Feature: Users can add comments to tasks. Comments have an author, a body, and a created timestamp. The API exposes endpoints to create and list comments for a task.

| Task | Depends On | File | Estimated Complexity |
|---|---|---|---|
| 1. Define `Comment` ORM model | None | `models/comment.py` | Low |
| 2. Create `CommentCreate` and `CommentRead` Pydantic schemas | Task 1 | `schemas/comment.py` | Low |
| 3. Write Alembic migration for `comments` table | Task 1 | `migrations/versions/xxx_add_comments.py` | Low |
| 4. Implement `create_comment(task_id, user_id, body, db)` | Tasks 1, 2 | `services/comment_service.py` | Medium |
| 5. Implement `list_comments(task_id, db)` | Task 1 | `services/comment_service.py` | Low |
| 6. Write unit tests for service functions | Tasks 4, 5 | `tests/test_comment_service.py` | Medium |
| 7. Add `POST /tasks/{task_id}/comments` endpoint | Tasks 2, 4 | `api/routes/comments.py` | Medium |
| 8. Add `GET /tasks/{task_id}/comments` endpoint | Tasks 2, 5 | `api/routes/comments.py` | Low |
| 9. Register comment router in `api/main.py` | Tasks 7, 8 | `api/main.py` | Low |
| 10. Write integration tests for comment endpoints | Tasks 7, 8, 9 | `tests/test_comment_endpoints.py` | Medium |

You execute these tasks in order, verifying after each one before proceeding to the next.

### Using AI to Produce the Decomposition

Before implementation begins, you can use the API to generate the decomposition itself. Let the model do the structural analysis, then review and correct the output before committing to the order.

```python
import anthropic
import json

client = anthropic.Anthropic()

DECOMPOSITION_SYSTEM = """You are a senior software architect. Given a feature description for a
Python FastAPI + SQLAlchemy project, produce a dependency-ordered list of implementation tasks.

Output a JSON array. Each element has:
- task_number: int
- description: str (one sentence, imperative)
- depends_on: list[int] (task numbers this task depends on; [] for none)
- file: str (relative file path)
- complexity: "Low" | "Medium" | "High"

Order tasks so that dependencies always appear before dependents.
Output only the JSON array, no explanation."""


def decompose_feature(feature_description: str) -> list[dict]:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system=DECOMPOSITION_SYSTEM,
        messages=[
            {
                "role": "user",
                "content": f"Feature: {feature_description}\n\nProject structure: FastAPI app, "
                           f"SQLAlchemy ORM, Pydantic v2 schemas, Alembic migrations, pytest tests.",
            }
        ],
    )
    raw = response.content[0].text.strip()
    return json.loads(raw)


if __name__ == "__main__":
    tasks = decompose_feature(
        "Add task commenting feature: users can create and list comments on a task. "
        "Each comment has an author (user_id), a non-empty body (max 2000 chars), "
        "and a created_at timestamp. Comments cannot be edited or deleted."
    )
    print(f"{'#':<4} {'Description':<55} {'Deps':<12} {'File':<40} {'Complexity'}")
    print("-" * 125)
    for t in tasks:
        deps = str(t["depends_on"]) if t["depends_on"] else "[]"
        print(f"{t['task_number']:<4} {t['description']:<55} {deps:<12} {t['file']:<40} {t['complexity']}")
```

Review the output critically. The model may miss dependencies or place tasks in an order that seems logical but is subtly wrong. Correct it before executing.

---

## 7.5 Managing Multi-File and Cross-Cutting Changes

### The Coherence Problem

When an AI agent generates code for a multi-file change, each file is generated in a context window that may not include the other files. This creates a coherence problem: field names may differ between the model and the schema; function signatures may not match between the service and the endpoint; the test may import a symbol using a path that was reorganised in a different generation step.

The engineer is responsible for cross-file consistency. The agent handles local correctness within a file; you handle global correctness across files.

### Context Window Strategy

Before generating a file, decide explicitly which other files belong in the context. The rule: include all files that the new file directly imports from, all files that directly import from the new file, and the specification document. Nothing else.

If you are generating the `create_comment` service function, include `models/comment.py` (the ORM model it uses), `schemas/comment.py` (the Pydantic schema it returns), and the spec. Do not include `models/task.py` unless the comment service directly touches the task model. Including irrelevant files does not help and actively harms coherence by giving the model patterns to import that you did not intend.

### Cross-Cutting Changes

A cross-cutting change touches many files for a structural reason — adding a field to a data model, renaming a function, changing a function signature. These are the highest-risk changes in multi-file generation because every consumer of the changed artefact must be updated consistently.

The protocol for cross-cutting changes:
1. Identify all files that reference the artefact being changed. Use `grep` or your IDE's "find all references" before starting.
2. Update the definition first (the model, the function, the schema).
3. For each consumer file, include the updated definition in the context when generating the consumer's change.
4. Run the full test suite after every file change, not at the end.

### Refactoring with Agents

Always require test coverage before refactoring. If tests do not exist for the code you are about to refactor, generate them first. Then refactor. Then run the tests. This sequence is non-negotiable — refactoring without tests produces code that looks clean and may be silently wrong.

The practical rule: for any change that touches more than three files, decompose into per-file steps with a test run between each step. A failing test after a single-file change is easy to diagnose. A failing test after a seven-file change requires you to reason about which of seven changes introduced the regression.

---

## 7.6 Generating Tests, Documentation, and Scaffolding

### Test Stub Generation

AI agents are excellent at generating test structure — the scaffolding of test functions, the parametrize decorators, the fixture setup. They are less reliable at generating the specific assertion values, because assertions encode business logic that requires understanding the specification precisely.

The strategy: generate test stubs with `assert False, "TODO"` as the body, then fill in the assertions yourself. The agent handles the repetitive structural work (what test cases exist, how to parametrize, what fixtures to use). You handle the semantically meaningful work (what the function should return for this input).

```python
import anthropic

client = anthropic.Anthropic()

TEST_STUB_SYSTEM = """You are a Python test engineer. Given a function specification, generate
pytest test stubs.

Rules:
- Use pytest, not unittest
- Generate one test function per behaviour described in the spec
- Use descriptive test names: test_<function>_<scenario>
- For each test, include the setup and the function call, then end with:
  assert False, "TODO: fill in assertion"
- Use @pytest.mark.parametrize for cases that differ only in input values
- Do NOT implement assertions — only stubs
- Output only the Python code, no explanation"""


def generate_test_stubs(function_name: str, spec: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system=TEST_STUB_SYSTEM,
        messages=[
            {
                "role": "user",
                "content": f"Function: {function_name}\n\nSpecification:\n{spec}",
            }
        ],
    )
    return response.content[0].text


ASSIGN_TASK_SPEC = """
assign_task(task_id: int, user_id: int, db: Session) -> Task

Behaviour:
1. Fetches the task by task_id; raises LookupError if not found.
2. Fetches the user by user_id; raises LookupError if not found.
3. If task.status == "completed", raises ValueError("Cannot assign a completed task").
4. Sets task.assignee_id = user_id and task.updated_at to the current UTC time.
5. Commits the session and refreshes the task.
6. Returns the updated Task object.
"""

if __name__ == "__main__":
    stubs = generate_test_stubs("assign_task", ASSIGN_TASK_SPEC)
    print(stubs)
```

A typical output includes `test_assign_task_success`, `test_assign_task_task_not_found`, `test_assign_task_user_not_found`, `test_assign_task_raises_for_completed_task`, and `test_assign_task_updates_updated_at`. Each has the setup code but ends with `assert False, "TODO: fill in assertion"`. You then fill in the specific assertions — the return value checks, the exact exception messages, the timestamp comparison logic.

### Docstring Generation

```python
def generate_docstring(function_signature: str, spec: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        system=(
            "You are a Python documentation writer. Given a function signature and specification, "
            "generate a Google-style docstring. Include Args, Returns, and Raises sections. "
            "Be concise. Output only the docstring text (the triple-quoted string content), "
            "no surrounding code."
        ),
        messages=[
            {
                "role": "user",
                "content": f"Signature:\n{function_signature}\n\nSpec:\n{spec}",
            }
        ],
    )
    return response.content[0].text
```

### OpenAPI Schema and Database Migration Generation

For FastAPI route schemas, provide the function signature and Pydantic models and ask for the `@router.post(...)` decorator with the correct `response_model`, `status_code`, and summary. For Alembic migrations, describe the model change in plain English ("add nullable `email` column of type VARCHAR(320) to the `users` table") and ask for the `upgrade()` and `downgrade()` functions. In both cases, review the output — the agent will produce plausible structure, but the specific column types, constraints, and index choices require your judgment.

### The Rule: Generate Scaffolding, Own the Logic

Let the agent handle structure, and own the semantically significant decisions yourself. The test case names, the fixture setup, the parametrize decorator, the docstring format, the migration boilerplate — these are structural and repetitive. The assertion values, the migration constraints, the API contract decisions — these encode business logic and belong to you.

---

## 7.7 Generation Patterns and Anti-Patterns

| Anti-pattern | Why it fails | Better approach |
|---|---|---|
| One giant prompt for the entire feature | The agent produces incoherent multi-file output; field names drift, imports break, error handling is inconsistent | Decompose into per-function tasks using feature → file → function; generate and verify each independently |
| No spec, just a vague description | The model guesses at constraints, chooses arbitrary defaults, misses error cases | Write the specification template (signature, behaviour, error cases, edge cases) before starting any generation |
| Accepting first output without review | Studies consistently show defect rates of 30–40% for non-trivial AI-generated functions | Verify every output against the spec; use the checklist: signature match, error case coverage, style compliance |
| Regenerating after failure without diagnosing | The model will reproduce the same bug if the prompt still implies it | Identify which spec clause was violated; update the spec to make the constraint explicit |
| Including the entire codebase in context | The model's attention diffuses across irrelevant patterns; it imports utilities from unrelated modules | Include only the files that are direct dependencies or direct consumers of the code being generated |
| Asking for "best practices" without constraints | Gets generic code, not project-specific code | Specify your conventions explicitly in the system prompt |
| Treating the agent's confidence as correctness | Agents produce fluent, confident-sounding code regardless of correctness | Verification is always required; fluency is not a signal of correctness |

### When to Abandon and Re-Specify

Three signals indicate you should stop refining a generation and return to the specification:

**The output does not resemble what you wanted, despite multiple refinements.** If you have corrected the prompt three times and the agent keeps producing the same structural error, the specification is probably ambiguous in a way you have not yet identified. Stop prompting. Read the spec again. Find the word the model is interpreting differently.

**The model makes the same error repeatedly.** An error that persists across regenerations with the same prompt is diagnostic of a missing constraint. Add it explicitly. If the model keeps adding a `try/except` block you did not ask for, add "Do NOT use try/except" as an explicit negative constraint.

**The spec is the problem, not the generation.** Sometimes the first generation reveals that the specification is incomplete or contradictory. Fix the spec first — a patched implementation on top of a broken spec will fail again when the code is next modified.

---

## 7.8 Tutorial: End-to-End Generation for the Course Project

### Task

Implement `get_overdue_tasks(tasks: list[Task], today: date | None = None) -> list[Task]` for the course project task management system.

### Specification

```
Function: get_overdue_tasks

Signature: get_overdue_tasks(tasks: list[Task], today: date | None = None) -> list[Task]

Behaviour:
1. If today is None, use datetime.date.today() as the reference date.
2. Return a list containing only the tasks where:
   a. task.due_date is not None, AND
   b. task.due_date < today (strictly less than — a task due today is NOT overdue), AND
   c. task.status != "completed"
3. Preserve the order of the input list.
4. If tasks is not a list, raise TypeError("tasks must be a list").
5. If tasks is an empty list, return an empty list.
```

### Generation Call

```python
import anthropic
from datetime import date

client = anthropic.Anthropic()

SYSTEM = """You are a Python backend engineer implementing functions for a task management system.
Python 3.12. Use X | None instead of Optional[X]. Type-annotate everything.
The Task class is a dataclass with fields: id (int), title (str), status (str),
due_date (datetime.date | None), assignee_id (int | None).
Output only the function implementation. No markdown. No explanation."""

SPEC = """
Implement get_overdue_tasks according to this specification.

Signature: get_overdue_tasks(tasks: list[Task], today: date | None = None) -> list[Task]

Behaviour:
1. If today is None, use date.today() as the reference date.
2. Return tasks where: task.due_date is not None AND task.due_date < today AND task.status != "completed".
3. Preserve input order.
4. If tasks is not a list, raise TypeError("tasks must be a list").
5. If tasks is empty, return [].

Edge cases:
- A task with due_date == today is NOT overdue (use strictly less than, not less-than-or-equal).
- A completed task with a past due_date is NOT overdue.
- A task with due_date = None is NOT overdue.

Do NOT sort the output. Do NOT modify the input list.
"""

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    system=SYSTEM,
    messages=[{"role": "user", "content": SPEC}],
)
print(response.content[0].text)
```

### Verification Checklist

- [ ] `tasks` not a list raises `TypeError("tasks must be a list")`
- [ ] `today=None` defaults to `date.today()`
- [ ] Filters on `due_date is not None`
- [ ] Filters on `due_date < today` (strictly less than)
- [ ] Filters out completed tasks (`status != "completed"`)
- [ ] Preserves order
- [ ] Returns empty list for empty input
- [ ] Does not sort, does not mutate input

### One Refinement

Suppose the first output uses `<=` instead of `<` — a task due today is overdue in this version. The spec says `<` and the prompt repeats it, but the prose in the spec said "a past due_date," which is ambiguous. Update the spec's prose to match the formal condition and add the edge case explicitly. Regenerate. The corrected output uses `<`. This is the correct pattern: diagnose the ambiguity in the spec, fix the spec, regenerate. Do not just say "use `<` not `<=`" without updating the spec — the spec remains the source of truth.

---

## Chapter Summary

- **Generation is abundant; specification and verification are scarce.** The engineer's role in AI-native development is upstream: writing precise specifications, constructing effective prompts, and verifying outputs against requirements.

- **Prompts have structure.** The system prompt carries session-level conventions; the user prompt carries the specific task. Positive examples and negative constraints steer the model away from its priors and toward your project's patterns.

- **Decompose before generating.** Agent reliability degrades sharply with task size. The feature → file → function breakdown produces atomic tasks that agents can handle reliably. Dependency ordering prevents the most common multi-file coherence failures.

- **Three intervention points govern every agent session.** Before execution (is the plan correct?), during execution (is it on track?), and after execution (does it satisfy the spec?). The highest-leverage intervention is before execution starts.

- **Generate scaffolding; own the logic.** Use agents to produce test stubs, docstrings, migration boilerplate, and route schemas. Fill in the assertions, constraints, and business logic decisions yourself.

- **Anti-patterns are diagnostic.** Repeated failures at generation usually indicate a problem with the specification, not the model. When generation keeps producing the same error, stop prompting and fix the spec.

---

## Review Questions

**Question 1 (Conceptual):** Why is task decomposition important for agent reliability? Explain the relationship between task complexity, context window behaviour, and error rate. Describe at least one structural mechanism that decomposition provides beyond simply making tasks smaller.

**Question 2 (Applied):** You are asked to add a `tag` field (a nullable string, maximum 50 characters) to tasks. The field affects the `Task` ORM model, the `TaskCreate` and `TaskRead` Pydantic schemas, the task service's `create_task` function, the `POST /tasks` REST endpoint, and three existing test files. Describe your complete decomposition strategy step by step, including the order in which you would generate each change, which files you would include in the context window for each generation, and how you would verify correctness between steps.

**Question 3 (Conceptual):** What is the difference between one-shot and iterative generation? Give one concrete example of a task that is well-suited to one-shot generation and one that is better handled iteratively. For the iterative example, describe what the first prompt would produce and how you would formulate the second prompt based on that output.

**Question 4 (Applied):** An agent generates the following `create_task` function. It does not validate that the title is non-empty. Classify this as: (a) missing requirement in the spec, (b) requirement present in the spec but absent from the prompt, or (c) requirement present in the prompt but ignored by the model. Then describe how to fix both the specification and the prompt, and write the corrected constraint as it would appear in each.

```python
def create_task(title: str, assignee_id: int | None, db: Session) -> Task:
    task = Task(title=title, assignee_id=assignee_id, status="open")
    db.add(task)
    db.commit()
    db.refresh(task)
    return task
```

**Question 5 (Applied):** Write a complete prompt — system prompt and user prompt — that would reliably generate pytest test stubs for a `filter_tasks` function with this specification: accepts a list of tasks and optional keyword arguments `status`, `priority`, and `assignee_id`; returns only tasks matching all provided filters; raises `TypeError` if the input is not a list; raises `ValueError` if `priority` is provided but is not one of `"low"`, `"medium"`, `"high"`. Your prompt should specify the output format (stubs with `assert False, "TODO"`), the test naming convention, and any parametrize usage.
