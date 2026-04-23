# Chapter 9: Refine — Iterative Improvement with AI Coding Agents

*"The first draft is never the product. Refinement is where engineering happens."*

---

Refinement is not debugging. It is the systematic discipline of improving what has been generated and verified — closing the gap between the current codebase and the intended system, one loop at a time. In the Agentic SDLC, Refine is the phase that makes the loop productive: without it, each Generate-Verify cycle produces isolated outputs; with it, the codebase converges toward correctness, quality, and maintainability.

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Distinguish reactive patching from systematic refinement, and explain why the difference matters.
2. Diagnose why an agent produced a wrong or suboptimal result: spec problems, model limitations, or context gaps.
3. Refine prompts and specifications using constraints, counter-examples, and structured re-anchoring.
4. Apply safe refactoring patterns with AI agents, including test-coverage prerequisites and rollback strategies.
5. Use AI to identify, prioritise, and plan the resolution of technical debt.
6. Build feedback loops from evaluation results back into specifications and prompt libraries.
7. Integrate continuous refinement into CI/CD with nightly code quality checks and automated PR review agents.
8. Recognise when to stop refining and ship.

---

## 9.1 Refine is Not Fixing Bugs — It is a Discipline

Every software team fixes bugs. Most teams patch the immediate symptom: a test fails, an exception is reported, a user complains, and an engineer opens the file, changes the offending line, and closes the issue. This is reactive patching. It is necessary — you cannot ship broken software — but it is not sufficient in an agentic workflow, because the same agent with the same specification will generate the same class of bug the next time it touches that code.

Systematic refinement is different in kind, not just degree. Where reactive patching targets a single failure, systematic refinement targets the process that produced the failure. When a function fails, a systematic refinement cycle asks: does the specification adequately constrain this case? Did the prompt give the agent the context it needed? Is the agent capable of this task without further decomposition? The answers feed back into the generation process so that the next cycle produces better output without additional human intervention.

The distinction has practical consequences. Suppose an agent generates a pricing function that silently returns zero instead of raising an error when a product is not found. A reactive patch changes the return statement. A systematic refinement adds an explicit constraint to the spec — "if `product_id` is not in the catalogue, raise `ProductNotFoundError`" — and updates the prompt template for all pricing functions. The next time any pricing function is generated, the constraint is in place before the first line of code is written.

Refinement has three distinct targets, and confusing them is a common source of wasted effort:

**The specification.** Specs degrade over time through omission: new requirements are implemented without updating the formal spec, edge cases are discovered but not codified, and the spec drifts from the actual behaviour. Refining the spec means closing these gaps explicitly — not in comments, not in tests alone, but in the document that drives generation.

**The generation process.** The prompt is the agent's primary input. A prompt that lacks relevant context, uses imprecise terminology, or fails to provide worked examples will consistently produce suboptimal output regardless of how good the underlying model is. Refining the generation process means treating prompt templates as first-class engineering artefacts: versioned, reviewed, and measured.

**The codebase itself.** The generated code accumulates technical debt, inconsistencies, and patterns that made sense at the time but no longer fit the system. Refactoring — done safely, with test coverage in place — is the third target of refinement.

Think of the Spec → Generate → Verify → Refine loop as an engine. Specification defines the goal, generation produces a candidate, verification measures the gap, and refinement is the feedback path that closes it. An engine without a feedback path is open-loop: it produces output, but it does not improve. Refinement closes the loop. Each iteration makes the specification more precise, the prompts more effective, and the codebase more coherent. The loop becomes self-improving over time — and that is the engineering advantage of the Agentic SDLC.

---

## 9.2 Diagnosing Failures: Why Did the Agent Get It Wrong?

Before you can fix a failure systematically, you need to know why it happened. Agents fail for different reasons, and the remediation depends entirely on the root cause. Applying the wrong fix — rewriting a prompt when the real problem is a missing constraint in the spec — wastes time and leaves the underlying issue intact.

The following taxonomy covers the most common failure modes in AI-generated code:

| Failure category | Description | Symptom | Fix |
|---|---|---|---|
| Spec ambiguity | Spec had multiple valid interpretations; agent chose the wrong one | Code is internally consistent but wrong | Clarify the spec with concrete examples |
| Missing constraint | Spec omitted a rule; agent made a reasonable default choice | Code passes happy-path tests, fails edge cases | Add the missing constraint explicitly |
| Context gap | Agent lacked relevant context (data model, convention, existing code) | Code uses wrong patterns or invents new abstractions | Add the missing context to the prompt |
| Model limitation | Task was at the edge of the model's capability | Code is partially correct but logically flawed | Decompose the task; use a more capable model |
| Hallucinated API | Agent used a function or library that does not exist | `ImportError` or `AttributeError` at runtime | Provide the correct API reference in the spec |

These categories are not exhaustive, but they cover the vast majority of failures encountered in practice. The key discipline is to identify the category before writing any fix.

**The debugging workflow.** A structured four-step process prevents the common mistake of patching symptoms rather than causes:

1. **Reproduce the failure with a minimal failing test.** Write the test first, before touching the implementation. The test is your precise statement of what the code should do but does not. If you cannot write the test, you do not yet understand the failure.

2. **Map the code to the spec clause by clause.** Read the spec and the implementation in parallel. Mark each spec clause as satisfied or violated. If a clause is violated, you have found the failure point. If the clause is missing entirely, you have found a missing constraint.

3. **Identify the failure category from the table.** Is the spec present but ambiguous? Missing? Was the context absent from the prompt? Is the logic partially correct in a way that suggests a capability limit?

4. **Apply the appropriate fix.** A spec problem requires a spec change. A context gap requires a prompt change. A model limitation requires task decomposition. Do not apply a patch without knowing which category you are in.

**Concrete example: the missing constraint category.** Consider a function that calculates a late payment penalty:

```python
# Generated implementation
def calculate_overdue_penalty(principal: float, days_overdue: int) -> float:
    """Calculate the penalty for an overdue payment."""
    daily_rate = 0.0005  # 0.05% per day
    return principal * daily_rate * days_overdue
```

The spec read: "Calculate the overdue penalty as 0.05% of principal per day overdue." The agent followed the spec correctly. But the following test fails:

```python
def test_no_penalty_when_not_overdue():
    # If days_overdue is 0 or negative, no penalty should apply.
    assert calculate_overdue_penalty(1000.0, 0) == 0.0
    assert calculate_overdue_penalty(1000.0, -3) == 0.0  # early payment

def test_penalty_does_not_exceed_principal():
    # Penalty is capped at 100% of principal.
    assert calculate_overdue_penalty(1000.0, 3000) <= 1000.0
```

The `days_overdue=0` case returns `0.0` correctly by coincidence (multiplication by zero), but `days_overdue=-3` returns `-1.5`, and a 3000-day overdue on a £1,000 principal returns £1,500 — more than the principal itself. The failure category is **missing constraint**: the spec said nothing about negative inputs or a cap. The fix is not to patch the implementation directly; it is to update the spec:

```markdown
## Updated spec clause (calculate_overdue_penalty)

- If `days_overdue` is less than or equal to 0, return 0.0.
- The penalty shall not exceed the value of `principal`.
- The daily rate is 0.05% (0.0005) of principal per day overdue.
```

With this constraint in the spec, the next generation will handle both edge cases without requiring a manual patch. The implementation fix follows from the spec change, not the other way around.

---

## 9.3 Prompt and Specification Refinement

### The Spec as a Living Document

A specification is not a design document written once at the start of a project. It is a living artefact that evolves with the codebase. In the Agentic SDLC, the spec is also the primary driver of code generation: a stale or imprecise spec produces stale or imprecise code. Treating spec changes as second-class artefacts — informal notes, Slack messages, undocumented verbal agreements — is the single most common reason AI-generated code regresses over time.

The discipline is simple: version-control specs alongside code. The spec lives in the repository, in a `specs/` directory that mirrors the structure of `src/`. A function in `src/payments/penalties.py` has its spec in `specs/payments/penalties.md`. When you change the spec, you commit it in the same commit as the implementation change. The spec and the implementation stay in sync because the repository enforces it.

### Three Refinement Techniques

**1. Adding constraints.** The most common refinement operation. After a failure is diagnosed as a missing constraint, the constraint is written into the spec in imperative, unambiguous language. Use the form "Do NOT X; use Y instead because Z" when the agent has a predictable wrong default. The "because Z" is not optional — it tells the agent (and future engineers) why the constraint exists, which prevents it from being silently removed during future edits.

Example of a constraint addition to a spec:
```markdown
## Constraints

- Do NOT return `None` for empty result sets; return an empty list `[]` instead,
  because callers iterate the result and `None` is not iterable.
- Do NOT use `datetime.now()`; use `datetime.now(timezone.utc)` instead,
  because the system stores all timestamps in UTC.
```

**2. Adding counter-examples.** Worked negative examples are often more effective than abstract constraint statements, because they show the agent both the input that caused the failure and the expected output. Counter-examples anchor generation to specific cases and are directly usable as test inputs.

```markdown
## Counter-examples

| Input | Expected output | Common wrong output |
|-------|-----------------|---------------------|
| `calculate_overdue_penalty(1000.0, 0)` | `0.0` | `0.0` (correct by coincidence) |
| `calculate_overdue_penalty(1000.0, -3)` | `0.0` | `-1.5` (wrong: negative days) |
| `calculate_overdue_penalty(1000.0, 3000)` | `1000.0` | `1500.0` (wrong: exceeds principal) |
```

**3. Re-anchoring after drift.** In multi-step generation — where an agent generates a module incrementally over several prompts — the generated code can drift from the original intent as context accumulates. The agent starts solving the most recent sub-problem rather than the overall goal. The fix is explicit re-anchoring: at the start of each new generation prompt, restate the core goal in one sentence before providing the new task. "This function is part of the payment penalty system. Its sole responsibility is X. With that context, implement Y."

### The Constraint Accumulation Pattern

Start with a minimal spec. Generate. Find the failure. Add the failing case as an explicit constraint. Repeat. This pattern produces specs that are empirically grounded in actual failure modes rather than speculatively exhaustive — which means every constraint in the spec was put there because it was needed.

Before (initial spec for `calculate_overdue_penalty`):
```markdown
# Spec: calculate_overdue_penalty

Calculate the overdue penalty as 0.05% of principal per day overdue.

Parameters:
- principal (float): the outstanding amount
- days_overdue (int): number of days past the due date
```

After three refinement iterations:
```markdown
# Spec: calculate_overdue_penalty
# Version: 1.3 | Last updated: 2026-04-23

Calculate the overdue penalty as 0.05% of principal per day overdue.

Parameters:
- principal (float): the outstanding amount; must be >= 0
- days_overdue (int): number of days past the due date

Constraints [added in v1.1]:
- If days_overdue <= 0, return 0.0 (no penalty for on-time or early payment).

Constraints [added in v1.2]:
- The returned penalty must not exceed principal (cap at 100%).

Constraints [added in v1.3]:
- If principal <= 0, raise ValueError("principal must be positive").

Counter-examples:
- calculate_overdue_penalty(1000.0, -3) → 0.0
- calculate_overdue_penalty(1000.0, 3000) → 1000.0
- calculate_overdue_penalty(0.0, 10) → raises ValueError
```

### The Prompt Library

A prompt library is a shared team repository of tested, effective prompt templates for common task types. Each entry specifies the template, a worked example, and known failure modes to avoid. The library is version-controlled and reviewed like any other shared codebase asset.

A simple YAML format for prompt library entries:

```yaml
# prompt_library/implement_pure_function.yaml
id: implement_pure_function_v2
task_type: function_implementation
description: Generate a pure function from a formal spec
template: |
  You are implementing a Python function for a production codebase.
  Coding conventions: {conventions_file}
  
  Specification:
  {spec_content}
  
  Requirements:
  - Implement only the function described in the spec.
  - Do not add logging, print statements, or comments beyond docstrings.
  - Raise named exceptions for invalid inputs; do not return sentinel values.
  - All edge cases listed in the spec's counter-examples must be handled.
  
  Return only the function implementation, no surrounding code.
known_failures:
  - Agent may omit input validation if spec does not explicitly require it.
  - Agent may return None instead of raising on invalid input.
last_validated: 2026-04-10
validated_by: payments-team
```

Loading a spec and filling in a prompt template before sending to the API:

```python
import anthropic
import yaml
from pathlib import Path


def load_prompt_template(template_id: str) -> dict:
    library_path = Path("prompt_library") / f"{template_id}.yaml"
    with library_path.open() as f:
        return yaml.safe_load(f)


def build_prompt(template_id: str, spec_path: str, conventions_path: str) -> str:
    template = load_prompt_template(template_id)
    spec_content = Path(spec_path).read_text()
    conventions = Path(conventions_path).read_text()
    return template["template"].format(
        spec_content=spec_content,
        conventions_file=conventions,
    )


def generate_from_spec(spec_path: str) -> str:
    client = anthropic.Anthropic()
    prompt = build_prompt(
        template_id="implement_pure_function_v2",
        spec_path=spec_path,
        conventions_path="docs/coding_conventions.md",
    )
    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}],
    )
    return message.content[0].text
```

This pattern decouples the prompt structure (maintained by the team) from the task-specific content (the spec), so improving the prompt template improves all future generations that use it.

---

## 9.4 Refactoring with AI Agents

### The Prerequisite: Test Coverage

The non-negotiable condition for asking an agent to refactor code is adequate test coverage of the code being changed. Refactoring is, by definition, changing the structure of code without changing its behaviour. If you do not have tests that verify the behaviour, you have no way to know whether the agent changed the behaviour — and agents frequently do, because they optimise for code that looks clean rather than code that is behaviourally identical.

"Adequate" means coverage of the public interface of the module, including the main paths, all named edge cases from the spec, and any known failure modes. Eighty percent line coverage is a rough floor; what matters more is that every clause in the spec has a corresponding test. If coverage is below that threshold, write the missing tests before touching the implementation. This is not overhead — it is the work that makes refactoring safe.

### Safe Refactoring Patterns

**1. Extract function.** The agent reads a file, identifies repeated logic, and extracts it into a shared helper. The prompt for this is precise and directive:

```
Read the attached file `src/billing/invoice.py`.
Find all locations where overdue penalty calculation logic is repeated inline.
Extract this logic into a single function `_calculate_penalty(principal, days)` 
placed at the top of the module.
Replace each inline occurrence with a call to `_calculate_penalty`.
Do not change the logic, only the structure.
Return the complete modified file.
```

The key instruction is "do not change the logic, only the structure." Without it, agents will often optimise the extracted function, which introduces semantic changes that break tests.

**2. Rename across the codebase.** Renaming a function or variable consistently across a large codebase is tedious for humans and reliable for agents — but only if the scope is explicit:

```
The function `get_outstanding_amount` in `src/billing/` has been renamed to 
`get_unpaid_balance` to align with the domain model.
Update every call site in the `src/` directory to use the new name.
Do not change any call sites in `tests/` — those will be updated separately.
List every file you changed and how many occurrences were replaced.
```

The verification step: run `grep -r "get_outstanding_amount" src/` after the agent completes. Any result is a missed replacement that requires another pass.

**3. Modernise syntax.** Agents are effective at mechanical syntax upgrades that follow deterministic rules. For example, upgrading Python 3.8 type annotation patterns to 3.11 equivalents:

```
Update the type annotations in the attached file to use Python 3.11 syntax:
- Replace `Optional[X]` with `X | None`
- Replace `Union[X, Y]` with `X | Y`  
- Replace `List[X]` with `list[X]`
- Replace `Dict[K, V]` with `dict[K, V]`
- Replace `Tuple[X, Y]` with `tuple[X, Y]`
Do not change any other code. Return the complete modified file.
```

**4. Eliminate dead code.** Agents can identify and remove unused functions, but this requires caution: "unused" in a static sense may mean "called via reflection" or "part of a public API". The prompt must be explicit about what "unused" means in the codebase's context:

```
Identify all functions in the attached file that have no callers within the 
`src/` directory AND are not exported in `__all__`. 
List them with their line numbers and a one-sentence explanation of why each 
appears to be unused. Do not remove them — list them for human review.
```

Dead code removal should be proposed by the agent and approved by a human before execution, because the consequences of removing a function that is called at runtime via an undiscovered path are severe.

### Rollback Strategy

Always create a git branch before starting an agent-assisted refactoring session. Commit the working state before each individual refactoring step. If a step breaks tests, revert to the previous commit rather than attempting to diagnose the agent's changes inline — the revert is instant and the diagnosis can happen on a clean copy.

```bash
git checkout -b refactor/extract-penalty-calculation
git add -A && git commit -m "wip: pre-refactor checkpoint"
# Run the agent step
# Run tests
# If tests pass:
git add -A && git commit -m "refactor: extract _calculate_penalty helper"
# If tests fail:
git checkout -- .  # revert and diagnose
```

### When Not to Use Agents for Refactoring

Architectural refactoring — moving code between layers, changing the data model, splitting a monolith into services — should not be delegated to an agent without tight human control. These refactorings span too many files, involve too many implicit dependencies, and require architectural judgement that agents cannot reliably exercise. The symptom of an agent attempting architectural refactoring without sufficient constraint is a codebase that compiles and passes individual unit tests but fails integration tests because the agent invented a new interface that doesn't match the rest of the system.

The rule: if the refactoring touches more than one layer of the architecture, break it into single-layer steps and review each step before proceeding.

### Automated Extract Function Refactoring

```python
import anthropic
from pathlib import Path


def extract_repeated_pattern(
    filepath: str,
    pattern_description: str,
    new_function_name: str,
) -> str:
    """
    Ask the agent to extract a repeated pattern into a shared function.
    Returns the modified file content.
    """
    client = anthropic.Anthropic()
    source_code = Path(filepath).read_text()

    prompt = f"""You are refactoring a Python file.

Source file: {filepath}
```python
{source_code}
```

Task: Find all locations where {pattern_description} is implemented inline.
Extract this logic into a single function named `{new_function_name}`.
Place the new function before its first caller.
Replace each inline occurrence with a call to `{new_function_name}`.
Do not change any logic — only the structure.
Do not add, remove, or reorder any imports.

Return ONLY the complete modified Python file, with no explanation or markdown fencing."""

    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}],
    )
    return message.content[0].text


def safe_refactor(filepath: str, pattern_description: str, new_function_name: str):
    """
    Performs extract-function refactoring with a pre-check that tests exist.
    The caller is responsible for running tests after this function returns.
    """
    import subprocess

    # Verify tests pass before touching the file
    result = subprocess.run(["python", "-m", "pytest", "--tb=no", "-q"], capture_output=True)
    if result.returncode != 0:
        raise RuntimeError("Tests are failing before refactoring. Fix them first.")

    modified = extract_repeated_pattern(filepath, pattern_description, new_function_name)

    # Write to a temporary file for review before overwriting
    review_path = filepath + ".refactored"
    Path(review_path).write_text(modified)
    print(f"Refactored version written to {review_path}")
    print("Review the diff, then run tests, then replace the original if tests pass.")
```

---

## 9.5 Technical Debt Identification and Prioritisation

### Using AI to Audit Dead Code and Inconsistencies

Technical debt auditing is one of the highest-value, lowest-risk uses of AI agents in the Refine phase, because the agent is producing a report rather than modifying code. A wrong audit finding wastes a small amount of review time; a wrong refactoring can break a production system.

An audit prompt for a single file covers the four most common categories of accumulated debt:

```
Analyse the attached Python file for the following categories of technical debt:

1. Unused imports: imports that are never referenced in the file body.
2. Unreachable branches: code after an unconditional return, or conditions 
   that can never be true given the type constraints.
3. Functions with no callers: functions not referenced within this file 
   (note: they may be called from other files — flag, do not assert).
4. Deprecated patterns: use of patterns marked deprecated in Python 3.10+ 
   (e.g., collections.Callable, distutils, or typing.Optional).

For each finding, provide:
- Category (one of the four above)
- Line number(s)
- One-sentence description
- Estimated effort to fix: Low (< 5 min) / Medium (< 30 min) / High (> 30 min)
- Risk of fixing: Low (no callers affected) / Medium / High

Return your findings as a JSON array.
```

**Auditing inconsistencies between functions** is a subtler but equally valuable audit type. When two functions in the same module implement similar responsibilities, one often accumulates improvements that the other does not:

```
Compare the two functions below.
Function A: `calculate_early_payment_discount`
Function B: `calculate_overdue_penalty`

List every convention that Function A uses that Function B violates.
Examples of conventions to check: input validation style, error handling approach,
return type consistency, docstring format, use of named constants vs. magic numbers.
```

### Generating a Refactoring Roadmap

After auditing, the next step is prioritisation. An agent can produce a prioritised list of debt items with estimated effort and risk levels, which is more useful than a flat list sorted by line number:

```
Given the following debt items from the audit, produce a prioritised refactoring
roadmap. Sort by: (1) security/correctness issues first, (2) items that block
testing, (3) inconsistencies that cause confusion, (4) dead code and style issues.
For each item, include: priority rank, effort estimate, risk level, and one sentence
explaining the business reason for fixing it.
```

**Triage criteria** should be agreed at the team level and codified in the project's contributing guide:

- **Fix now**: security vulnerabilities, logic errors, anything that makes tests unreliable or impossible to write.
- **Fix soon**: inconsistencies that will confuse the next engineer (or agent) who touches the code; patterns that will propagate further if left in place.
- **Fix eventually**: dead code, style inconsistencies, deprecated patterns that still work.

### The Debt Register

A simple markdown file tracking known debt items provides a shared team view of what is owed, to whom, and at what priority. The debt register is generated by AI and maintained by the team:

```markdown
# Technical Debt Register — payments module
# Last generated: 2026-04-23 | Generated by: audit_file.py

## Status: Open

| ID | File | Category | Description | Priority | Effort | Owner | Status |
|----|------|----------|-------------|----------|--------|-------|--------|
| D-041 | billing/invoice.py:87 | Dead code | `format_legacy_invoice` has no callers | Low | Low | unassigned | Open |
| D-042 | billing/penalties.py:23 | Inconsistency | Uses magic number 0.0005; other modules use named constant | Medium | Low | unassigned | Open |
| D-043 | billing/penalties.py:61 | Missing validation | No check for negative principal | High | Low | unassigned | Open |
```

### Structured Debt Audit Function

```python
import anthropic
import json
from dataclasses import dataclass
from pathlib import Path


@dataclass
class DebtItem:
    category: str
    line_numbers: list[int]
    description: str
    effort: str   # "Low" | "Medium" | "High"
    risk: str     # "Low" | "Medium" | "High"


def audit_file(filepath: str) -> list[DebtItem]:
    """
    Audit a Python source file for technical debt using Claude.
    Returns a list of structured DebtItem instances.
    """
    client = anthropic.Anthropic()
    source_code = Path(filepath).read_text()

    prompt = f"""Analyse the following Python file for technical debt.

```python
{source_code}
```

Check for:
1. Unused imports
2. Unreachable branches
3. Functions with no callers in this file
4. Deprecated Python patterns (pre-3.10 typing, distutils, etc.)

Return a JSON array of objects. Each object must have these fields:
- "category": string, one of ["unused_import", "unreachable_branch", "no_callers", "deprecated_pattern"]
- "line_numbers": array of integers
- "description": string, one sentence
- "effort": string, one of ["Low", "Medium", "High"]
- "risk": string, one of ["Low", "Medium", "High"]

Return only the JSON array. No markdown, no explanation."""

    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}],
    )

    raw = message.content[0].text.strip()
    items = json.loads(raw)

    return [
        DebtItem(
            category=item["category"],
            line_numbers=item["line_numbers"],
            description=item["description"],
            effort=item["effort"],
            risk=item["risk"],
        )
        for item in items
    ]


def print_debt_report(filepath: str):
    items = audit_file(filepath)
    print(f"\nDebt audit: {filepath} — {len(items)} item(s) found\n")
    for i, item in enumerate(items, 1):
        lines = ", ".join(str(ln) for ln in item.line_numbers)
        print(f"  [{i}] {item.category.upper()} (line {lines})")
        print(f"      {item.description}")
        print(f"      Effort: {item.effort} | Risk: {item.risk}\n")
```

---

## 9.6 Feedback Loops: Learning from Evaluation Results

### The Feedback Loop Structure

The Refine phase only improves the process if findings are systematically fed back into the inputs of the next cycle. A failure that is fixed in isolation and not recorded in the spec or prompt library is a failure that will recur. The feedback loop has six steps:

1. **Test failure**: a test fails in CI, a human reviewer catches an error, or an audit surfaces a debt item.
2. **Diagnose failure category**: apply the taxonomy from section 9.2 to identify the root cause — spec ambiguity, missing constraint, context gap, model limitation, or hallucinated API.
3. **Update the spec**: add the missing constraint or counter-example. Version the spec.
4. **Update the prompt library**: if the failure reveals a systematic weakness in a prompt template — agents consistently omit validation, or consistently choose the wrong exception type — update the template and increment its version number.
5. **Verify the fix**: regenerate and run the full test suite. Do not close the loop until the regenerated code passes.
6. **Commit both artefacts**: the updated spec and the updated implementation are committed together, with a message that references the failure that drove the change.

This loop is cheap when it runs correctly. The diagnosis takes five minutes; the spec update takes five minutes; the regeneration and verification take a few minutes more. The total cost of a systematic refinement cycle is lower than the cost of the same failure recurring three times.

### Building a Prompt Library

The prompt library is effective only if it is maintained as a shared, reviewed resource — not a personal folder of working prompts. The entry format shown in section 9.3 captures three things that are essential for the library to be useful: the template itself, a worked example showing the output the template produces, and the known failure modes — the things that go wrong when this template is used incorrectly or with insufficient context.

Organise the library by task type, not by project or module. A template for "implement a pure validation function" is reusable across all projects; a template for "implement the invoice discount function" is not. The test of whether a template belongs in the shared library is whether a team member working on a different module could use it unchanged.

### Team Conventions as AI Context

The most underused input to AI generation is a team's implicit coding conventions. Every team has rules that are not in the style guide: "we use repository pattern, not active record"; "we raise `ValueError` for business logic errors and `TypeError` for type errors"; "we always return `list`, never `tuple`, from query functions." These rules are invisible to the agent unless they are written down and included in the generation prompt.

The practical step is to create a `docs/coding_conventions.md` file that documents these implicit rules explicitly. This file is included in every generation prompt via the prompt template. Each time an agent produces code that violates an implicit convention, the convention is added to this file. Over time, the file becomes a precise specification of how the team writes code, and generation quality improves measurably.

### Measuring Improvement

Refinement should be measured, not assumed. The metrics that matter are:

- **First-generation pass rate**: of the functions generated this week, what percentage passed all tests on the first attempt without any manual changes?
- **Average refinement iterations**: how many generate-verify cycles did it take, on average, to reach a passing implementation?
- **Time-to-verified**: from "prompt submitted" to "all tests green", how long did the cycle take, on average?

Track these per task type. If "implement database query functions" has a low first-generation pass rate, that task type needs a better prompt template or a more precise spec convention. If "implement pure validation functions" is consistently high, the prompt template for that type is mature and can be used as a reference for improving others.

### The Weekly Spec Review

A fifteen-minute weekly ritual: the team reviews all spec failures from the previous week. For each failure, the engineer who diagnosed it presents: the failure, the failure category, the spec or prompt change made. The group decides whether the fix is sufficient or whether a broader change is needed. New constraint patterns are added to the shared conventions document. Prompt templates that were updated are briefly reviewed.

This ritual keeps the feedback loop connected to the team as a whole, not just to the engineer who happened to encounter the failure. It is one of the most cost-effective quality practices in the Agentic SDLC.

---

## 9.7 Continuous Refinement in CI/CD

### Nightly AI-Driven Code Quality Checks

A nightly CI job that runs the agent over the codebase serves a different purpose than the human-visible test suite. Tests verify functional correctness. The nightly AI check looks for structural drift: has newly generated code introduced patterns that violate the project's conventions? Are there new unused imports? Has the spec-implementation gap grown? These signals are not caught by tests because they are not functional failures — they are quality degradations that accumulate quietly until they become functional failures.

The nightly check runs the debt audit function from section 9.5 over every modified file and compares the result against the debt register. New debt items are added to the register automatically and flagged for triage. Items that have been open longer than thirty days trigger a reminder in the team's notification channel.

### Automated PR Review Agents

An AI PR reviewer runs on every pull request as a CI check. It reads the diff and posts a structured review comment that covers correctness, convention adherence, and spec alignment. Unlike a human reviewer, it is consistent: it applies the same checks to every PR, every time, without fatigue. Unlike a linter, it understands intent: it can see that a function is missing error handling for a specific case because it has read the spec for that function.

The GitHub Actions workflow for the automated PR reviewer:

```yaml
name: AI PR Review
on: [pull_request]
jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install dependencies
        run: pip install anthropic requests
      - name: Run AI PR review
        run: python scripts/ai_pr_review.py ${{ github.event.pull_request.number }}
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITHUB_REPOSITORY: ${{ github.repository }}
```

The reviewer script:

```python
#!/usr/bin/env python3
"""
scripts/ai_pr_review.py
Fetches the PR diff, sends it to Claude for review, and posts the result
as a PR comment via the GitHub API.
"""
import os
import sys
import anthropic
import requests


REVIEW_PROMPT = """You are an automated code reviewer for a Python codebase.
Your job is to review the following git diff and produce a structured review.

Project conventions:
- All functions must validate inputs and raise named exceptions (not return None/sentinel)
- Use `X | None` not `Optional[X]` for type annotations
- Never use bare `except:` — always catch a specific exception type
- All public functions must have docstrings

Diff to review:
```diff
{diff}
```

Produce your review in this exact format:

## AI Code Review

**Functions added or modified:** <count>

For each function, write one paragraph:
- Function name and what it does
- Whether it correctly handles error cases and edge inputs
- Any convention violations from the list above
- Any spec alignment issues if a spec file exists in `specs/`

**Summary verdict:** APPROVE | REQUEST_CHANGES | COMMENT
**One-line reason for verdict:**
"""


def get_pr_diff(pr_number: int) -> str:
    repo = os.environ["GITHUB_REPOSITORY"]
    token = os.environ["GITHUB_TOKEN"]
    url = f"https://api.github.com/repos/{repo}/pulls/{pr_number}"
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github.v3.diff",
    }
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.text


def post_pr_comment(pr_number: int, body: str):
    repo = os.environ["GITHUB_REPOSITORY"]
    token = os.environ["GITHUB_TOKEN"]
    url = f"https://api.github.com/repos/{repo}/issues/{pr_number}/comments"
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github.v3+json",
    }
    response = requests.post(url, headers=headers, json={"body": body})
    response.raise_for_status()


def review_pr(pr_number: int):
    diff = get_pr_diff(pr_number)

    if not diff.strip():
        print("Empty diff — skipping review.")
        return

    # Truncate very large diffs to avoid exceeding context limits
    max_diff_chars = 30_000
    if len(diff) > max_diff_chars:
        diff = diff[:max_diff_chars] + "\n\n[diff truncated — review the full diff manually]"

    client = anthropic.Anthropic()
    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        messages=[
            {
                "role": "user",
                "content": REVIEW_PROMPT.format(diff=diff),
            }
        ],
    )

    review_body = message.content[0].text
    post_pr_comment(pr_number, review_body)
    print(f"Review posted to PR #{pr_number}.")


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: ai_pr_review.py <pr_number>")
        sys.exit(1)
    review_pr(int(sys.argv[1]))
```

### The Refinement Signal from the PR Reviewer

The PR reviewer is not just a quality gate — it is a measurement instrument. When the reviewer flags the same pattern in multiple consecutive PRs, that pattern is a signal that the team's generation process has a systematic gap. The correct response is not to keep fixing the pattern manually; it is to add the pattern to the project's `CLAUDE.md` file (or equivalent conventions document) and to the relevant prompt template, so that the next generation does not produce it.

A practical threshold: if the AI reviewer flags the same category of issue in three or more PRs in a single sprint, that category is promoted to a spec convention or prompt constraint in the next weekly spec review. The CI reviewer thus drives the refinement of the generation inputs, closing the feedback loop at the team level.

---

## 9.8 When to Stop Refining: Shipping vs. Perfecting

The Refine loop is productive until it is not. Each refinement iteration has a real cost: engineering time, API tokens, context switching, and the opportunity cost of not working on the next feature. At some point — and the point arrives sooner than most engineers expect — the marginal improvement produced by another refinement iteration is not worth its cost.

**Defining done in the spec.** The exit condition for the Refine loop should be written in the spec before the first line of code is generated: "This function is complete when it passes all tests in `tests/billing/test_penalties.py` and the debt audit returns zero High-priority items." Exit conditions that are vague — "when the code looks good" or "when we're satisfied" — produce refinement loops that never end, because satisfaction is a moving target.

**The 80/20 rule for AI refinement.** In practice, the first generation of a well-specified function gets you approximately 70% of the way to a correct implementation. The first refinement loop — diagnose, update spec, regenerate — typically gets you to 90%. Each subsequent loop has sharply diminishing returns, because the remaining failures are at the edges of the spec's precision, and making the spec precise enough to drive perfect generation often costs more than fixing the edge case manually. The practical implication is that most functions should require at most two refinement iterations; if you are on your fourth, you are either working on an extremely complex function or you are over-refining.

**Shipping criteria.** The gates for closing the Refine loop and moving to the next phase should be objective and team-agreed:

- All tests pass, including integration tests and any spec-derived property tests.
- All static checks pass (type checker, linter, security scanner).
- Human review is complete — at least one engineer who did not write the spec has read the implementation.
- The security checklist has been checked for any function that handles user input, authentication, or data persistence.

These are the gates. They are not negotiable based on how the code "feels."

**The perfectionism trap.** Engineers who have written code by hand for years have an intuition for what "good" code looks like, and AI-generated code frequently violates that intuition — not because it is wrong, but because it is generated differently from how a human would write it. The impulse to refine AI code toward the style a human would have chosen is the perfectionism trap: it is invisible work that has no effect on correctness, and it consumes the time savings that AI generation was supposed to provide. The discipline is to distinguish style discomfort from functional deficiency. If the code passes all the specified criteria, it is done.

---

## 9.9 Tutorial: A Complete Refinement Cycle

This tutorial walks through a single end-to-end refinement cycle: from a generated function with a bug, through diagnosis and spec update, to a verified regenerated implementation.

### Starting Point

The initial spec for `get_overdue_tasks` was:

```markdown
# Spec: get_overdue_tasks

Return all tasks that are past their due date.

Parameters:
- tasks (list[Task]): all tasks in the system
- reference_date (date): the date to compare against

Returns:
- list[Task]: tasks where task.due_date < reference_date
```

The agent generated this implementation:

```python
from datetime import date

def get_overdue_tasks(tasks: list[Task], reference_date: date) -> list[Task]:
    """Return all tasks that are past their due date."""
    return [task for task in tasks if task.due_date < reference_date]
```

### Step 1: Reproduce the Bug with a Failing Test

During integration testing, completed tasks appear in the overdue list. The failing test:

```python
from datetime import date
from models import Task

def test_completed_tasks_not_in_overdue():
    past_date = date(2026, 1, 1)
    reference = date(2026, 4, 23)

    completed_overdue = Task(due_date=past_date, status="completed")
    open_overdue = Task(due_date=past_date, status="open")
    open_current = Task(due_date=reference, status="open")

    result = get_overdue_tasks(
        [completed_overdue, open_overdue, open_current],
        reference_date=reference,
    )

    # Completed tasks must not appear, even if their due date has passed
    assert completed_overdue not in result
    assert open_overdue in result
    assert open_current not in result
```

This test fails. `completed_overdue` appears in the result.

### Step 2: Diagnose the Failure Category

Map the code to the spec clause by clause:

- "Return all tasks that are past their due date" — implemented as `task.due_date < reference_date`. Correct.
- No spec clause mentions task status.

The failure category is **missing constraint**. The spec did not say anything about task status. The agent made the reasonable default choice of filtering only on date, which is exactly what the spec asked for.

### Step 3: Update the Spec

```markdown
# Spec: get_overdue_tasks
# Version: 1.1 | Last updated: 2026-04-23

Return all tasks that are past their due date AND are not yet completed.

Parameters:
- tasks (list[Task]): all tasks in the system
- reference_date (date): the date to compare against

Returns:
- list[Task]: tasks where task.due_date < reference_date AND task.status != "completed"

Constraints [added in v1.1]:
- Completed tasks (task.status == "completed") must NOT appear in the result,
  even if their due date is in the past.

Counter-examples:
- A task with due_date=2026-01-01, status="completed", reference_date=2026-04-23
  → must NOT be in the result.
- A task with due_date=2026-01-01, status="open", reference_date=2026-04-23
  → MUST be in the result.
```

### Step 4: Regenerate Using the Anthropic API

```python
import anthropic
from pathlib import Path


def regenerate_function(spec_path: str) -> str:
    client = anthropic.Anthropic()
    spec_content = Path(spec_path).read_text()
    conventions = Path("docs/coding_conventions.md").read_text()

    prompt = f"""You are implementing a Python function for a production codebase.

Coding conventions:
{conventions}

Specification:
{spec_content}

Requirements:
- Implement only the function described in the spec.
- Handle all constraints listed in the spec.
- Return only the function implementation with its imports. No surrounding code.
"""

    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}],
    )
    return message.content[0].text


if __name__ == "__main__":
    result = regenerate_function("specs/tasks/get_overdue_tasks.md")
    print(result)
```

The regenerated implementation:

```python
from datetime import date

def get_overdue_tasks(tasks: list[Task], reference_date: date) -> list[Task]:
    """Return all tasks that are past their due date and not yet completed."""
    return [
        task for task in tasks
        if task.due_date < reference_date and task.status != "completed"
    ]
```

### Step 5: Run Verification

```
$ python -m pytest tests/tasks/test_get_overdue_tasks.py -v
PASSED tests/tasks/test_get_overdue_tasks.py::test_completed_tasks_not_in_overdue
PASSED tests/tasks/test_get_overdue_tasks.py::test_open_overdue_tasks_returned
PASSED tests/tasks/test_get_overdue_tasks.py::test_empty_task_list

3 passed in 0.12s
```

All tests pass.

### Step 6: Commit the Updated Spec Alongside the Fixed Implementation

```bash
git add specs/tasks/get_overdue_tasks.md
git add src/tasks/queries.py
git commit -m "fix(tasks): exclude completed tasks from overdue list

Spec v1.1 adds explicit constraint: task.status != 'completed'.
Root cause was missing constraint (failure category 2).
Regenerated from updated spec; all tests pass."
```

The commit message records the failure category, the spec version, and the verification outcome. Future engineers — and future agents — can read the git history to understand why the constraint exists.

---

## Chapter Summary

- **Refine is a discipline, not an activity**: the difference between reactive patching and systematic refinement is whether the fix prevents the same failure from recurring in the next generation cycle. Systematic refinement targets the specification, the generation process, and the codebase — in that order.

- **Failure diagnosis precedes failure fixing**: every failure belongs to one of five categories (spec ambiguity, missing constraint, context gap, model limitation, hallucinated API), and the correct fix depends entirely on the category. Applying the wrong fix wastes time and leaves the root cause intact.

- **Specs are living, versioned artefacts**: the constraint accumulation pattern — start minimal, generate, find the failure, add the constraint, repeat — produces specs that are empirically grounded in actual failures. Version-controlled specs committed alongside implementation changes keep the two in sync.

- **Refactoring with agents requires tests first**: without test coverage, agent-assisted refactoring is indistinguishable from uncontrolled code change. The safe pattern is: verify tests pass, create a git branch, refactor one step at a time, verify tests pass after each step.

- **CI/CD closes the refinement loop at scale**: automated PR reviewers and nightly quality audits provide continuous measurement of the gap between the codebase and its specifications. When the reviewer flags the same pattern repeatedly, that pattern is promoted to a spec convention or prompt constraint — the feedback loop drives systematic improvement without manual intervention.

---

## Review Questions

**1. (Conceptual)** What is the difference between reactive patching and systematic refinement? Give an example of each.

Reactive patching addresses a single observed failure by changing the implementation: a function returns `None` when it should return `[]`, you change the return statement, and you close the issue. Systematic refinement addresses the process that produced the failure: you identify that the spec did not specify the return type for empty results, add "return `[]`, never `None`" as an explicit constraint, update the prompt template to include this constraint for all list-returning functions, and verify that the regenerated implementation is correct. Reactive patching prevents this particular bug in this particular function; systematic refinement prevents the same class of bug in every function generated from this point forward.

**2. (Applied)** An AI-generated function `send_notification(user_id, message)` consistently fails to validate that `message` is non-empty. Classify the failure category and write the spec clause that would prevent it.

The failure category is **missing constraint**: the spec presumably said "send a notification to the user with the given message" without specifying what to do if `message` is empty. The agent made the reasonable default choice of not validating the input. The spec clause to add:

```markdown
## Input Constraints

- `message` must be a non-empty string. If `message` is an empty string or
  contains only whitespace, raise `ValueError("message must not be empty")`.
- Do NOT send a notification if the input validation fails.

Counter-example:
- send_notification(user_id=42, message="") → raises ValueError("message must not be empty")
- send_notification(user_id=42, message="  ") → raises ValueError("message must not be empty")
```

**3. (Applied)** You want to refactor a module that has 60% test coverage. Describe the steps you would take before, during, and after the refactoring to minimise the risk of introducing regressions.

**Before:** Identify which lines and branches are not covered by the existing 60%. For any public function that the refactoring will touch, write tests that cover the uncovered paths — particularly edge cases and error paths. Do not proceed until coverage of the functions being refactored is at or above 85%, with all spec counter-examples covered. Create a git branch: `git checkout -b refactor/<description>`. Commit the current state.

**During:** Refactor one logical unit at a time. After each unit, run the full test suite. If any test fails, revert to the previous commit (`git checkout -- .`) and diagnose the failure before continuing. Do not accumulate multiple refactoring steps without a passing test run between them.

**After:** Run the full test suite. Run the static type checker and linter. Run the debt auditor and verify that the refactoring did not introduce new debt items. Open a PR and allow the automated PR reviewer to run. Review the diff yourself before merging. Merge only when all checks pass and at least one human reviewer has approved.

**4. (Conceptual)** What should a team's prompt library contain? Describe three entries you would include for a typical web API project.

A prompt library entry contains: a task type identifier, a prompt template with named placeholders, a worked example showing the input and expected output, and a list of known failure modes — patterns the agent gets wrong with this template if the context is insufficient.

Three entries for a web API project:

- **`implement_api_endpoint`**: template for generating a FastAPI or Flask route handler from an OpenAPI spec fragment. Known failure modes: agent omits authentication middleware, agent uses 200 for created resources instead of 201, agent returns raw exceptions instead of structured error responses.

- **`implement_repository_method`**: template for generating a database query method given a data model and a query description. Known failure modes: agent uses ORM patterns inconsistent with the project (e.g., active record vs. repository pattern), agent omits pagination for list queries, agent does not handle the case where the record is not found.

- **`write_unit_tests_for_function`**: template for generating a test suite given a function implementation and its spec. Known failure modes: agent tests implementation details rather than behaviour, agent omits edge cases listed in the spec's counter-examples, agent uses `assert` without messages, making failures hard to diagnose.

**5. (Applied)** Your CI pipeline runs an AI PR reviewer. It flags the same "missing None check" pattern in 8 consecutive PRs. What systemic action should you take, and how would you implement it?

Eight consecutive flags of the same pattern is a clear signal that the generation process has a systematic gap — engineers are consistently generating functions that do not guard against `None` inputs, because nothing in the generation inputs requires them to. The systemic action is to close the feedback loop at the prompt library level, not at the individual PR level.

**Implementation:** First, add an explicit constraint to `docs/coding_conventions.md`: "Every function that accepts a parameter that could be `None` must check for `None` at the entry point and either raise `TypeError` or handle the `None` case explicitly. Do not propagate `None` silently into the function body." Second, update the relevant prompt templates (`implement_api_endpoint`, `implement_repository_method`, etc.) to include this convention in the prompt. Third, update the `CLAUDE.md` file at the repository root — if your team uses Claude Code or similar tools interactively, this file is included in every session, and the constraint will be applied in interactive generation as well. Finally, at the next weekly spec review, verify that the new PRs generated after the change no longer trigger the flag. If they do, the constraint is not specific enough and needs a counter-example added to the conventions document.
