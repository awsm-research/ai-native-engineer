# Chapter 3: Software Design, Architecture, and Patterns

> *"A designer knows he has achieved perfection not when there is nothing left to add, but when there is nothing left to take away."*
> — Antoine de Saint-Exupéry

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Read and produce UML diagrams: use case, class, sequence, and component diagrams.
2. Compare and select appropriate architectural patterns for a given system.
3. Identify and apply common Gang of Four design patterns.
4. Apply SOLID principles and other design guidelines to produce maintainable code.
5. Write clean, readable Python code following established conventions.
6. Use AI tools to assist with design and scaffolding — and critically evaluate what they produce.

---

## 3.1 Why Design Matters

Writing code that works is necessary but not sufficient. Code must also be maintainable — readable and modifiable by other developers (and by your future self) over months and years. Poor design decisions made early in a project compound over time: a monolithic module that is difficult to test becomes more difficult to test as it grows; a tangled dependency structure becomes harder to untangle as more code depends on it.

Software design is the activity of deciding *how* a system will be structured before (or alongside) the activity of writing code. Good design:

- Makes the system easier to understand
- Makes the system easier to test
- Makes the system easier to change in response to new requirements
- Reduces the risk of introducing bugs when modifying existing functionality

This chapter covers design at three levels:
1. **Diagrams**: visual representations of system structure and behaviour
2. **Architecture**: high-level decisions about system organisation
3. **Patterns**: proven solutions to recurring design problems

---

## 3.2 UML Diagrams

The Unified Modeling Language (UML) is a standardised notation for visualising software systems ([OMG, 2017](https://www.omg.org/spec/UML/2.5.1/)). It provides a shared vocabulary for communicating design decisions between developers, architects, and stakeholders.

We focus on four diagram types that are most commonly used in practice.

### 3.2.1 Use Case Diagrams

Use case diagrams show the interactions between *actors* (users or external systems) and the *use cases* (features) a system provides. They communicate system scope at a high level and are useful for stakeholder communication early in a project.

**Elements:**
- **Actor**: A stick figure representing a user role or external system
- **Use case**: An oval representing a system function
- **Association**: A line connecting an actor to the use cases they participate in
- **System boundary**: A rectangle enclosing all use cases in scope

**Example — Task Management System:**

```
┌─────────────────────────────────────────────────────┐
│                 Task Management System               │
│                                                      │
│   (Create Task)    (Assign Task)    (Close Task)     │
│                                                      │
│   (View Dashboard)  (Generate Report)                │
│                                                      │
│   (Receive Notification)                             │
└─────────────────────────────────────────────────────┘
        │               │                │
      User          Manager          Email Service
```

Use case diagrams intentionally omit implementation detail — they show *what* the system does, not *how*.

### 3.2.2 Class Diagrams

Class diagrams show the static structure of a system — the classes, their attributes and methods, and the relationships between them. They are the most widely used UML diagram type for communicating object-oriented design.

**Key relationships:**
- **Association**: A uses B (solid line)
- **Aggregation**: A has B, B can exist without A (hollow diamond)
- **Composition**: A contains B, B cannot exist without A (filled diamond)
- **Inheritance**: A is a B (hollow triangle arrow)
- **Interface implementation**: A implements B (dashed line with hollow triangle)
- **Dependency**: A depends on B (dashed arrow)

**Example — Task Management Domain Model:**

```
┌────────────────┐         ┌────────────────┐
│    Project     │1      * │     Task       │
│────────────────│─────────│────────────────│
│ id: UUID       │         │ id: UUID       │
│ name: str      │         │ title: str     │
│ owner: User    │         │ description:str│
│────────────────│         │ due_date: date │
│ add_task()     │         │ priority: Enum │
│ get_tasks()    │         │ status: Enum   │
└────────────────┘         │────────────────│
                           │ assign(user)   │
                           │ complete()     │
                           └───────┬────────┘
                                   │* assignees
                           ┌───────┴────────┐
                           │      User      │
                           │────────────────│
                           │ id: UUID       │
                           │ email: str     │
                           │ role: Enum     │
                           └────────────────┘
```

### 3.2.3 Sequence Diagrams

Sequence diagrams show how objects interact over time to accomplish a specific use case. They are valuable for documenting the flow of a complex operation, particularly when multiple components or services are involved.

**Example — Assigning a task:**

```
Client       API Gateway    TaskService    UserService    NotificationService
  │               │               │               │               │
  │  POST /assign │               │               │               │
  │──────────────>│               │               │               │
  │               │ assign(id,email)              │               │
  │               │──────────────>│               │               │
  │               │               │ getUser(email)│               │
  │               │               │──────────────>│               │
  │               │               │   user        │               │
  │               │               │<──────────────│               │
  │               │               │                   notify(user)│
  │               │               │──────────────────────────────>│
  │               │               │                   email sent  │
  │               │               │<──────────────────────────────│
  │               │  200 OK       │               │               │
  │<──────────────│               │               │               │
```

### 3.2.4 Component Diagrams

Component diagrams show the high-level organisation of a system into components and their dependencies. They bridge the gap between architecture diagrams and class diagrams.

**Example — Task Management API components:**

```
┌──────────────────────────────────────────────────────────┐
│                     Task Management API                   │
│                                                          │
│  ┌─────────────┐    ┌──────────────┐   ┌─────────────┐  │
│  │  REST API   │───>│   Service    │──>│  Repository │  │
│  │  (FastAPI)  │    │   Layer      │   │  Layer      │  │
│  └─────────────┘    └──────────────┘   └──────┬──────┘  │
│                                               │         │
│  ┌─────────────┐                      ┌───────┴──────┐  │
│  │   Auth      │                      │  PostgreSQL  │  │
│  │  (JWT)      │                      │  Database    │  │
│  └─────────────┘                      └─────────────┘  │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │            Notification Service                  │    │
│  │         (Email via SendGrid)                     │    │
│  └─────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

## 3.3 Architectural Patterns

An architectural pattern is a high-level strategy for organising the major components of a system. Selecting the right architectural pattern for a system's requirements is one of the most consequential decisions a software team makes — and one of the hardest to reverse.

### 3.3.1 Layered (N-Tier) Architecture

The layered pattern organises a system into horizontal layers, where each layer serves the layer above it and depends only on the layer below it ([Buschmann et al., 1996](https://www.wiley.com/en-us/Pattern+Oriented+Software+Architecture%2C+Volume+1%2C+A+System+of+Patterns-p-9780471958697)).

```
┌─────────────────────────────┐
│       Presentation Layer    │  (HTTP endpoints, request/response)
├─────────────────────────────┤
│       Business Logic Layer  │  (Services, domain logic, rules)
├─────────────────────────────┤
│       Data Access Layer     │  (Repositories, ORM, queries)
├─────────────────────────────┤
│       Database Layer        │  (PostgreSQL, Redis, etc.)
└─────────────────────────────┘
```

**Strengths:** Simple to understand; good separation of concerns; easy to test each layer independently.

**Weaknesses:** Can lead to "pass-through" layers that add no logic; performance overhead from passing data through many layers; tendency toward monolithic deployment.

**Suitable for:** Business applications, CRUD-heavy APIs, systems where the team is primarily familiar with this pattern.

### 3.3.2 Model-View-Controller (MVC)

MVC separates a system into three components ([Reenskaug, 1979](https://folk.universitetetioslo.no/trygver/themes/mvc/mvc-index.html)):

- **Model**: The data and business logic
- **View**: The presentation layer (what the user sees)
- **Controller**: Handles user input and coordinates Model and View

MVC is widely used in web frameworks: Django, Ruby on Rails, and Spring MVC all implement variants of this pattern.

### 3.3.3 Event-Driven Architecture

In an event-driven architecture, components communicate by producing and consuming events rather than calling each other directly. An *event broker* (such as Apache Kafka or RabbitMQ) decouples producers from consumers.

```
Producer ──> [Event Broker] ──> Consumer A
                            ──> Consumer B
                            ──> Consumer C
```

**Strengths:** High decoupling; components can scale independently; easy to add new consumers without modifying producers.

**Weaknesses:** Harder to reason about system state; distributed tracing is complex; eventual consistency requires careful handling.

**Suitable for:** High-throughput systems, microservices that need loose coupling, real-time notification systems, audit log pipelines.

### 3.3.4 Microservices

A microservices architecture decomposes a system into small, independently deployable services, each responsible for a single bounded domain ([Newman, 2015](https://www.oreilly.com/library/view/building-microservices/9781491950340/)). Each service has its own database and communicates with others via APIs or events.

**Strengths:** Services can be deployed, scaled, and rewritten independently; teams can work autonomously on separate services; fault isolation.

**Weaknesses:** Significant operational complexity (service discovery, distributed tracing, network latency, eventual consistency); not appropriate for small teams or early-stage products.

**Suitable for:** Large teams (multiple squads, each owning a service); systems where different components have very different scaling requirements.

### 3.3.5 Monolithic Architecture

A monolith is a single deployable unit containing all the system's functionality. Despite its reputation, a well-structured monolith is often the right choice for small teams and early-stage systems ([Fowler, 2015](https://martinfowler.com/bliki/MonolithFirst.html)).

**Strengths:** Simple to develop, test, and deploy; no network latency between components; easy to refactor across the codebase.

**Weaknesses:** Entire system must be redeployed for any change; scaling requires scaling the entire application; risk of components becoming tightly coupled over time.

**The "Monolith First" principle**: Start with a well-structured monolith. Extract services only when you have clear evidence that a specific component needs independent scaling or when team boundaries demand it.

---

## 3.4 Design Patterns (Gang of Four)

Design patterns are proven, reusable solutions to commonly occurring problems in software design ([Gamma et al., 1994](https://en.wikipedia.org/wiki/Design_Patterns)). The original catalog, published by the "Gang of Four" (GoF), describes 23 patterns in three categories:

- **Creational**: How objects are created
- **Structural**: How objects are composed
- **Behavioural**: How objects interact and distribute responsibility

We cover the patterns most commonly encountered in Python backend development.

### 3.4.1 Singleton (Creational)

Ensures a class has only one instance and provides a global access point to it.

**Use case**: Database connection pools, configuration objects, logging instances.

```python
# singleton.py
class DatabaseConnection:
    _instance: "DatabaseConnection | None" = None

    def __new__(cls) -> "DatabaseConnection":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._connect()
        return cls._instance

    def _connect(self) -> None:
        # Initialise the connection once
        self.connection = "connected"  # placeholder

    def query(self, sql: str) -> list:
        # Execute query using self.connection
        return []


# Both variables point to the same instance
db1 = DatabaseConnection()
db2 = DatabaseConnection()
assert db1 is db2  # True
```

**Caution**: Singletons introduce global state, which can make testing difficult. In Python, dependency injection (passing the instance explicitly) is often preferable.

### 3.4.2 Factory Method (Creational)

Defines an interface for creating objects but lets subclasses decide which class to instantiate.

**Use case**: Creating notification objects (email, SMS, push) based on user preference.

```python
# factory.py
from abc import ABC, abstractmethod


class Notification(ABC):
    @abstractmethod
    def send(self, message: str, recipient: str) -> None: ...


class EmailNotification(Notification):
    def send(self, message: str, recipient: str) -> None:
        print(f"Sending email to {recipient}: {message}")


class SMSNotification(Notification):
    def send(self, message: str, recipient: str) -> None:
        print(f"Sending SMS to {recipient}: {message}")


def create_notification(channel: str) -> Notification:
    """Factory function — returns the appropriate Notification subclass."""
    channels: dict[str, type[Notification]] = {
        "email": EmailNotification,
        "sms": SMSNotification,
    }
    if channel not in channels:
        raise ValueError(f"Unknown notification channel: {channel}")
    return channels[channel]()


# Usage
notifier = create_notification("email")
notifier.send("Your task has been assigned.", "alice@example.com")
```

### 3.4.3 Observer (Behavioural)

Defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified automatically.

**Use case**: Event systems, UI data binding, notification pipelines.

```python
# observer.py
from abc import ABC, abstractmethod


class EventListener(ABC):
    @abstractmethod
    def on_event(self, event: dict) -> None: ...


class TaskEventBus:
    def __init__(self) -> None:
        self._listeners: list[EventListener] = []

    def subscribe(self, listener: EventListener) -> None:
        self._listeners.append(listener)

    def publish(self, event: dict) -> None:
        for listener in self._listeners:
            listener.on_event(event)


class EmailNotifier(EventListener):
    def on_event(self, event: dict) -> None:
        if event.get("type") == "task_assigned":
            print(f"Email: task {event['task_id']} assigned to {event['assignee']}")


class AuditLogger(EventListener):
    def on_event(self, event: dict) -> None:
        print(f"Audit log: {event}")


# Usage
bus = TaskEventBus()
bus.subscribe(EmailNotifier())
bus.subscribe(AuditLogger())

bus.publish({"type": "task_assigned", "task_id": "123", "assignee": "alice"})
```

### 3.4.4 Strategy (Behavioural)

Defines a family of algorithms, encapsulates each one, and makes them interchangeable.

**Use case**: Sorting algorithms, payment processing, priority calculation.

```python
# strategy.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import date


@dataclass
class Task:
    id: str
    title: str
    due_date: date
    priority: int  # 1 (low) to 4 (critical)


class SortStrategy(ABC):
    @abstractmethod
    def sort(self, tasks: list[Task]) -> list[Task]: ...


class SortByDueDate(SortStrategy):
    def sort(self, tasks: list[Task]) -> list[Task]:
        return sorted(tasks, key=lambda t: t.due_date)


class SortByPriority(SortStrategy):
    def sort(self, tasks: list[Task]) -> list[Task]:
        return sorted(tasks, key=lambda t: t.priority, reverse=True)


class TaskList:
    def __init__(self, strategy: SortStrategy) -> None:
        self._strategy = strategy

    def set_strategy(self, strategy: SortStrategy) -> None:
        self._strategy = strategy

    def get_sorted(self, tasks: list[Task]) -> list[Task]:
        return self._strategy.sort(tasks)
```

### 3.4.5 Repository (Architectural Pattern)

While not in the original GoF catalog, the Repository pattern ([Fowler, 2002](https://martinfowler.com/eaaCatalog/repository.html)) is essential in modern backend development. It abstracts the data access layer, presenting a collection-like interface to the domain model.

```python
# repository.py
from abc import ABC, abstractmethod
from uuid import UUID
from dataclasses import dataclass
from datetime import date


@dataclass
class Task:
    id: UUID
    title: str
    due_date: date | None = None


class TaskRepository(ABC):
    """Abstract repository — defines the interface."""

    @abstractmethod
    def find_by_id(self, task_id: UUID) -> Task | None: ...

    @abstractmethod
    def find_all_by_project(self, project_id: UUID) -> list[Task]: ...

    @abstractmethod
    def save(self, task: Task) -> Task: ...

    @abstractmethod
    def delete(self, task_id: UUID) -> None: ...


class InMemoryTaskRepository(TaskRepository):
    """In-memory implementation — used in tests."""

    def __init__(self) -> None:
        self._store: dict[UUID, Task] = {}

    def find_by_id(self, task_id: UUID) -> Task | None:
        return self._store.get(task_id)

    def find_all_by_project(self, project_id: UUID) -> list[Task]:
        return list(self._store.values())  # simplified

    def save(self, task: Task) -> Task:
        self._store[task.id] = task
        return task

    def delete(self, task_id: UUID) -> None:
        self._store.pop(task_id, None)
```

The key benefit: services depend on the abstract `TaskRepository`, not on a specific database implementation. Swapping PostgreSQL for SQLite in tests requires only a different concrete class.

---

## 3.5 Design Principles

Design patterns tell you *what* to do in specific situations. Design principles tell you *how to think* about design in general. These principles have been distilled from decades of practical experience.

### 3.5.1 SOLID Principles

The SOLID principles ([Martin, 2000](https://web.archive.org/web/20150906155800/http://www.objectmentor.com/resources/articles/Principles_and_Patterns.pdf)) are five guidelines for writing maintainable object-oriented code:

**S — Single Responsibility Principle (SRP)**

> A class should have only one reason to change.

A class that handles HTTP parsing, business logic, *and* database queries will need to change whenever any of those three concerns changes. Separating them into different classes means each has one reason to change.

```python
# Violates SRP — this class does too much
class TaskService:
    def create_task(self, title: str, user_id: str) -> dict:
        # Business logic
        if not title.strip():
            raise ValueError("Title cannot be empty")
        # Database access (should be in repository)
        db.execute("INSERT INTO tasks ...")
        # Email sending (should be in notification service)
        smtp.send_email(user_id, "Task created")
        return {"id": "...", "title": title}
```

**O — Open/Closed Principle (OCP)**

> Software entities should be open for extension, but closed for modification.

You should be able to add new behaviour without modifying existing code. The Strategy pattern from Section 3.4.4 is a direct application of OCP: new sort strategies can be added without modifying `TaskList`.

**L — Liskov Substitution Principle (LSP)**

> Objects of a subclass should be substitutable for objects of the superclass without altering program correctness.

If `InMemoryTaskRepository` is a subclass of `TaskRepository`, any code that works with `TaskRepository` must work identically with `InMemoryTaskRepository`. Violating LSP typically indicates that the inheritance relationship is wrong.

**I — Interface Segregation Principle (ISP)**

> Clients should not be forced to depend on interfaces they do not use.

Rather than one large interface, prefer several small, focused ones. A `ReadOnlyTaskRepository` interface (with only `find_by_id` and `find_all`) is more appropriate for a reporting service than a full `TaskRepository` that includes `save` and `delete`.

**D — Dependency Inversion Principle (DIP)**

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

```python
# Violates DIP — TaskService depends directly on the concrete PostgreSQL implementation
class TaskService:
    def __init__(self) -> None:
        self.repo = PostgresTaskRepository()  # concrete dependency

# Follows DIP — TaskService depends on the abstract interface
class TaskService:
    def __init__(self, repo: TaskRepository) -> None:
        self.repo = repo  # injected abstraction
```

This is *dependency injection* — the concrete implementation is passed in from outside, typically by an application container. It makes `TaskService` testable with `InMemoryTaskRepository`.

### 3.5.2 DRY: Don't Repeat Yourself

> Every piece of knowledge must have a single, unambiguous, authoritative representation within a system. ([Hunt & Thomas, 1999](https://pragprog.com/titles/tpp/the-pragmatic-programmer/))

Duplicated code is duplicated knowledge. When the logic changes (and it will), you must find and update every copy. The solution is not always to extract a function — sometimes the duplication is accidental and the two pieces of code will diverge. Use judgment: extract when the duplication represents the *same concept*, not just the same syntax.

### 3.5.3 Composition Over Inheritance

Prefer composing objects from smaller, focused components over building deep inheritance hierarchies. Inheritance creates tight coupling between parent and child; composition allows components to be mixed and matched.

### 3.5.4 Hollywood Principle

> "Don't call us, we'll call you."

High-level components should control when and how low-level components are used, not the reverse. This is the principle behind inversion of control (IoC) frameworks and the Observer pattern.

---

## 3.6 Clean Code

Clean code is code that is easy to read, understand, and modify ([Martin, 2008](https://www.oreilly.com/library/view/clean-code-a/9780136083238/)). It is not about aesthetics — it is about reducing the cognitive load on the next developer who reads it (who is often you, six months later).

### 3.6.1 Naming

Names should reveal intent. Avoid abbreviations, single-letter variables (except in well-established contexts like loop counters), and misleading names.

```python
# Poor naming
def proc(d: list, f: bool) -> list:
    r = []
    for i in d:
        if i["s"] == 1 or f:
            r.append(i)
    return r

# Clean naming
def get_active_tasks(tasks: list[dict], include_archived: bool = False) -> list[dict]:
    return [
        task for task in tasks
        if task["status"] == 1 or include_archived
    ]
```

### 3.6.2 Functions

Functions should do one thing and do it well. A function that can be described with "and" in its name (e.g., `validate_and_save_task`) is doing too much. Keep functions short — typically 5–20 lines. If a function is longer, it is probably doing more than one thing.

### 3.6.3 Comments

Write code that does not need comments. When a comment is necessary, explain *why*, not *what* — the code already shows what it does.

```python
# Poor comment — explains what the code does, which is obvious
# Loop through tasks and add them to the result list
result = [task for task in tasks if task.is_active()]

# Good comment — explains a non-obvious constraint
# Skip soft-deleted tasks: the UI shows these with a strikethrough
# but the API should not return them in list endpoints
result = [task for task in tasks if not task.deleted_at]
```

### 3.6.4 Code Structure and Style

Consistent structure and formatting reduce cognitive load. For Python, follow [PEP 8](https://peps.python.org/pep-0008/) — the official style guide — and use `ruff` (introduced in Chapter 1) to enforce it automatically.

Key conventions:
- 4-space indentation
- Maximum line length: 88–120 characters (team decision)
- Two blank lines between top-level definitions
- Type annotations on all function signatures (enforced by `mypy`)

---

## 3.7 AI-Assisted Design

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

