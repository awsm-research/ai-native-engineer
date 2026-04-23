# Chapter 8: Verify — Ensuring Correctness in AI-Generated Software

*"AI-generated code is a hypothesis, not a solution. Verification is the scientific method applied to software."*

---

**Learning Objectives**

By the end of this chapter, you will be able to:

1. Articulate why verification is the critical skill in AI-native engineering, and what the new verification burden looks like.
2. Identify what must be verified in AI-generated code: functional correctness, edge cases, security, performance, and licence provenance.
3. Apply evaluation-driven development (EDD): define criteria before generating, then evaluate systematically.
4. Use automated testing as the first verification gate, including AI-generated test cases from specs.
5. Apply static analysis and linting tools (Ruff, mypy, Semgrep, Bandit) within an agentic loop.
6. Use LLM-as-judge techniques to have AI review AI-generated code, and understand their limitations.
7. Conduct a security review of AI-generated code against the OWASP Top 10.
8. Identify what still requires human judgment: architecture alignment, business logic, and ethical concerns.
9. Integrate verification gates into CI/CD pipelines.

---

## 8.1 Why Verification is the Critical Skill

Software engineers have always needed to verify their work. But when a human writes code, the act of writing is itself a form of reasoning — the developer thinks through the problem, considers alternatives, and makes deliberate choices. Mistakes are introduced, but they are typically local and traceable to specific misunderstandings. When an AI agent writes code, that relationship between reasoning and output is severed. The model produces plausible-looking text that often works — but "often" is not "always," and in software, the difference between 95% correct and 100% correct can be catastrophic.

The research supports treating AI-generated code as a hypothesis to be tested rather than a solution to be deployed. [Pearce et al. (2021)](https://arxiv.org/abs/2108.09293) found that approximately 40% of GitHub Copilot suggestions contain bugs. [Perry et al. (2022)](https://arxiv.org/abs/2211.03622) found that 25–40% of security-relevant code produced by LLM assistants is insecure, and that developers using AI assistants were significantly more likely to introduce security vulnerabilities than those working without them. These are not edge cases. They are the baseline.

The speed-versus-rigour tension is the central failure mode of AI-native engineering. AI makes code generation fast — sometimes dramatically so. A function that might take an experienced developer thirty minutes to write carefully can be generated in seconds. This speed creates a psychological pressure to skip verification: the code looks correct, the tests seem like redundant work, and there is always more to build. Teams that succumb to this pressure accumulate a debt of unverified assumptions. When those assumptions are wrong — and some fraction of them always are — the bugs are harder to find because no one knows where the AI's reasoning went astray.

The new verification burden is structural, not incidental. If an AI agent can generate ten functions in the time it previously took to write one, then the volume of code requiring verification has increased by an order of magnitude. Manual code review at that volume is not feasible. This means verification must itself be partially automated: tests, static analysis, and LLM-as-judge techniques must absorb most of the verification load, with human judgment applied selectively to the decisions that carry the highest risk.

The key insight of this chapter is that verification skill is what separates engineers who benefit from AI from engineers who are harmed by it. An engineer who can define precise evaluation criteria, write tests that probe the specification rather than the implementation, and interpret static analysis results is positioned to use AI as a force multiplier. An engineer who generates code and deploys it on the basis that it looked right is building on sand.

---

## 8.2 What to Verify

Effective verification requires knowing what to look for. AI-generated code can fail along multiple dimensions simultaneously, and catching one class of failure does not mean the code is sound overall. The following table structures the verification space:

| Dimension | What to check | Primary tool |
|---|---|---|
| Functional correctness | Does it do what the spec says? | Automated tests |
| Edge cases | Boundary values, empty input, None, overflow | Tests + manual review |
| Security | Injection, auth, secrets, OWASP Top 10 | Static analysis + SAST |
| Type correctness | Are type annotations correct and consistent? | mypy --strict |
| Style and conventions | Does it match the project conventions? | ruff, linting |
| Performance | O(n²) where O(n) is expected? Memory leaks? | Profiling, code review |
| Licence provenance | Is generated code derived from GPL code? | Legal review, tooling |

**Functional correctness** is the minimum threshold. A function that does not do what its specification says is simply broken, regardless of how clean the code looks. Functional correctness must be verified against the original specification, not against the code — testing the code against itself is circular and reveals nothing.

**Edge cases** are where AI-generated code fails disproportionately. Language models are trained to produce typical patterns, so they handle the typical case well. Edge cases — empty lists, None inputs, negative numbers passed to functions expecting positive values, maximum integer values, concurrent access — are underrepresented in training data and in the model's implicit assumptions. Every spec implies edge cases; they must be made explicit in the test suite.

**Security** is covered in detail in section 8.7. The critical point here is that security failures are often not visible in code review: a SQL query that looks correct can be injectable; a file path that looks fine can traverse directories. Security verification requires dedicated tools and a specific mindset.

**Type correctness** matters because AI models produce type annotations that look plausible but are sometimes wrong. A function annotated as returning `str` that can actually return `None` in some branch will pass casual review and fail at runtime. Running `mypy --strict` catches this class of error systematically.

**Style and conventions** are a proxy for maintainability. Code that violates the project's established patterns is harder to review, harder to modify, and harder to debug. The AI does not know your project's conventions unless you tell it, and even then it may drift. Running `ruff` on every generated file catches the obvious cases.

**Performance** is the dimension most often deferred and most often regretted. AI models tend to produce correct, naive implementations — correct in the sense of producing the right answer, naive in the sense of choosing the obvious algorithm rather than the efficient one. An O(n²) implementation that works correctly on a dataset of 100 items will silently destroy performance when the dataset grows to 100,000. Performance verification requires both code review — looking for nested loops, redundant database queries, repeated computations — and profiling against realistic data volumes.

**Licence provenance** is the most commonly overlooked dimension. AI models are trained on large corpora of public code, including code licensed under the GPL and other copyleft licences. A model trained on GPL code may reproduce verbatim or near-verbatim segments of that code in its output. If such code is incorporated into a proprietary product without notice, the result may be a licence violation with real legal consequences. The current state of tooling for licence provenance in AI-generated code is immature, but the legal risk is real. Engineers should be particularly cautious with generated implementations of well-known algorithms, data structures, and utility functions, which are the code most likely to have been reproduced from specific sources.

---

## 8.3 Evaluation-Driven Development

Evaluation-Driven Development (EDD) is the disciplined practice of defining what success looks like before asking an AI to produce it. The workflow is:

```
1. Define evaluation criteria from the specification
2. Write tests and evaluation artefacts BEFORE generating
3. Generate the implementation
4. Run evaluation — pass or fail
5. If fail: diagnose, refine the spec or prompt, repeat from step 3
```

The reason "before generating" is essential is anchoring bias. When you read the generated code first and then write tests, you unconsciously write tests that reflect what the code does rather than what the specification requires. The tests pass, the function ships, and the bug is discovered in production when a user encounters the case the code handles wrong. This is not hypothetical; it is the standard failure mode of teams that treat AI as a shortcut through the discipline of test-first development.

EDD is TDD extended to the AI context. In test-driven development, the test suite is the specification made executable. The code is written to make the tests pass. In EDD, the same discipline applies, but the code is written by an AI agent rather than a human. The tests serve the same function: they define correct behaviour precisely enough that any implementation can be evaluated against them objectively.

Consider a concrete example. Suppose the specification for a task-management system includes the following requirement:

> `assign_task(task_id, assignee_email, assignee_role)` — assigns a task to a user. Raises `ValueError` if the task does not exist. Raises `PermissionError` if the assignee's role is `"viewer"`. Returns the updated task object. If the task is already assigned, reassigns it without error.

Before invoking any AI agent, the engineer writes the test suite:

```python
# tests/test_assign_task.py
import pytest
from tasks import assign_task, Task

class TestAssignTask:
    def test_assigns_task_to_valid_user(self, db_with_task):
        """Spec: returns the updated task object when assignment succeeds."""
        task = assign_task(
            task_id="task-001",
            assignee_email="alice@example.com",
            assignee_role="editor",
        )
        assert isinstance(task, Task)
        assert task.assignee_email == "alice@example.com"

    def test_raises_on_nonexistent_task(self, db_with_task):
        """Spec: raises ValueError if the task does not exist."""
        with pytest.raises(ValueError, match="task-999"):
            assign_task(
                task_id="task-999",
                assignee_email="alice@example.com",
                assignee_role="editor",
            )

    def test_raises_on_viewer_role(self, db_with_task):
        """Spec: raises PermissionError if assignee_role is 'viewer'."""
        with pytest.raises(PermissionError):
            assign_task(
                task_id="task-001",
                assignee_email="viewer@example.com",
                assignee_role="viewer",
            )

    def test_reassigns_already_assigned_task(self, db_with_assigned_task):
        """Spec: if already assigned, reassigns without error."""
        task = assign_task(
            task_id="task-002",
            assignee_email="bob@example.com",
            assignee_role="editor",
        )
        assert task.assignee_email == "bob@example.com"

    def test_returns_task_object_not_none(self, db_with_task):
        """Spec: return type is the updated task object, not None."""
        result = assign_task(
            task_id="task-001",
            assignee_email="alice@example.com",
            assignee_role="editor",
        )
        assert result is not None
```

Only after this test suite exists does the engineer invoke the AI agent with a prompt that includes the specification. The agent generates `assign_task`. The tests run. If they pass, the function is provisionally correct for the specified cases. If they fail, the failure output is fed back to the agent as a diagnostic: "Test `test_raises_on_viewer_role` failed — your implementation does not raise `PermissionError` for viewer-role assignees." The loop continues until the tests pass.

The discipline is in the ordering. The tests must precede the code.

---

## 8.4 Automated Testing as the First Gate

The test suite is the first and most important verification gate. If a function does not pass its tests, nothing else about it matters. Static analysis, LLM review, and human inspection are all supplementary; tests are the ground truth.

**Test-first** is not optional in AI-native engineering. The temptation to generate first and test afterward is strong because the generated code often appears to work. Resist it. The tests that matter most are the ones written before you know what the implementation looks like.

AI agents can be used to generate test cases from specifications, which is one of the more reliable uses of LLMs in the verification workflow. A model given a precise specification can enumerate test cases systematically, including edge cases a human reviewer might overlook. The following function uses the Anthropic API to generate pytest test stubs from a specification:

```python
import anthropic

def generate_test_cases(spec: str) -> str:
    """
    Given a function specification, returns a string containing
    pytest test stubs covering the spec's requirements and edge cases.
    """
    client = anthropic.Anthropic()

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        system=(
            "You are a senior software engineer writing pytest test suites. "
            "Given a function specification, generate a complete pytest test class "
            "that covers: (1) all stated requirements, (2) boundary values, "
            "(3) error conditions, (4) type edge cases (None, empty string, zero). "
            "Write precise assertions. Do not write tests that only check that no "
            "exception is raised — check the actual return values. "
            "Output only the Python code, no explanation."
        ),
        messages=[
            {
                "role": "user",
                "content": f"Write a pytest test suite for the following specification:\n\n{spec}",
            }
        ],
    )

    return response.content[0].text


if __name__ == "__main__":
    spec = """
    filter_tasks(tasks: list[dict], status: str) -> list[dict]

    Filters a list of task dictionaries to those whose 'status' field matches
    the given status string (case-insensitive). Returns a new list; does not
    modify the input. If tasks is empty, returns an empty list. If no tasks
    match, returns an empty list. Raises TypeError if tasks is not a list.
    Raises ValueError if status is an empty string.
    """
    print(generate_test_cases(spec))
```

**Coverage as a signal, not a goal.** 100% line coverage achieved with weak assertions is worse than 70% coverage with precise assertions. A test that calls `filter_tasks([], "done")` and asserts only that it does not raise an exception contributes nothing to the verification of the return value. Every assertion should correspond to a specific claim made by the specification.

**Mutation testing** verifies the tests themselves. Tools like [mutmut](https://github.com/boxed/mutmut) and [pytest-mutagen](https://github.com/d-miketa/pytest-mutagen) introduce small mutations into the source code — changing `>` to `>=`, negating a condition, replacing a return value — and check whether the test suite detects them. If a mutation passes all tests, the test suite has a gap. Mutation testing is expensive on large codebases, but running it on the most critical functions ensures the tests are actually doing their job.

A complete test suite for `filter_tasks` against its specification:

```python
# tests/test_filter_tasks.py
import pytest
from tasks import filter_tasks


class TestFilterTasks:
    """Tests for filter_tasks — each test corresponds to one spec clause."""

    def test_returns_matching_tasks(self):
        """Spec: returns tasks whose status matches the given string."""
        tasks = [
            {"id": 1, "status": "done"},
            {"id": 2, "status": "open"},
            {"id": 3, "status": "done"},
        ]
        result = filter_tasks(tasks, "done")
        assert len(result) == 2
        assert all(t["status"] == "done" for t in result)

    def test_case_insensitive_match(self):
        """Spec: matching is case-insensitive."""
        tasks = [{"id": 1, "status": "Done"}, {"id": 2, "status": "DONE"}]
        result = filter_tasks(tasks, "done")
        assert len(result) == 2

    def test_returns_new_list_not_mutation(self):
        """Spec: returns a new list; does not modify the input."""
        tasks = [{"id": 1, "status": "done"}]
        original_length = len(tasks)
        result = filter_tasks(tasks, "open")
        assert len(tasks) == original_length  # input not modified
        assert result is not tasks            # new list returned

    def test_empty_input_returns_empty_list(self):
        """Spec: if tasks is empty, returns an empty list."""
        assert filter_tasks([], "done") == []

    def test_no_match_returns_empty_list(self):
        """Spec: if no tasks match, returns an empty list."""
        tasks = [{"id": 1, "status": "open"}]
        assert filter_tasks(tasks, "done") == []

    def test_raises_type_error_on_non_list(self):
        """Spec: raises TypeError if tasks is not a list."""
        with pytest.raises(TypeError):
            filter_tasks({"id": 1, "status": "done"}, "done")

    def test_raises_value_error_on_empty_status(self):
        """Spec: raises ValueError if status is an empty string."""
        with pytest.raises(ValueError):
            filter_tasks([{"id": 1, "status": "done"}], "")
```

Each test is labelled with the specific spec clause it verifies. This makes it trivial to identify which requirement a failing test corresponds to when the agent's implementation is under review.

---

## 8.5 Static Analysis and Linting

Static analysis tools examine code without executing it. They are fast, deterministic, and able to catch entire classes of bugs that would otherwise require running the code under specific conditions. In an agentic workflow, they function as a cheap first filter: any generated code that fails static analysis is rejected before human review time is spent on it.

**Ruff** is a Python linter and formatter written in Rust. It runs in milliseconds on large codebases and covers the rules from Flake8, isort, pycodestyle, and dozens of plugins. For AI-generated code, the most useful rules are those that catch logical errors rather than style preferences: unused imports that suggest copy-paste from unrelated code, undefined variables, unreachable code after a return statement, and comparison patterns that are always true or always false.

**mypy --strict** enforces the type system. AI-generated code frequently has annotation errors that are invisible to casual reading: a function that returns `Optional[str]` where the spec requires `str`, a list typed as `list[str]` that in practice receives `list[Any]`, a method call on a value that could be `None`. Running mypy in strict mode — which requires explicit annotations throughout and disallows implicit `Any` — catches these before runtime.

**Bandit** is a security-focused linter that scans Python code for common vulnerability patterns. It flags SQL queries built with string formatting, calls to `subprocess` with `shell=True`, use of `pickle` for deserialisation, MD5 and SHA1 for cryptographic hashing, and hardcoded password literals. AI models produce these patterns routinely because they appear frequently in training data.

**Semgrep** is a pattern-based static analysis tool that can enforce custom rules. Where Bandit covers a fixed set of known-bad patterns, Semgrep allows teams to codify their own security and architectural policies. A rule that prevents the use of `subprocess.run(shell=True)` in any file can be written in Semgrep's YAML format and enforced automatically. Rules can also capture project-specific anti-patterns: calling a deprecated internal API, constructing URLs by concatenation, or accessing environment variables outside the designated configuration module.

The following shell script integrates all four tools into a single verification step:

```bash
#!/bin/bash
# verify.sh — run after each agent generation step
# Exits with a non-zero status if any check fails.
set -e

echo "=== Ruff: linting and formatting check ==="
ruff check src/
ruff format --check src/

echo "=== mypy: type checking (strict mode) ==="
mypy src/ --strict --no-error-summary

echo "=== Bandit: security patterns ==="
bandit -r src/ -ll

echo "=== Semgrep: custom rules ==="
semgrep --config=.semgrep/ src/ --error

echo "All static checks passed."
```

As a concrete example of what these tools catch: Bandit will flag the following AI-generated code immediately, before any human reviewer sees it:

```python
# AI-generated: plausible-looking but flagged by Bandit (B608)
def get_user(email: str) -> dict:
    query = f"SELECT * FROM users WHERE email = '{email}'"
    return db.execute(query).fetchone()
```

Bandit raises `B608: Possible SQL injection via string-based query construction`. The safe version — using parameterised queries — is not the pattern the model defaulted to because parameterised queries are more verbose and appear less frequently in tutorial-style training data.

---

## 8.6 LLM-as-Judge: AI Reviews AI

Static analysis is blind to semantic errors. A function can pass every linting and type-checking rule while implementing the wrong algorithm, handling the wrong cases, or subtly misunderstanding the specification. LLM-as-judge is a technique in which a second LLM call is used to review the generated code against the specification, looking for semantic issues that tools cannot detect.

[Zheng et al. (2023)](https://arxiv.org/abs/2306.05685) demonstrated that LLMs can serve as reliable judges of code and text quality under specific conditions, but also identified systematic failure modes: the judge tends to prefer verbose responses, is biased toward agreeing with the first argument it encounters, and shares the blind spots of the generator when both are the same model. These findings establish both the utility and the limits of the technique.

LLM-as-judge helps most with:

- **Wrong algorithm**: the function produces correct output on the happy path but will produce incorrect results for a class of inputs the judge can identify.
- **Missed error handling**: the spec requires a specific exception to be raised; the generated code does not raise it.
- **Misunderstood semantics**: the function interprets a specification clause differently from the intended meaning.

LLM-as-judge fails most often with:

- **Shared blind spots**: if the generator and the judge are the same model, they will make the same mistakes. The judge will often approve code that has the same error the generator introduced.
- **False positives on style**: the judge may flag code as incorrect because it looks unusual, when in fact it is correct.
- **Over-confidence**: LLMs frequently express high confidence in their assessments even when they are wrong.

The calibration trick is to use a stricter system prompt for the judge than for the generator, and to ask for specific failure modes rather than general quality. A judge asked "is this code correct?" will frequently say yes. A judge asked "does this code correctly handle None inputs, empty lists, and negative numbers?" will be more precise.

The following implementation returns a structured verdict:

```python
import anthropic
from dataclasses import dataclass, field


@dataclass
class ReviewResult:
    verdict: str           # "PASS", "FAIL", or "UNCERTAIN"
    issues: list[str]      # specific problems found
    suggested_fixes: list[str]
    confidence: str        # "HIGH", "MEDIUM", or "LOW"


def llm_review(spec: str, code: str) -> ReviewResult:
    """
    Uses an LLM to review generated code against its specification.
    Returns a structured ReviewResult.
    """
    client = anthropic.Anthropic()

    prompt = f"""You are a strict code reviewer. Your job is to find bugs, not to approve code.

SPECIFICATION:
{spec}

GENERATED CODE:
{code}

Review the code against the specification with maximum scepticism. Check specifically:
1. Does every requirement in the spec have a corresponding implementation?
2. Does the code handle None inputs, empty collections, and boundary values correctly?
3. Are all error conditions (exceptions) raised exactly as specified?
4. Is the return type and value correct for all code paths?
5. Are there any off-by-one errors, type mismatches, or wrong comparison operators?

Respond in this exact format:
VERDICT: PASS|FAIL|UNCERTAIN
CONFIDENCE: HIGH|MEDIUM|LOW
ISSUES:
- <issue 1, or "none" if no issues>
- <issue 2>
SUGGESTED FIXES:
- <fix 1, or "none" if no fixes needed>
- <fix 2>"""

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system=(
            "You are a code reviewer whose only job is to find errors. "
            "Err on the side of flagging potential issues. "
            "Never approve code unless you are certain it is correct."
        ),
        messages=[{"role": "user", "content": prompt}],
    )

    text = response.content[0].text
    lines = text.strip().split("\n")

    verdict = "UNCERTAIN"
    confidence = "LOW"
    issues: list[str] = []
    suggested_fixes: list[str] = []
    current_section = None

    for line in lines:
        line = line.strip()
        if line.startswith("VERDICT:"):
            verdict = line.split(":", 1)[1].strip()
        elif line.startswith("CONFIDENCE:"):
            confidence = line.split(":", 1)[1].strip()
        elif line == "ISSUES:":
            current_section = "issues"
        elif line == "SUGGESTED FIXES:":
            current_section = "fixes"
        elif line.startswith("- ") and current_section == "issues":
            issue = line[2:]
            if issue.lower() != "none":
                issues.append(issue)
        elif line.startswith("- ") and current_section == "fixes":
            fix = line[2:]
            if fix.lower() != "none":
                suggested_fixes.append(fix)

    return ReviewResult(
        verdict=verdict,
        issues=issues,
        suggested_fixes=suggested_fixes,
        confidence=confidence,
    )
```

When using this in practice, treat `PASS` with `LOW` confidence identically to `UNCERTAIN`. The judge's output is one signal among several; it does not replace tests or static analysis. Reserve human escalation for any result where the judge returns `FAIL` or `UNCERTAIN` with specific, actionable issues.

---

## 8.7 Security Review of AI-Generated Code

AI-generated code is particularly security-risky for three compounding reasons. First, the models are trained on large quantities of public code, and public code contains a substantial proportion of insecure patterns — Stack Overflow answers that omit sanitisation, tutorial code that hardcodes credentials for simplicity, legacy code that predates modern security practices. Second, the model optimises for functional correctness and code plausibility, not security; it produces the most common-looking implementation, which is frequently not the most secure one. Third, the output looks competent and professional, which makes it harder for reviewers to approach with appropriate scepticism.

The following are the OWASP Top 10 vulnerability classes most commonly produced by AI code generators, with examples of each.

**SQL Injection (CWE-89)** is the most common AI-generated security flaw. The model defaults to string formatting because it is the most natural way to construct dynamic queries in most tutorial contexts:

```python
# AI-generated — SQL injectable
def get_task(task_id: str, owner_email: str) -> dict | None:
    query = f"SELECT * FROM tasks WHERE id = '{task_id}' AND owner = '{owner_email}'"
    return db.execute(query).fetchone()
```

An attacker can set `owner_email` to `' OR '1'='1` and retrieve all tasks. The safe version uses parameterised queries:

```python
# Safe — parameterised query
def get_task(task_id: str, owner_email: str) -> dict | None:
    query = "SELECT * FROM tasks WHERE id = ? AND owner = ?"
    return db.execute(query, (task_id, owner_email)).fetchone()
```

**Hardcoded secrets (CWE-798)** appear when the AI generates complete working examples. A model asked to write database connection code will often include a placeholder that looks like a real credential:

```python
# AI-generated — hardcoded secret
DATABASE_URL = "postgresql://admin:SuperSecret123@localhost:5432/tasks"
API_KEY = "sk-proj-abc123def456"
```

These literals end up in version control. The safe pattern reads credentials from environment variables:

```python
# Safe — environment variables
import os
DATABASE_URL = os.environ["DATABASE_URL"]
API_KEY = os.environ["ANTHROPIC_API_KEY"]
```

**Command injection (CWE-78)** is produced when the AI uses `subprocess` with `shell=True` for convenience:

```python
# AI-generated — command injectable
import subprocess
def run_report(report_name: str) -> str:
    result = subprocess.run(
        f"python reports/{report_name}.py",
        shell=True, capture_output=True, text=True
    )
    return result.stdout
```

If `report_name` is set to `; rm -rf /`, the consequences are severe. The safe version avoids shell interpolation:

```python
# Safe — no shell, validated input
import subprocess
import re
def run_report(report_name: str) -> str:
    if not re.match(r'^[a-zA-Z0-9_-]+$', report_name):
        raise ValueError(f"Invalid report name: {report_name}")
    result = subprocess.run(
        ["python", f"reports/{report_name}.py"],
        shell=False, capture_output=True, text=True
    )
    return result.stdout
```

**Path traversal (CWE-22)** occurs when the AI constructs file paths from user input without validation:

```python
# AI-generated — path traversal vulnerable
def get_upload(filename: str) -> bytes:
    path = f"./uploads/{filename}"
    with open(path, "rb") as f:
        return f.read()
```

An attacker sets `filename` to `../../etc/passwd`. The safe version resolves and validates the path:

```python
# Safe — path confined to uploads directory
from pathlib import Path
def get_upload(filename: str) -> bytes:
    uploads_dir = Path("./uploads").resolve()
    target = (uploads_dir / filename).resolve()
    if not str(target).startswith(str(uploads_dir)):
        raise PermissionError("Access outside uploads directory denied")
    return target.read_bytes()
```

**Prompt injection via generated code** is an emerging class of vulnerability specific to AI-native applications. It occurs when AI-generated code passes user-controlled input directly to an LLM without sanitisation. The model generating the code does not consider that the user might craft input designed to override the system prompt, exfiltrate context, or manipulate the downstream agent's behaviour.

For automated secrets scanning, integrate `detect-secrets` into your pre-commit and CI workflow:

```bash
# Scan for secrets in all Python files
detect-secrets scan src/ --baseline .secrets.baseline

# Audit the baseline — any new secrets fail the build
detect-secrets audit .secrets.baseline
```

**Security review checklist for every AI-generated PR:**

1. Search for any string containing `password`, `secret`, `key`, `token`, or `credential` that is not read from an environment variable or secrets manager.
2. Audit every database query for parameterisation. No f-strings or `.format()` calls in SQL.
3. Audit every `subprocess` call. `shell=True` requires explicit justification.
4. Audit every file path constructed from user input for bounds checking.
5. Check all serialisation and deserialisation: avoid `pickle.loads` on untrusted input.
6. Verify all cryptographic operations use modern algorithms: no MD5 or SHA1 for security purposes, no `random` module for security tokens.
7. Check all user-facing LLM calls for prompt injection vectors.

---

## 8.8 What Still Requires Human Judgment

Automated verification — tests, static analysis, LLM-as-judge, security scanning — can cover a large fraction of the risk in AI-generated code. But there is a residual category that these tools cannot reach, and this is where human judgment is not optional but essential.

**Architecture alignment** is invisible to automated tools. A function that passes every test and every linter may nonetheless introduce an inconsistent pattern into the codebase: a new abstraction that duplicates an existing one, a dependency inversion that contradicts the project's established structure, an interface that will be difficult to extend when the next requirement arrives. These failures are not bugs in the traditional sense — the function is correct and the tests pass — but they accumulate into a codebase that becomes progressively harder to maintain. Only an engineer with context about the system's intended architecture can evaluate this.

**Business logic correctness** is distinct from specification correctness. A specification can be technically correct — all the clauses are consistent, the edge cases are covered — and still not capture what the business actually wants. A discount function that correctly implements the written spec may apply discounts to already-discounted items in a way that is technically correct but commercially unintended. The engineer who understands both the code and the business context is the only one who can catch this class of error.

**Ethical and legal concerns** sit entirely outside the scope of automated verification. Code that implements a feature correctly may enable discrimination, privacy violation, or a terms-of-service breach with a third-party platform. An AI agent generating a recommendation algorithm has no awareness of the protected characteristics that anti-discrimination law prohibits using, no awareness of GDPR's requirements for data minimisation, and no awareness of the contractual constraints on how user data may be processed. These require human review with specific expertise.

**Long-range dependencies** are correctness properties that are invisible at the function level. A function that is correct in isolation may interact with shared mutable state, caching, or concurrency in ways that produce incorrect system-level behaviour. Integration testing addresses some of this, but the reasoning that connects function-level behaviour to system-level guarantees requires human understanding of the full architecture.

The rule is: human review is not optional for anything that reaches production. Automated verification narrows the surface area that human reviewers must examine — it eliminates the obvious errors, the style violations, the type mismatches, and many of the security patterns — but it does not eliminate the need for human judgment on the decisions that carry the highest risk.

---

## 8.9 CI/CD Integration: Verification Gates in the Pipeline

Verification is only as effective as its enforcement. A test suite that engineers are expected to run locally before pushing will be run inconsistently. Static analysis that is advisory rather than blocking will be ignored under deadline pressure. The verification pipeline must be automated, mandatory, and integrated into the development workflow at the point where it can actually prevent bad code from progressing.

The **gate model** is a staged pipeline where each gate must pass before the next runs. The ordering matters: cheap, fast checks run first so that slow, expensive checks are not wasted on code that fails the basics.

```
push → lint (seconds) → type check (seconds) → unit tests (seconds to minutes)
  → integration tests (minutes) → security scan (minutes) → PR gate → merge
```

Fast gates — linting, type checking, unit tests — run on every push. Slow gates — integration tests, security scans, LLM-as-judge review — run on pull requests. The PR gate is the last line of automated defence before human review; it must include everything.

The following GitHub Actions workflow implements this pipeline:

```yaml
name: Verify AI-Generated Code
on: [push, pull_request]

jobs:
  fast-gates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Ruff lint
        run: ruff check src/

      - name: Ruff format check
        run: ruff format --check src/

      - name: Type check (strict)
        run: mypy src/ --strict

      - name: Unit tests
        run: pytest tests/unit/ -v --tb=short

  security-gates:
    runs-on: ubuntu-latest
    needs: fast-gates
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Bandit security scan
        run: bandit -r src/ -ll

      - name: Semgrep custom rules
        run: semgrep --config=.semgrep/ src/ --error

      - name: Detect secrets
        run: detect-secrets scan src/ --baseline .secrets.baseline

      - name: Integration tests
        run: pytest tests/integration/ -v --tb=short
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
```

The **PR review agent pattern** extends this pipeline with an AI-generated review comment. A GitHub Actions step runs `llm_review` (from section 8.6) against each changed file and its corresponding specification, then posts a structured comment to the PR using the GitHub API. This gives human reviewers a baseline assessment before they read the code, with specific issues flagged and the LLM's confidence level made explicit.

Balancing rigour with velocity requires conscious design. The fast gates run in under two minutes on most Python projects; there is no velocity argument for skipping them. The slow gates — integration tests and security scans — add five to fifteen minutes and are appropriate on pull requests but wasteful on every commit to a feature branch. The LLM-as-judge review adds API cost and latency; it is most valuable on the PR gate, where it catches semantic issues before human review.

---

## 8.10 Tutorial: Building a Verification Pipeline

This tutorial assembles the techniques from the chapter into a complete end-to-end pipeline. The pipeline generates a function, then runs each verification step in sequence, using the output of each step to inform the next.

```python
"""
tutorial_pipeline.py — end-to-end verification pipeline for AI-generated code.

Demonstrates: generation → static analysis → testing → LLM review → acceptance.
"""

import subprocess
import sys
import tempfile
import textwrap
from pathlib import Path

import anthropic

from llm_review import ReviewResult, llm_review


SPEC = """
calculate_discount(price: float, user_tier: str) -> float

Returns the discounted price for a given user tier:
  - "gold": 20% discount
  - "silver": 10% discount
  - "bronze": 5% discount
  - any other tier: no discount (0%)

Raises ValueError if price is negative.
Raises ValueError if price is zero.
Returns a float rounded to 2 decimal places.
"""

TESTS = """
import pytest
from generated_function import calculate_discount

class TestCalculateDiscount:
    def test_gold_tier_twenty_percent(self):
        assert calculate_discount(100.0, "gold") == 80.0

    def test_silver_tier_ten_percent(self):
        assert calculate_discount(100.0, "silver") == 90.0

    def test_bronze_tier_five_percent(self):
        assert calculate_discount(100.0, "bronze") == 95.0

    def test_unknown_tier_no_discount(self):
        assert calculate_discount(100.0, "unknown") == 100.0

    def test_raises_on_negative_price(self):
        with pytest.raises(ValueError):
            calculate_discount(-10.0, "gold")

    def test_raises_on_zero_price(self):
        with pytest.raises(ValueError):
            calculate_discount(0.0, "gold")

    def test_result_rounded_to_two_decimals(self):
        result = calculate_discount(99.99, "silver")
        assert result == round(result, 2)

    def test_returns_float(self):
        assert isinstance(calculate_discount(50.0, "gold"), float)
"""


def generate_implementation(spec: str) -> str:
    """Step 1: Generate the function from the specification."""
    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system=(
            "You are a Python engineer. Write a single, complete Python function "
            "that implements the given specification. Include type annotations. "
            "Output only the Python code, no explanation, no markdown fences."
        ),
        messages=[{"role": "user", "content": f"Implement:\n\n{spec}"}],
    )
    return response.content[0].text


def run_static_analysis(code: str, work_dir: Path) -> tuple[bool, str]:
    """Step 2: Run Ruff and mypy on the generated code."""
    source_file = work_dir / "generated_function.py"
    source_file.write_text(code)

    results = []
    passed = True

    for cmd in [
        ["ruff", "check", str(source_file)],
        ["mypy", str(source_file), "--strict", "--no-error-summary"],
    ]:
        result = subprocess.run(cmd, capture_output=True, text=True)
        if result.returncode != 0:
            passed = False
            results.append(f"FAIL [{' '.join(cmd[:1])}]:\n{result.stdout}{result.stderr}")
        else:
            results.append(f"PASS [{' '.join(cmd[:1])}]")

    return passed, "\n".join(results)


def run_tests(code: str, tests: str, work_dir: Path) -> tuple[bool, str]:
    """Step 3: Run the pre-written test suite against the generated code."""
    (work_dir / "generated_function.py").write_text(code)
    (work_dir / "test_generated.py").write_text(tests)

    result = subprocess.run(
        [sys.executable, "-m", "pytest", str(work_dir / "test_generated.py"), "-v", "--tb=short"],
        capture_output=True,
        text=True,
        cwd=work_dir,
    )
    passed = result.returncode == 0
    return passed, result.stdout + result.stderr


def pipeline(spec: str, tests: str, max_iterations: int = 3) -> str | None:
    """
    Full verification pipeline. Returns the verified code or None if
    all iterations fail.
    """
    with tempfile.TemporaryDirectory() as tmp:
        work_dir = Path(tmp)
        diagnosis = ""

        for iteration in range(1, max_iterations + 1):
            print(f"\n{'='*60}")
            print(f"Iteration {iteration}/{max_iterations}")
            print('='*60)

            # Step 1: Generate
            prompt = spec if not diagnosis else f"{spec}\n\nPrevious attempt failed:\n{diagnosis}\nFix these issues."
            print("Generating implementation...")
            code = generate_implementation(prompt)
            print(f"Generated:\n{code}\n")

            # Step 2: Static analysis
            print("Running static analysis...")
            static_ok, static_output = run_static_analysis(code, work_dir)
            print(static_output)
            if not static_ok:
                diagnosis = f"Static analysis failed:\n{static_output}"
                continue

            # Step 3: Tests
            print("Running test suite...")
            tests_ok, test_output = run_tests(code, tests, work_dir)
            print(test_output)
            if not tests_ok:
                diagnosis = f"Tests failed:\n{test_output}"
                continue

            # Step 4: LLM-as-judge
            print("Running LLM review...")
            review: ReviewResult = llm_review(spec, code)
            print(f"Verdict: {review.verdict} (confidence: {review.confidence})")
            if review.issues:
                print(f"Issues: {review.issues}")
            if review.verdict == "FAIL" and review.confidence == "HIGH":
                diagnosis = f"LLM review flagged issues:\n" + "\n".join(review.issues)
                continue

            # All gates passed
            print(f"\nAll verification gates passed on iteration {iteration}.")
            return code

    print(f"\nVerification failed after {max_iterations} iterations.")
    return None


if __name__ == "__main__":
    verified_code = pipeline(SPEC, TESTS)
    if verified_code:
        print("\nFinal verified implementation:")
        print(verified_code)
    else:
        print("\nCould not produce a verified implementation.")
        sys.exit(1)
```

When you run this pipeline, you observe the agentic verification loop in action. On a first iteration, the generated code may pass static analysis but fail two test cases — for instance, failing to round the result and failing to raise `ValueError` on zero price. The failure output is fed back into the generation prompt on the second iteration with a specific diagnosis. On the second iteration, the tests pass, static analysis is clean, and the LLM review returns `PASS`. The function is accepted.

The key structural property of the pipeline is that a human engineer wrote the tests and the specification before any code was generated. The pipeline automates the evaluation of candidate implementations against those criteria. The criteria themselves are the human contribution.

---

## Chapter Summary

- **Verification is not optional and must be partially automated.** AI-generated code contains bugs at a rate of approximately 40% (Pearce et al., 2021) and security vulnerabilities at a rate of 25–40% (Perry et al., 2022). The volume of AI-generated code makes manual-only verification infeasible; automated testing, static analysis, and LLM-as-judge must absorb the bulk of the verification load.

- **Evaluation-Driven Development (EDD) is the foundational discipline.** Write the test suite and evaluation criteria before invoking the AI agent. This prevents anchoring bias — the tendency to write tests that pass the code rather than tests that test the specification — and ensures the AI's output is evaluated against an objective standard.

- **Static analysis tools catch deterministic error classes cheaply.** Ruff, mypy --strict, Bandit, and Semgrep together catch type errors, style violations, and security anti-patterns in seconds. They should be run after every generation step and fail the build on any finding above the configured threshold.

- **LLM-as-judge fills the gap between static analysis and semantic correctness, but has known failure modes.** The judge shares blind spots with the generator, tends toward false positives, and should be calibrated with a stricter system prompt. Its output is one signal among several, not a definitive verdict.

- **Human judgment remains essential for architecture alignment, business logic, and ethical concerns.** Automated verification narrows the surface area but cannot replace the engineer who understands the system's design, the business's intent, and the legal and ethical constraints on the code's behaviour.

---

## Review Questions

**1. (Conceptual)** Why must evaluation criteria be defined before code generation begins, rather than after the generated code has been reviewed?

When evaluation criteria are defined after the code is seen, engineers unconsciously write tests and checks that reflect what the code does rather than what the specification requires. This anchoring bias means that bugs introduced by the generator — including misinterpretations of the specification — are invisible in the tests because the tests were written to match the misinterpretation. Defining criteria first creates an objective standard against which the code is evaluated, independent of the code's actual implementation.

**2. (Applied)** You are verifying AI-generated `calculate_discount(price, user_tier)`. List five specific test cases that your test suite must include, based on the specification from section 8.10.

A thorough test suite should include at minimum: (1) a gold-tier user receives exactly a 20% discount on a known price, asserting the exact float value; (2) a silver-tier user receives exactly a 10% discount; (3) an unrecognised tier string receives no discount and the original price is returned; (4) a negative price raises `ValueError`; (5) a price of exactly zero raises `ValueError`. Additional high-value cases include verifying that the return value is rounded to exactly two decimal places (use a price that would produce rounding without it, such as 99.99 with a 10% discount), and verifying that the return type is `float` and not `int` or `None`.

**3. (Conceptual)** What are the limitations of LLM-as-judge? Under what conditions should you trust its verdict?

The primary limitations are: shared blind spots between generator and judge (if both are the same model, they make the same systematic errors); a tendency toward false positives — approving code that is incorrect because it looks plausible; and over-confidence in assessments, with high-confidence verdicts that are nonetheless wrong. Additionally, [Zheng et al. (2023)](https://arxiv.org/abs/2306.05685) identified position bias (the judge prefers the first argument it encounters) and verbosity bias (longer responses are rated more highly regardless of correctness).

Trust the verdict most when: the judge is a different, more capable model than the generator; the system prompt specifies exact failure modes to check rather than asking for general quality; the verdict is `FAIL` with high confidence and specific, verifiable issues attached; and the judge's findings align with static analysis output. Treat any `PASS` verdict as provisional rather than definitive — LLM-as-judge is a filter, not a certificate.

**4. (Applied)** The AI generates: `query = f"SELECT * FROM tasks WHERE assignee = '{email}'"`. Identify the vulnerability, name the CWE, and write the safe version.

The vulnerability is SQL injection — **CWE-89: Improper Neutralisation of Special Elements used in an SQL Command**. An attacker who controls the `email` parameter can inject arbitrary SQL. For example, setting `email` to `' OR '1'='1' --` causes the query to return all tasks regardless of assignee. The safe version uses a parameterised query:

```python
# Safe version — parameterised query
query = "SELECT * FROM tasks WHERE assignee = ?"
cursor.execute(query, (email,))
```

For database libraries that use `%s` placeholders (such as psycopg2), the equivalent is:
```python
query = "SELECT * FROM tasks WHERE assignee = %s"
cursor.execute(query, (email,))
```

The user-supplied value is passed as a separate argument and the database driver handles escaping, ensuring it can never be interpreted as SQL syntax.

**5. (Applied)** Design the verification pipeline for a PR that modifies the authentication middleware. Which gates are required? Which are optional? Justify your choices.

Authentication middleware is among the highest-risk code in any application. A failure in it can result in complete authentication bypass or privilege escalation.

**Required gates:**

- **Ruff and mypy --strict**: required on all PRs; style and type errors in security-critical code are unacceptable.
- **Bandit at high severity**: required; authentication code is precisely where hardcoded secrets, insecure hashing, and unsafe random number generation appear. Raise the severity threshold to flag all medium and high findings, not just high.
- **Semgrep with authentication-specific rules**: required; enforce rules preventing direct password comparison, requiring constant-time string comparison for secrets, and prohibiting any logging of authentication credentials.
- **Full unit test suite**: required; every authentication path — successful login, failed login, expired token, missing token, role escalation attempt — must have a test.
- **Integration tests against a real authentication flow**: required; authentication middleware cannot be adequately tested in isolation because its interaction with the session store, token validation library, and HTTP layer is the source of most real vulnerabilities.
- **LLM-as-judge review with a security-focused system prompt**: required; ask specifically whether timing attacks are possible (constant-time comparisons), whether token expiry is enforced, and whether the function correctly distinguishes between authentication failure and authorisation failure.
- **Human security review**: required; all authentication changes must be reviewed by an engineer with explicit security expertise before merging.

**Optional gates:**

- **Performance profiling**: optional; authentication latency matters but is not a primary concern for a PR that modifies correctness, not the hot path.
- **Detect-secrets scan**: useful but not required if Bandit is already configured to catch hardcoded credentials and if secrets scanning is already part of the repository-level pre-commit configuration.

The justification is proportionality of risk: every gate that can catch an authentication vulnerability before production is worth the latency cost. The only optional items are those that address concerns not specific to the security properties of authentication middleware.

---

*End of Chapter 8*
