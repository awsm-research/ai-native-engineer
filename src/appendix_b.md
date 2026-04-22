# Appendix B: Design Pattern Reference

This appendix provides a quick-reference summary of design patterns mentioned in the book, including patterns from Chapter 3 and additional patterns encountered in AI-native codebases. For each pattern, the reference includes the problem it solves, the Python implementation sketch, and common use cases.

For a comprehensive treatment, see Gamma et al. (1994), *Design Patterns: Elements of Reusable Object-Oriented Software* ([Amazon](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612)) and Fowler (2002), *Patterns of Enterprise Application Architecture* ([martinfowler.com](https://martinfowler.com/eaaCatalog/)).

---

## B.1 Creational Patterns

### Singleton

**Problem**: Ensure a class has only one instance and provide global access to it.

**When to use**: Database connection pools, configuration objects, logging instances. Use sparingly — Singletons introduce global state that makes testing harder.

```python
class Config:
    _instance: "Config | None" = None

    def __new__(cls) -> "Config":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._loaded = False
        return cls._instance

    def load(self, path: str) -> None:
        if not self._loaded:
            # load configuration from file
            self._loaded = True
```

**Preferred alternative in Python**: Module-level instances. A module is loaded once; a module-level variable is effectively a singleton:
```python
# config.py
import os
DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:///dev.db")
```

### Factory Method

**Problem**: Create objects without specifying the exact class, letting subclasses or configuration decide.

**When to use**: When the type of object to create depends on runtime information; when you want to decouple creation from use.

```python
from abc import ABC, abstractmethod


class Notifier(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> None: ...


class EmailNotifier(Notifier):
    def send(self, recipient: str, message: str) -> None:
        print(f"Sending email to {recipient}: {message}")


class SMSNotifier(Notifier):
    def send(self, recipient: str, message: str) -> None:
        print(f"Sending SMS to {recipient}: {message}")


def create_notifier(channel: str) -> Notifier:
    match channel:
        case "email":
            return EmailNotifier()
        case "sms":
            return SMSNotifier()
        case _:
            raise ValueError(f"Unknown channel: {channel}")
```

### Builder

**Problem**: Construct a complex object step by step, separating construction from representation.

**When to use**: Objects with many optional parameters; when the construction process must allow different representations.

```python
from dataclasses import dataclass, field


@dataclass
class Query:
    table: str
    filters: list[str] = field(default_factory=list)
    order_by: str | None = None
    limit: int | None = None

    def where(self, condition: str) -> "Query":
        return Query(self.table, self.filters + [condition], self.order_by, self.limit)

    def order(self, column: str) -> "Query":
        return Query(self.table, self.filters, column, self.limit)

    def take(self, n: int) -> "Query":
        return Query(self.table, self.filters, self.order_by, n)


# Usage (fluent builder pattern)
query = (
    Query("tasks")
    .where("status = 'open'")
    .where("priority >= 2")
    .order("due_date")
    .take(10)
)
```

---

## B.2 Structural Patterns

### Repository

**Problem**: Abstract the data access layer, presenting a collection-like interface to the domain model.

**When to use**: Any application with a persistence layer. Repository is one of the most important patterns for testable code — it allows tests to substitute in-memory storage for a real database.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import date


@dataclass
class Task:
    id: int
    title: str
    status: str
    due_date: date | None = None


class TaskRepository(ABC):
    @abstractmethod
    def get_by_id(self, task_id: int) -> Task | None: ...

    @abstractmethod
    def list_all(self) -> list[Task]: ...

    @abstractmethod
    def save(self, task: Task) -> Task: ...

    @abstractmethod
    def delete(self, task_id: int) -> None: ...


class InMemoryTaskRepository(TaskRepository):
    def __init__(self) -> None:
        self._store: dict[int, Task] = {}
        self._next_id = 1

    def get_by_id(self, task_id: int) -> Task | None:
        return self._store.get(task_id)

    def list_all(self) -> list[Task]:
        return list(self._store.values())

    def save(self, task: Task) -> Task:
        if task.id == 0:
            task = Task(self._next_id, task.title, task.status, task.due_date)
            self._next_id += 1
        self._store[task.id] = task
        return task

    def delete(self, task_id: int) -> None:
        self._store.pop(task_id, None)
```

### Adapter

**Problem**: Convert the interface of a class into an interface that clients expect. Allows classes with incompatible interfaces to work together.

**When to use**: Integrating third-party libraries; wrapping legacy APIs; abstracting over multiple AI provider SDKs.

```python
from abc import ABC, abstractmethod


class AIClient(ABC):
    @abstractmethod
    def complete(self, prompt: str, max_tokens: int = 1024) -> str: ...


class AnthropicAdapter(AIClient):
    def __init__(self) -> None:
        import anthropic
        self._client = anthropic.Anthropic()

    def complete(self, prompt: str, max_tokens: int = 1024) -> str:
        response = self._client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=max_tokens,
            messages=[{"role": "user", "content": prompt}],
        )
        return response.content[0].text


class OpenAIAdapter(AIClient):
    def __init__(self) -> None:
        from openai import OpenAI
        self._client = OpenAI()

    def complete(self, prompt: str, max_tokens: int = 1024) -> str:
        response = self._client.chat.completions.create(
            model="gpt-4o",
            max_tokens=max_tokens,
            messages=[{"role": "user", "content": prompt}],
        )
        return response.choices[0].message.content or ""
```

### Decorator

**Problem**: Attach additional behaviour to an object dynamically, without modifying its class.

**When to use**: Cross-cutting concerns (logging, caching, rate limiting, retry); extending third-party classes without subclassing.

```python
import functools
import time
from typing import Callable, TypeVar

F = TypeVar("F", bound=Callable)


def retry(max_attempts: int = 3, delay: float = 1.0):
    """Retry a function on exception."""
    def decorator(fn: F) -> F:
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return fn(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay * (attempt + 1))
        return wrapper  # type: ignore[return-value]
    return decorator


def cached(fn: F) -> F:
    """Simple in-memory cache."""
    cache: dict = {}
    @functools.wraps(fn)
    def wrapper(*args):
        if args not in cache:
            cache[args] = fn(*args)
        return cache[args]
    return wrapper  # type: ignore[return-value]
```

---

## B.3 Behavioural Patterns

### Observer

**Problem**: Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified.

**When to use**: Event systems, UI data binding, audit logging, webhook dispatch.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass


class EventHandler(ABC):
    @abstractmethod
    def handle(self, event: dict) -> None: ...


class EventBus:
    def __init__(self) -> None:
        self._handlers: dict[str, list[EventHandler]] = {}

    def subscribe(self, event_type: str, handler: EventHandler) -> None:
        self._handlers.setdefault(event_type, []).append(handler)

    def publish(self, event_type: str, payload: dict) -> None:
        for handler in self._handlers.get(event_type, []):
            handler.handle({"type": event_type, **payload})


class AuditLogger(EventHandler):
    def handle(self, event: dict) -> None:
        print(f"[AUDIT] {event}")


# Usage
bus = EventBus()
bus.subscribe("task.completed", AuditLogger())
bus.publish("task.completed", {"task_id": 42, "user_id": 7})
```

### Strategy

**Problem**: Define a family of algorithms, encapsulate each one, and make them interchangeable.

**When to use**: Sorting algorithms, pricing strategies, authentication methods, AI model selection.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import date


@dataclass
class Task:
    id: int
    title: str
    priority: int
    due_date: date | None


class SortStrategy(ABC):
    @abstractmethod
    def sort(self, tasks: list[Task]) -> list[Task]: ...


class ByPriority(SortStrategy):
    def sort(self, tasks: list[Task]) -> list[Task]:
        return sorted(tasks, key=lambda t: t.priority, reverse=True)


class ByDueDate(SortStrategy):
    def sort(self, tasks: list[Task]) -> list[Task]:
        return sorted(tasks, key=lambda t: (t.due_date is None, t.due_date))


class TaskList:
    def __init__(self, strategy: SortStrategy) -> None:
        self._strategy = strategy

    def set_strategy(self, strategy: SortStrategy) -> None:
        self._strategy = strategy

    def get_sorted(self, tasks: list[Task]) -> list[Task]:
        return self._strategy.sort(tasks)
```

### Command

**Problem**: Encapsulate a request as an object, allowing parameterisation, queuing, and undo operations.

**When to use**: Undo/redo functionality, task queues, audit trails, agent tool-use implementations.

```python
from abc import ABC, abstractmethod


class Command(ABC):
    @abstractmethod
    def execute(self) -> str: ...

    @abstractmethod
    def undo(self) -> str: ...


class CreateTaskCommand(Command):
    def __init__(self, repo, title: str, priority: int) -> None:
        self._repo = repo
        self._title = title
        self._priority = priority
        self._created_id: int | None = None

    def execute(self) -> str:
        task = self._repo.create(self._title, self._priority)
        self._created_id = task.id
        return f"Created task {task.id}"

    def undo(self) -> str:
        if self._created_id:
            self._repo.delete(self._created_id)
            return f"Deleted task {self._created_id}"
        return "Nothing to undo"


class CommandHistory:
    def __init__(self) -> None:
        self._history: list[Command] = []

    def execute(self, command: Command) -> str:
        result = command.execute()
        self._history.append(command)
        return result

    def undo(self) -> str:
        if not self._history:
            return "Nothing to undo"
        return self._history.pop().undo()
```

---

## B.4 AI-Specific Patterns

These patterns emerge specifically in AI-native systems and are not in the original GoF catalog.

### Prompt Template

**Problem**: Construct prompts programmatically from variables while maintaining readability and testability.

```python
from string import Template


REVIEW_TEMPLATE = Template("""
You are a senior software engineer reviewing a pull request.

## Context
Project: $project_name
Language: Python 3.11
Style guide: PEP 8 with type hints

## Code to review
$code

## Review focus
$focus_areas

Provide specific, actionable feedback. For each issue, provide:
- The line or section with the problem
- Why it is a problem
- A concrete suggestion for improvement
""")


def create_review_prompt(
    project_name: str, code: str, focus_areas: str
) -> str:
    return REVIEW_TEMPLATE.substitute(
        project_name=project_name,
        code=code,
        focus_areas=focus_areas,
    )
```

### ReAct Agent Loop

**Problem**: Enable an AI model to interleave reasoning and tool use in an iterative loop until a task is complete.

See Chapter 8 for the full implementation. The core pattern:

```python
def agent_loop(task: str, tools: dict, max_steps: int = 10) -> str:
    messages = [{"role": "user", "content": task}]

    for _ in range(max_steps):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=list(tools.values()),
            messages=messages,
        )

        if response.stop_reason == "end_turn":
            return extract_text(response)

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})
            tool_results = execute_tools(response.content, tools)
            messages.append({"role": "user", "content": tool_results})

    return "Max steps reached without completing task"
```

### Evaluator-Generator

**Problem**: Use a separate AI call to evaluate the output of a generation call, rather than trusting the generator to self-evaluate.

```python
def generate_and_evaluate(specification: str, code: str) -> dict:
    """Generate code, then evaluate it independently."""
    eval_response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"""Evaluate whether this code correctly implements the specification.

Specification:
{specification}

Code:
{code}

For each requirement in the specification, state: PASS or FAIL with brief justification.
End with: OVERALL: PASS or OVERALL: FAIL"""
        }],
    )
    result = eval_response.content[0].text
    return {
        "verdict": "PASS" if "OVERALL: PASS" in result else "FAIL",
        "details": result,
    }
```
