# Chapter 7: Evaluation-Driven Development

> *"Testing shows the presence of bugs, not their absence — but not testing shows the absence of rigour."*

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Articulate why AI-generated code must be treated as a hypothesis rather than a solution.
2. Design evaluation criteria before generating code.
3. Apply multiple evaluation strategies: automated tests, static analysis, LLM-as-judge, and human review.
4. Build a basic evaluation harness for AI-generated code.
5. Measure and interpret hallucination in code generation outputs.
6. Apply evaluation-driven development as an end-to-end workflow.

---

## 7.1 The Hypothesis Framing

In scientific research, a hypothesis is a proposed explanation that must be tested against evidence before it is accepted. In AI-native engineering, a similar discipline applies to AI-generated code: it is a *hypothesis about what the correct implementation looks like*, not a verified solution.

This framing is not semantic — it has practical consequences:

- A hypothesis requires testing before it is trusted
- The tests must be designed before (or independently of) generating the hypothesis
- A hypothesis that passes all tests is *supported*, not *proven*
- A hypothesis that fails any test must be investigated, not blindly regenerated

The failure mode of treating AI-generated code as a solution is well-documented. GitHub's own research found that roughly 40% of Copilot-generated code suggestions contained bugs when evaluated against the intent of the surrounding code ([Pearce et al., 2021](https://arxiv.org/abs/2108.09293)). A more targeted study of security-relevant code found that AI assistants generated insecure code in 25–40% of cases ([Perry et al., 2022](https://arxiv.org/abs/2211.03622)).

These numbers are not arguments against using AI — they are arguments for evaluation.

---

## 7.2 Evaluation-Driven Development (EDD)

Evaluation-Driven Development (EDD) is a workflow that makes evaluation the primary activity, with generation as a means to an end.

The EDD workflow:

```
1. Define what correct looks like (evaluation criteria)
   ↓
2. Write evaluation artefacts (tests, specifications, rubrics)
   ↓
3. Generate candidate implementation
   ↓
4. Evaluate candidate against criteria
   ↓
5. If evaluation passes → accept
   If evaluation fails → diagnose, refine specification, repeat from 3
```

The critical principle: **evaluation criteria must be defined before generation begins**. If you define your criteria after seeing the generated output, you are at risk of unconsciously adjusting them to accept whatever was produced.

This is analogous to test-driven development (TDD) in traditional engineering ([Beck, 2002](https://www.oreilly.com/library/view/test-driven-development/0321146530/)): write the test first, then write the code that makes it pass. EDD extends this to AI-native workflows where the "code" is generated rather than hand-written.

---

## 7.3 Evaluation Strategies

Evaluation is not a single activity — it is a portfolio of complementary techniques, each catching different classes of defects.

### 7.3.1 Automated Tests (Functional Evaluation)

Automated tests are the first line of evaluation for AI-generated code. If you followed the specification template from Chapter 6, you have already defined the expected input-output pairs — these become your test cases directly.

```python
# Evaluation via test suite
import pytest
from decimal import Decimal
from datetime import date
from src.task_service import calculate_overdue_penalty


class TestCalculateOverduePenalty:
    """Evaluation suite derived directly from the specification examples."""

    def test_completed_on_time_returns_zero(self) -> None:
        result = calculate_overdue_penalty(
            due_date=date(2024, 1, 5),
            completion_date=date(2024, 1, 5),
            daily_rate=Decimal("10"),
        )
        assert result == Decimal("0.00")

    def test_completed_before_due_returns_zero(self) -> None:
        result = calculate_overdue_penalty(
            due_date=date(2024, 1, 5),
            completion_date=date(2024, 1, 3),
            daily_rate=Decimal("10"),
        )
        assert result == Decimal("0.00")

    def test_completed_one_day_late(self) -> None:
        result = calculate_overdue_penalty(
            due_date=date(2024, 1, 1),
            completion_date=date(2024, 1, 2),
            daily_rate=Decimal("10"),
        )
        assert result == Decimal("10.00")

    def test_completed_four_days_late(self) -> None:
        result = calculate_overdue_penalty(
            due_date=date(2024, 1, 1),
            completion_date=date(2024, 1, 5),
            daily_rate=Decimal("10"),
        )
        assert result == Decimal("40.00")

    def test_uses_today_when_no_completion_date(self) -> None:
        result = calculate_overdue_penalty(
            due_date=date(2024, 1, 1),
            completion_date=None,
            daily_rate=Decimal("10"),
            today=date(2024, 1, 3),
        )
        assert result == Decimal("20.00")

    def test_raises_for_negative_daily_rate(self) -> None:
        with pytest.raises(ValueError):
            calculate_overdue_penalty(
                due_date=date(2024, 1, 1),
                completion_date=None,
                daily_rate=Decimal("-1"),
            )

    def test_result_uses_decimal_not_float(self) -> None:
        result = calculate_overdue_penalty(
            due_date=date(2024, 1, 1),
            completion_date=date(2024, 1, 5),
            daily_rate=Decimal("10"),
        )
        assert isinstance(result, Decimal), "Result must be Decimal, not float"
```

**Write tests before generating.** If you write your tests after seeing the generated code, you risk writing tests that are shaped by the implementation rather than the specification.

### 7.3.2 Static Analysis (Structural Evaluation)

Static analysis evaluates the generated code's structure without executing it. It catches a different class of defect from functional tests.

```bash
# Run the full static analysis suite on generated code
ruff check src/generated_function.py      # Style and common errors
mypy src/generated_function.py --strict   # Type correctness
bandit src/generated_function.py          # Security vulnerabilities
```

Static analysis is particularly important for AI-generated code because:
- AI models can generate code with correct runtime behaviour but incorrect type annotations
- AI models occasionally generate insecure patterns (SQL concatenation, shell injection, hardcoded credentials)
- AI models sometimes generate dead code or unreachable branches

### 7.3.3 LLM-as-Judge (Semantic Evaluation)

LLM-as-judge uses a second language model to evaluate the output of the first. It is useful for capturing semantic properties that are difficult to express as automated tests: correctness of approach, adherence to conventions, readability, and potential edge cases the tester missed.

```python
import anthropic

client = anthropic.Anthropic()


def llm_evaluate_code(specification: str, generated_code: str) -> dict[str, str]:
    """
    Use an LLM to evaluate generated code against its specification.
    Returns a dict with 'verdict', 'issues', and 'suggestions'.
    """
    response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=1024,
        system="You are a senior software engineer conducting a code review. "
               "Be specific and critical. Do not give generic praise.",
        messages=[
            {
                "role": "user",
                "content": f"""Review the following generated code against its specification.

SPECIFICATION:
{specification}

GENERATED CODE:
{generated_code}

Evaluate:
1. Does the code correctly implement all specified behaviour?
2. Does it handle all specified error cases?
3. Does it satisfy all specified constraints?
4. Are there any edge cases in the specification that the code mishandles?
5. Are there any security issues?

Format your response as:
VERDICT: [PASS / FAIL / NEEDS_REVIEW]
ISSUES: [Specific issues found, or "None"]
SUGGESTIONS: [Specific improvements, or "None"]""",
            }
        ],
    )

    text = response.content[0].text
    result: dict[str, str] = {}
    for line in text.strip().split("\n"):
        for key in ("VERDICT", "ISSUES", "SUGGESTIONS"):
            if line.startswith(f"{key}:"):
                result[key.lower()] = line[len(key) + 1 :].strip()
    return result
```

**Important caveats for LLM-as-judge:**

- A second LLM can miss the same systematic errors as the first
- LLM judges tend to be lenient — they favour approving plausible-looking code
- LLM evaluation is not a substitute for automated tests; it is a complement
- Use the same model family with a different configuration (e.g., system prompt with stricter review instructions) for best results

Research on LLM-as-judge for code ([Zheng et al., 2023](https://arxiv.org/abs/2306.05685)) confirms both its utility and its tendency toward false positives — it is best used as a screening step before human review, not as a final verdict.

### 7.3.4 Human Review (Expert Evaluation)

For any code that will reach production, human code review remains essential. The chapter on code review in Chapter 4 applies fully to AI-generated code — in fact, AI-generated code warrants *more* scrutiny, not less, because:

- It may look superficially correct while containing subtle logic errors
- It may have been generated from a training distribution that does not match your domain
- It may follow common patterns that are correct in general but wrong for your specific context

When reviewing AI-generated code, pay particular attention to:
- Boundary conditions (what happens at the edges of input ranges?)
- Error handling (are all error cases handled, or just the ones in the specification?)
- Security (does the code handle untrusted input safely?)
- Resource management (are files closed? Are database connections released?)

---

## 7.4 Measuring Hallucination in Code

In the context of code generation, *hallucination* refers to the model generating plausible-looking code that is factually incorrect: calling non-existent functions, using incorrect API signatures, or inventing business rules that were not specified.

### 7.4.1 Types of Hallucination

| Type | Example | Detection |
|---|---|---|
| **API hallucination** | Calls `tasks.filter_by_status()` which doesn't exist | Import error at runtime; static analysis |
| **Logic hallucination** | Invents a penalty calculation formula not in the spec | Failing test cases |
| **Constraint violation** | Uses string concatenation instead of parameterised queries | Static analysis; security review |
| **Type hallucination** | Returns `None` instead of raising `ValueError` | Type checker; test assertions |
| **Factual hallucination** | Assumes Python 3.9 syntax in a Python 3.11 codebase | Runtime error; version check |

### 7.4.2 Measuring Hallucination Rate

For evaluating AI tools in your specific codebase, you can measure hallucination rate systematically:

```python
from dataclasses import dataclass, field


@dataclass
class EvaluationResult:
    specification: str
    generated_code: str
    tests_passed: int
    tests_total: int
    static_analysis_issues: list[str] = field(default_factory=list)
    hallucinations: list[str] = field(default_factory=list)

    @property
    def pass_rate(self) -> float:
        if self.tests_total == 0:
            return 0.0
        return self.tests_passed / self.tests_total

    @property
    def hallucination_count(self) -> int:
        return len(self.hallucinations)


def evaluate_generation(
    specification: str, generated_code: str, test_results: dict
) -> EvaluationResult:
    """Combine test results and static analysis into a single evaluation record."""
    return EvaluationResult(
        specification=specification,
        generated_code=generated_code,
        tests_passed=test_results["passed"],
        tests_total=test_results["total"],
        static_analysis_issues=test_results.get("static_issues", []),
        hallucinations=test_results.get("hallucinations", []),
    )
```

Over multiple evaluations, tracking these metrics tells you:
- Which types of functions AI generates reliably vs. unreliably
- Whether a particular model or prompt pattern reduces hallucination
- Where in your codebase AI assistance is highest-value vs. highest-risk

---

## 7.5 Building an Evaluation Harness

An *evaluation harness* is a system that automates the full evaluation pipeline: specification in, verdict out.

```python
# eval_harness.py
import subprocess
import tempfile
import os
from dataclasses import dataclass
import anthropic

client = anthropic.Anthropic()


@dataclass
class HarnessResult:
    specification: str
    generated_code: str
    syntax_valid: bool
    type_check_passed: bool
    tests_passed: bool
    llm_verdict: str
    llm_issues: str


def generate_code(specification: str) -> str:
    """Generate code from a specification."""
    response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=2048,
        system="You are a senior Python engineer. Output ONLY the function implementation, "
               "no explanation, no markdown fences.",
        messages=[{"role": "user", "content": specification}],
    )
    return response.content[0].text.strip()


def check_syntax(code: str) -> bool:
    """Check that the generated code is syntactically valid Python."""
    try:
        compile(code, "<generated>", "exec")
        return True
    except SyntaxError:
        return False


def run_type_check(code: str) -> bool:
    """Run mypy on the generated code."""
    with tempfile.NamedTemporaryFile(
        mode="w", suffix=".py", delete=False
    ) as tmp:
        tmp.write(code)
        tmp_path = tmp.name

    try:
        result = subprocess.run(
            ["mypy", tmp_path, "--strict", "--no-error-summary"],
            capture_output=True,
            text=True,
        )
        return result.returncode == 0
    finally:
        os.unlink(tmp_path)


def run_tests(code: str, test_code: str) -> bool:
    """Run a test suite against the generated code."""
    with tempfile.TemporaryDirectory() as tmp_dir:
        # Write generated code
        impl_path = os.path.join(tmp_dir, "impl.py")
        with open(impl_path, "w") as f:
            f.write(code)

        # Write tests (importing from impl)
        test_path = os.path.join(tmp_dir, "test_impl.py")
        with open(test_path, "w") as f:
            f.write(f"from impl import *\n\n{test_code}")

        result = subprocess.run(
            ["pytest", test_path, "-q"],
            capture_output=True,
            text=True,
            cwd=tmp_dir,
        )
        return result.returncode == 0


def run_evaluation(
    specification: str, test_code: str
) -> HarnessResult:
    """Run the full evaluation pipeline."""
    print("Generating code...")
    generated = generate_code(specification)

    print("Checking syntax...")
    syntax_ok = check_syntax(generated)

    print("Running type check...")
    type_ok = run_type_check(generated) if syntax_ok else False

    print("Running tests...")
    tests_ok = run_tests(generated, test_code) if syntax_ok else False

    print("Running LLM evaluation...")
    llm_result = llm_evaluate_code(specification, generated)

    return HarnessResult(
        specification=specification,
        generated_code=generated,
        syntax_valid=syntax_ok,
        type_check_passed=type_ok,
        tests_passed=tests_ok,
        llm_verdict=llm_result.get("verdict", "UNKNOWN"),
        llm_issues=llm_result.get("issues", ""),
    )


def llm_evaluate_code(specification: str, code: str) -> dict[str, str]:
    response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=512,
        messages=[
            {
                "role": "user",
                "content": f"Review this code against the specification. "
                           f"Be critical.\n\nSPEC:\n{specification}\n\nCODE:\n{code}\n\n"
                           f"Format: VERDICT: [PASS/FAIL]\nISSUES: [list or None]",
            }
        ],
    )
    text = response.content[0].text
    result: dict[str, str] = {}
    for line in text.strip().split("\n"):
        for key in ("VERDICT", "ISSUES"):
            if line.startswith(f"{key}:"):
                result[key.lower()] = line[len(key) + 1 :].strip()
    return result
```

---

## 7.6 Tutorial: Evaluation Harness for the Course Project

### Setup

```bash
pip install anthropic pytest mypy ruff
```

### Running an Evaluation

```python
# example_evaluation.py
from eval_harness import run_evaluation

specification = """
Implement filter_tasks(tasks, *, status=None, priority=None, assignee=None) -> list[Task]
[... full specification from Chapter 6 ...]
"""

test_code = """
from datetime import date
from uuid import uuid4
from decimal import Decimal
import pytest

def make_task(status="open", priority=2, assignee=None):
    return Task(id=uuid4(), title="T", priority=priority,
                status=status, assignee=assignee)

def test_no_filter_returns_all():
    tasks = [make_task(), make_task(status="completed")]
    assert len(filter_tasks(tasks)) == 2

def test_filter_by_status():
    tasks = [make_task(status="open"), make_task(status="completed")]
    result = filter_tasks(tasks, status="open")
    assert len(result) == 1 and result[0].status == "open"

def test_filter_by_priority():
    tasks = [make_task(priority=1), make_task(priority=3)]
    result = filter_tasks(tasks, priority=1)
    assert len(result) == 1 and result[0].priority == 1

def test_invalid_priority_raises():
    with pytest.raises(ValueError):
        filter_tasks([], priority=0)

def test_not_a_list_raises():
    with pytest.raises(TypeError):
        filter_tasks("not a list")
"""

result = run_evaluation(specification, test_code)
print(f"Syntax valid:     {result.syntax_valid}")
print(f"Type check:       {result.type_check_passed}")
print(f"Tests passed:     {result.tests_passed}")
print(f"LLM verdict:      {result.llm_verdict}")
print(f"LLM issues:       {result.llm_issues}")
```

---

## 7.7 Debugging AI-Generated Code

Evaluation tells you *whether* the generated code is wrong. Debugging tells you *why* and *how to fix it*. Debugging code you did not write — and may not have fully read — requires a different approach from debugging code you authored.

### 7.7.1 The Core Challenge

When you write a function yourself, you carry a mental model of its design. When a bug surfaces, you can often reason directly: "I remember that edge case — I handled it in the wrong branch." With AI-generated code, you start without that model. You must *reconstruct* the logic from the code before you can identify where it diverges from the specification.

This is not a weakness of AI-generated code specifically — it is the same challenge faced when debugging any unfamiliar code. The difference is frequency: in an AI-native workflow, "unfamiliar code" describes a much larger fraction of the codebase.

### 7.7.2 Debugging Workflow

**Step 1: Reproduce the failure with a minimal test**

Before reading the code at all, write a test that reproduces the failure. A minimal failing test is more valuable than reading the code because it precisely defines the gap between actual and expected behaviour.

```python
# You observe: get_overdue_tasks returns t2 (a completed task) as overdue
# Write the failing test first:
def test_completed_task_not_returned_as_overdue() -> None:
    t2 = Task(id=uuid4(), title="T2", priority=1, status="completed",
              due_date=date(2024, 1, 1), assignee=None)
    result = get_overdue_tasks([t2], today=date(2024, 6, 1))
    assert result == [], f"Expected [], got {result}"
```

If the test passes, the bug is elsewhere. If it fails, you have precisely identified the defect.

**Step 2: Map the code to the specification**

Read the generated code section by section, checking each behaviour claim in the specification against the implementation. Mark which claims are satisfied and which are not.

```python
# Generated code under review:
def get_overdue_tasks(tasks: list[Task], today: date | None = None) -> list[Task]:
    if today is None:
        today = date.today()
    # BUG: missing status filter — returns completed tasks too
    return sorted(
        [t for t in tasks if t.due_date and t.due_date < today],
        key=lambda t: t.due_date,
    )
```

The specification said: *"status not in ('completed', 'cancelled')"*. The implementation omits that condition. The bug is a missing constraint, not a logic error.

**Step 3: Categorise the bug**

| Category | Description | Fix approach |
|---|---|---|
| **Missing constraint** | Spec had a rule; implementation ignored it | Add the missing condition |
| **Wrong interpretation** | Implementation chose one meaning of an ambiguous spec | Clarify the spec; regenerate |
| **Hallucinated behaviour** | Implementation does something not in the spec | Remove the extra behaviour |
| **Edge case gap** | Spec didn't cover this case; implementation guessed wrong | Add the case to the spec; regenerate or patch |
| **API misuse** | Implementation calls a function with wrong arguments | Provide correct signatures in spec; patch |

**Step 4: Decide: patch or regenerate**

For a missing constraint (Step 3 category): patch the specific line. The fix is mechanical.

For a wrong interpretation: the specification was ambiguous. Update the specification with the clarification, regenerate the function, and re-run the full evaluation suite. A patched-ambiguous-spec function will likely re-exhibit the same bug when the code is next modified.

**Step 5: Add the failing case to the specification**

Regardless of which fix path you take, add the failing case as an explicit example in the specification. This prevents the same bug from appearing in future regenerations.

```python
# Add to spec Examples section:
# t_completed: completed task with past due date — must NOT be returned
# get_overdue_tasks([t_completed], today=date(2024, 6, 1)) == []
```

### 7.7.3 Using AI to Assist Debugging

You can use AI to help diagnose a bug — with the appropriate critical eye:

```python
import anthropic

client = anthropic.Anthropic()


def ai_debug_assist(specification: str, code: str, failing_test: str) -> str:
    """Ask the model to identify why the code fails the test."""
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="You are a senior engineer debugging a function. "
               "Be specific about which line is wrong and why. "
               "Do not rewrite the whole function — identify the minimal fix.",
        messages=[
            {
                "role": "user",
                "content": f"""This function fails the test below.
Identify the exact bug (line number and reason) and suggest the minimal fix.

SPECIFICATION:
{specification}

CODE:
{code}

FAILING TEST:
{failing_test}""",
            }
        ],
    )
    return response.content[0].text
```

The model is often effective at identifying missing conditions and off-by-one errors. It is less reliable at bugs that require understanding the broader system context. Always verify its diagnosis against the failing test before applying the fix.

