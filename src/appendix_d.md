# Appendix D: Prompt Pattern Reference

This appendix collects the prompt patterns introduced throughout the book into a single reference. Each entry includes the pattern name, the problem it solves, the template, and a worked example from the course project.

For a systematic treatment of prompt patterns, see [White et al. (2023)](https://arxiv.org/abs/2302.11382), "A Prompt Pattern Catalog to Enhance Prompt Engineering with ChatGPT."

---

## D.1 Specification Patterns

These patterns structure the *specification* you write before asking an AI to generate code.

### D.1.1 Function Contract Pattern

**Problem**: Vague function descriptions produce plausible-but-wrong implementations.

**Template**:
```
Implement a Python function with the following specification:

Function signature:
  <function_name>(<params>) -> <return_type>

Context:
  <describe the domain, data structures, available imports>

Behaviour:
  - <rule 1>
  - <rule 2>
  ...

Constraints:
  - MUST <hard requirement>
  - MUST NOT <prohibited behaviour>
  - <edge case handling>

Examples:
  <input> -> <expected output>  # <brief explanation>
  <input> -> <expected output>  # edge case
  <input> -> <expected output>  # error case

Return only the Python function with type hints. No explanation.
```

**Course project example** (Chapter 6):
```
Implement a Python function with the following specification:

Function signature:
  get_overdue_tasks(tasks: list[Task], reference_date: date | None = None) -> list[Task]

Context:
  Task is a dataclass with fields: id (int), title (str), status (str),
  due_date (date | None), priority (int, 1=low 3=high).
  Import: from datetime import date

Behaviour:
  - Return tasks where due_date < reference_date AND status != "done"
  - If reference_date is None, use date.today()
  - Sort result by due_date ascending; break ties by priority descending

Constraints:
  - MUST NOT include tasks with status "done" even if past due
  - MUST NOT include tasks with due_date = None
  - MUST return [] (not raise) if tasks is empty

Examples:
  [Task(1,"Fix bug","open",date(2024,1,1),2)] with ref date(2024,3,1) -> [Task(1,...)]
  [Task(2,"Deploy","done",date(2024,1,1),2)] with ref date(2024,3,1) -> []
  [] -> []

Return only the Python function with type hints. No explanation.
```

---

### D.1.2 Class Design Pattern

**Problem**: Generating a class without context produces structurally incorrect or inconsistent designs.

**Template**:
```
Design a Python class with the following specification:

Class name: <ClassName>
Purpose: <one-sentence description>

Context:
  <related classes, domain model, constraints>

Public interface:
  __init__(<params>): <description>
  <method_name>(<params>) -> <return>: <description>
  ...

Invariants (always true after any public method):
  - <invariant 1>
  - <invariant 2>

Do not implement private helpers unless necessary for correctness.
Include type hints throughout.
```

---

### D.1.3 Refactoring Specification Pattern

**Problem**: "Refactor this code" without constraints produces unnecessary rewrites.

**Template**:
```
Refactor the following code. Apply only the changes described below.
Do not change behaviour, add features, or restructure code not mentioned.

Original code:
<paste code>

Required changes:
1. <specific change 1>
2. <specific change 2>

Do NOT:
- Change function signatures
- Add new functionality
- Rename variables not mentioned above
```

---

## D.2 Role Patterns

### D.2.1 Expert Role Pattern

**Problem**: Generic prompts produce generic responses. Assigning a role primes domain-specific knowledge.

**Template**:
```
You are a <role> with expertise in <domain>.

<task description>
```

**Worked examples**:

```python
# Security review
system = "You are a senior application security engineer specialising in Python " \
         "web application vulnerabilities. You are familiar with the OWASP Top 10 " \
         "and CWE/CVE classification systems."

# Architecture review
system = "You are a principal software architect with 15 years of experience " \
         "designing distributed systems. You prioritise operational simplicity " \
         "over theoretical elegance."

# Code review
system = "You are a senior Python engineer performing a pull request review. " \
         "Focus on correctness, type safety, and test coverage. " \
         "Be specific: cite line numbers, provide corrected code snippets."
```

**When to use**: Tasks where the quality of output depends on domain expertise — security review, architecture decisions, code review, requirements critique.

---

### D.2.2 Persona-as-Critic Pattern

**Problem**: AI tends to agree with the framing you provide. Assigning an adversarial persona surfaces weaknesses.

**Template**:
```
You are a <adversarial role>. Your job is to find flaws in the following <artefact>.

<artefact>

Identify every problem you can find. Do not suggest improvements unless asked.
Rate the severity of each issue: Critical / High / Medium / Low.
```

**Worked example**:
```
You are a malicious attacker looking for security vulnerabilities.
Your job is to find flaws in the following Python function.

<code>

Identify every exploitable vulnerability. Rate severity: Critical/High/Medium/Low.
Do not suggest fixes — only identify problems.
```

---

## D.3 Reasoning Patterns

### D.3.1 Chain-of-Thought Pattern

**Problem**: Complex implementation tasks produce errors when the model generates code without planning.

**Template**:
```
Before writing any code, think through:
1. What are the edge cases I need to handle?
2. What invariants must hold throughout?
3. What is the simplest correct algorithm?
4. What could go wrong in my implementation?

Then write the implementation.
```

**When to use**: Non-trivial algorithms, functions with multiple interacting constraints, debugging complex failures.

---

### D.3.2 Structured Decomposition Pattern

**Problem**: Large tasks produce poor outputs when attempted in a single prompt.

**Template**:
```
Break this task into subtasks, then implement each one:

Task: <description>

Step 1: List the subtasks required.
Step 2: For each subtask, describe the interface it needs to expose.
Step 3: Implement each subtask in order.
Step 4: Integrate the subtasks into the complete solution.
```

**When to use**: Features spanning multiple functions or classes; tasks with complex state management; agentic planning.

---

### D.3.3 Self-Consistency Pattern

**Problem**: A single generation may be confidently wrong. Multiple independent generations reveal uncertainty.

**Template**:
```python
# Generate the same function three times and compare
def generate_multiple(spec: str, n: int = 3) -> list[str]:
    results = []
    for _ in range(n):
        response = client.messages.create(
            model="claude-opus-4-7",
            max_tokens=1024,
            messages=[{"role": "user", "content": spec}],
        )
        results.append(response.content[0].text)
    return results

# If all three agree in structure → higher confidence
# If they diverge → specification is likely under-constrained
```

**When to use**: Security-critical functions; functions with subtle correctness requirements; when you cannot write comprehensive tests before generation.

---

## D.4 Evaluation Patterns

### D.4.1 Independent Evaluator Pattern

**Problem**: The model that generates code is biased toward finding it correct. An independent evaluation call is more reliable.

**Template**:
```
You are evaluating whether the following code correctly implements its specification.

Specification:
<specification>

Code:
<code>

For each requirement in the specification, state:
  PASS — the code correctly implements this requirement
  FAIL — the code does not implement this requirement (explain why)

End with: OVERALL: PASS or OVERALL: FAIL
```

**Python implementation**:
```python
import anthropic

client = anthropic.Anthropic()


def evaluate_against_spec(specification: str, code: str) -> dict:
    response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"""Evaluate whether this code correctly implements its specification.

Specification:
{specification}

Code:
{code}

For each requirement, state PASS or FAIL with brief justification.
End with: OVERALL: PASS or OVERALL: FAIL"""
        }],
    )
    text = response.content[0].text
    return {
        "verdict": "PASS" if "OVERALL: PASS" in text else "FAIL",
        "details": text,
    }
```

---

### D.4.2 Security Review Pattern

**Problem**: Standard code review misses security issues that require adversarial thinking.

**Template**:
```
You are a security engineer performing a security review of the following Python code.

Check for:
- Injection vulnerabilities (SQL, command, path traversal)
- Authentication and authorisation flaws
- Insecure data handling (hardcoded credentials, logging PII)
- Insecure cryptography (MD5/SHA1 for passwords, predictable tokens)
- Input validation gaps
- Dependency vulnerabilities (note any suspicious imports)

For each issue:
1. Name the vulnerability (use CWE identifier if applicable)
2. Quote the specific line(s) of code
3. Explain the risk
4. Provide a corrected version

Code:
<code>
```

---

### D.4.3 Test Generation Pattern

**Problem**: Human-written tests tend to test the happy path. AI can systematically generate edge case tests.

**Template**:
```
Generate pytest tests for the following Python function.

Function specification:
<specification>

Function implementation:
<code>

Write tests that cover:
1. The happy path (standard inputs producing expected outputs)
2. Boundary values (empty collections, zero, None, maximum values)
3. Invalid inputs (wrong types, out-of-range values)
4. Concurrency concerns if applicable

Each test should have a descriptive name explaining what it tests.
Use pytest fixtures where appropriate.
Do not test implementation details — test observable behaviour only.
```

---

## D.5 Context Management Patterns

### D.5.1 Minimal Context Pattern

**Problem**: Including too much context dilutes the model's focus and increases cost.

**Rule**: Include only what the model needs to generate the correct output. Test by asking: "Could the model produce a wrong output if I omitted this?"

**What to include**:
- Data structures / type definitions the function operates on
- Functions the implementation should call (not all functions in the file)
- Specific constraints from related code (e.g., "the database uses UTC timestamps")
- The function signature and return type

**What to omit**:
- Unrelated functions in the same file
- Import statements unless they introduce constraints
- Comments explaining unrelated parts of the codebase
- Historical context ("we changed this in version 2.0 because...")

---

### D.5.2 Context Compression Pattern

**Problem**: Long files contain more information than the context window can hold effectively.

**Template**:
```
Here is a summary of the codebase relevant to this task:

Data models:
  Task: id(int), title(str), status(str), due_date(date|None), priority(int 1-3)
  User: id(int), email(str), role(str)

Available utilities:
  get_db() -> Session   # SQLAlchemy session
  send_email(to, subject, body) -> None

Task service (task_service.py):
  - create_task(title, priority, assignee_id) -> Task
  - get_overdue_tasks(reference_date=None) -> list[Task]  ← implement this

Implement get_overdue_tasks with the following specification:
<specification>
```

---

## D.6 Pattern Selection Guide

| Situation | Recommended pattern(s) |
|---|---|
| Implementing a function for the first time | Function Contract + Chain-of-Thought |
| Security-critical code | Function Contract + Security Review |
| Complex algorithm | Chain-of-Thought + Self-Consistency |
| Evaluating existing AI-generated code | Independent Evaluator |
| Generating tests | Test Generation |
| Refactoring existing code | Refactoring Specification |
| Design / architecture decision | Expert Role + Structured Decomposition |
| Finding weaknesses in your own design | Persona-as-Critic |
| Long file, limited context | Context Compression |
