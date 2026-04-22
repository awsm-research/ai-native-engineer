# Chapter 1: Software Engineering in the Age of AI

> *"The tools we use have a profound influence on our thinking habits, and therefore on our thinking abilities."*
> — Edsger W. Dijkstra

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Describe the historical evolution of software engineering from its origins to the present day.
2. Explain the key software development lifecycle (SDLC) models: Waterfall, Agile, Scrum, and Kanban.
3. Apply the MoSCoW framework to prioritise requirements and manage scope.
4. Write basic user stories with story point estimates.
5. Articulate how AI is reshaping each phase of the SDLC and what this means for the role of the software engineer.

---

## 1.1 What Is Software Engineering?

Software engineering is the disciplined application of engineering principles to the design, development, testing, and maintenance of software systems. Unlike informal programming, software engineering emphasises process, quality, collaboration, and long-term maintainability.

The term was deliberately chosen. In 1968, NATO convened a conference in Garmisch, Germany, to address what organisers called the "software crisis" — a widespread recognition that software projects were routinely over budget, delivered late, and unreliable ([Naur & Randell, 1969](http://homepages.cs.ncl.ac.uk/brian.randell/NATO/nato1968.PDF)). The goal of using the word *engineering* was aspirational: to bring to software the same rigour, predictability, and professionalism that civil or mechanical engineers brought to bridges and engines.

That aspiration has guided the field ever since — and it remains relevant today, even as the tools, languages, and collaborators (including AI systems) have changed dramatically.

### Why Software Engineering Matters

Consider two scenarios:

- **Scenario A**: A solo developer writes a script to process CSV files for personal use. It works, mostly. When it breaks, they fix it themselves.
- **Scenario B**: A team of 50 engineers builds a financial trading platform used by millions of customers. Bugs can cause financial losses. Downtime can trigger regulatory penalties.

Software engineering is primarily concerned with Scenario B — or with preparing developers who will eventually work on systems of that scale and consequence. The principles covered in this book apply whether you are building a mobile app, a machine learning pipeline, or an AI-assisted development tool.

---

## 1.2 A Brief History of Software Engineering

Understanding where software engineering came from helps explain why its practices exist and why they are changing again now.

### 1.2.1 The Early Years (1940s–1960s)

The first programmers wrote machine code directly — sequences of binary instructions hand-crafted for specific hardware. Programming was considered a clerical task; the real intellectual work was thought to be mathematics and system design.

As software grew more complex through the 1950s, assembly languages and early high-level languages like FORTRAN (1957) and COBOL (1959) emerged. Programs grew from hundreds of lines to hundreds of thousands. Managing this complexity became a serious problem.

### 1.2.2 The Software Crisis and Structured Programming (1968–1980s)

The 1968 NATO conference crystallised the software crisis. Projects like the IBM OS/360 operating system — documented famously by Fred Brooks in *The Mythical Man-Month* ([Brooks, 1975](https://en.wikipedia.org/wiki/The_Mythical_Man-Month)) — demonstrated that adding more programmers to a late project often made it later. Software complexity was not a resource problem; it was a conceptual one.

The response was *structured programming*, championed by Dijkstra, Hoare, and Wirth. Programs should be built from clear, hierarchical control structures — sequences, selections, and iterations — rather than the chaos of `GOTO` statements. This was the beginning of thinking about software as something that could be reasoned about formally.

### 1.2.3 Object-Oriented Programming and Software Patterns (1980s–1990s)

The 1980s and 1990s saw the rise of object-oriented programming (OOP) — a paradigm in which software is modelled as interacting objects with state and behaviour. Languages like C++, Smalltalk, and later Java brought OOP to mainstream development.

In 1994, the "Gang of Four" — Gamma, Helm, Johnson, and Vlissides — published *Design Patterns: Elements of Reusable Object-Oriented Software* ([Gamma et al., 1994](https://en.wikipedia.org/wiki/Design_Patterns)), cataloguing 23 reusable solutions to common software design problems. These patterns are covered in depth in Chapter 3.

### 1.2.4 The Internet Era and Agile Methods (1990s–2000s)

The World Wide Web transformed software from shrink-wrapped products shipped on disks to continuously evolving services. Release cycles had to shrink from years to weeks. Traditional plan-driven methods struggled to keep pace.

In 2001, seventeen software practitioners gathered in Snowbird, Utah, and published the [Agile Manifesto](https://agilemanifesto.org/) — a short document that valued:

> *Individuals and interactions over processes and tools*
> *Working software over comprehensive documentation*
> *Customer collaboration over contract negotiation*
> *Responding to change over following a plan*

Agile methods — including Scrum, Extreme Programming (XP), and Kanban — spread rapidly through the industry. They emphasised short iterations, continuous feedback, and adaptive planning rather than upfront specification.

### 1.2.5 DevOps and Continuous Delivery (2010s)

As agile teams delivered software faster, operations teams struggled to deploy and maintain it. The DevOps movement ([Kim et al., 2016](https://itrevolution.com/product/the-devops-handbook/)) broke down the wall between development and operations, promoting:

- Continuous integration (CI): merging code frequently, building and testing automatically
- Continuous delivery (CD): keeping software always in a deployable state
- Infrastructure as code: managing servers and environments through version-controlled scripts

This shift made the pipeline from code commit to production deployment a first-class engineering concern — covered in depth in Chapter 4.

### 1.2.6 The AI Era (2020s–Present)

In 2021, GitHub released Copilot, powered by OpenAI Codex — a large language model trained on billions of lines of public code. For the first time, AI could generate syntactically correct, contextually relevant code at the level of individual functions and files ([Chen et al., 2021](https://arxiv.org/abs/2107.03374)).

By 2023, models like GPT-4 and Claude could engage in multi-turn conversations about software design, debug complex issues, write test suites, and generate entire application scaffolds from natural language descriptions. By 2024–2025, *agentic* systems — AI that can plan, use tools, and execute code autonomously — began to handle multi-step engineering tasks with minimal human intervention.

This is where this book begins.

---

## 1.3 The Software Development Lifecycle (SDLC)

The Software Development Lifecycle (SDLC) is a structured process for planning, creating, testing, and deploying software. While specific SDLC models differ in their details, most share a common set of phases:

| Phase | Key Activities |
|---|---|
| **Requirements** | Understand what the system should do |
| **Design** | Decide how the system will be structured |
| **Implementation** | Write the code |
| **Testing** | Verify the system works correctly |
| **Deployment** | Release the system to users |
| **Maintenance** | Fix bugs, add features, keep the system running |

### 1.3.1 Waterfall

The Waterfall model, introduced by Winston Royce in 1970 (though Royce actually presented it as a flawed approach in the same paper), organises development as a strict sequence of phases ([Royce, 1970](http://www-scf.usc.edu/~csci201/lectures/Lecture11/royce1970.pdf)):

```
Requirements → Design → Implementation → Testing → Deployment → Maintenance
```

Each phase must be completed before the next begins. The model assumes requirements can be fully and correctly specified at the start.

**Strengths:**
- Clear milestones and deliverables
- Easy to manage and document
- Works well for projects with stable, well-understood requirements (e.g., certain embedded systems, government contracts)

**Weaknesses:**
- Requirements almost never remain stable
- Errors discovered late are expensive to fix
- Users see no working software until the end
- Poor fit for projects with high uncertainty

### 1.3.2 Agile Software Development

Agile is not a single methodology but a family of approaches united by the values in the [Agile Manifesto](https://agilemanifesto.org/). The core insight is that software requirements and solutions evolve through collaboration, and that the ability to respond to change is more valuable than adherence to a plan.

Agile teams work in short cycles called *iterations* or *sprints*, typically 1–4 weeks long. Each iteration produces a working, tested increment of software. Stakeholders review the increment and provide feedback that informs the next iteration.

Key Agile principles include:

- Deliver working software frequently (weeks, not months)
- Welcome changing requirements, even late in development
- Business people and developers work together daily
- Simplicity — the art of maximising the amount of work *not* done — is essential

### 1.3.3 Scrum

Scrum is the most widely adopted Agile framework ([Schwaber & Sutherland, 2020](https://scrumguides.org/scrum-guide.html)). It defines specific roles, events, and artefacts:

**Roles:**
- **Product Owner**: Represents stakeholders; owns and prioritises the product backlog
- **Scrum Master**: Facilitates the process; removes impediments; coaches the team
- **Development Team**: Self-organising group that delivers the increment

**Events:**
- **Sprint**: A time-boxed iteration of 1–4 weeks
- **Sprint Planning**: The team selects backlog items and plans the sprint
- **Daily Scrum**: A 15-minute daily standup to synchronise and identify blockers
- **Sprint Review**: The team demonstrates the increment to stakeholders
- **Sprint Retrospective**: The team reflects on the process and identifies improvements

**Artefacts:**
- **Product Backlog**: An ordered list of everything that might be needed in the product
- **Sprint Backlog**: The backlog items selected for the current sprint, plus the delivery plan
- **Increment**: The sum of all completed backlog items at the end of a sprint

```
┌─────────────────────────────────────────────────────────┐
│                    Product Backlog                       │
│  (ordered list of features, bugs, improvements)         │
└───────────────────────┬─────────────────────────────────┘
                        │ Sprint Planning
                        ▼
┌─────────────────────────────────────────────────────────┐
│                    Sprint (1–4 weeks)                    │
│                                                          │
│  Sprint Backlog → Daily Scrum → Working Increment        │
└───────────────────────┬─────────────────────────────────┘
                        │ Sprint Review + Retrospective
                        ▼
                  Next Sprint...
```

### 1.3.4 Kanban

Kanban, adapted from Toyota's manufacturing system by David Anderson ([Anderson, 2010](https://kanbanbooks.com/)), is a flow-based method that focuses on visualising work, limiting work in progress (WIP), and continuously improving flow.

A Kanban board visualises work as cards moving through columns:

```
┌──────────┬──────────────┬──────────────┬──────────┐
│ Backlog  │  In Progress │   In Review  │   Done   │
│          │   (WIP: 3)   │   (WIP: 2)   │          │
├──────────┼──────────────┼──────────────┼──────────┤
│ Task E   │ Task B       │ Task A       │ Task D   │
│ Task F   │ Task C       │              │          │
│ Task G   │              │              │          │
└──────────┴──────────────┴──────────────┴──────────┘
```

**Key Kanban practices:**
- **Visualise the workflow**: Make all work and its status visible
- **Limit WIP**: Prevent overloading; finish before starting more
- **Manage flow**: Track cycle time and throughput; identify bottlenecks
- **Improve collaboratively**: Use data to drive continuous improvement

Kanban suits teams with highly variable incoming work (e.g., support and maintenance teams) or those who want a lighter-weight alternative to Scrum's ceremonies.

---

## 1.4 Scope Creep

*Scope creep* refers to the gradual, uncontrolled expansion of a project's scope beyond its original boundaries. It is one of the most common causes of project failure in software engineering ([PMI, 2021](https://www.pmi.org/learning/library/scope-creep-causes-effects-solutions-6181)).

Scope creep happens when:

- Stakeholders request new features after the project has started
- Requirements are poorly defined, leaving room for interpretation
- The team adds features without formal approval
- External factors force new work mid-project

**Managing scope creep** requires:

1. Clear initial scope definition — document what is in scope and explicitly what is *out* of scope
2. Formal change control processes — structured mechanisms for requesting, evaluating, and approving scope changes
3. Prioritisation frameworks — structured approaches for deciding what gets built and when
4. Regular backlog grooming — reviewing and re-prioritising as understanding evolves

---

## 1.5 User Stories and Story Points

### 1.5.1 User Stories

A *user story* is a short, simple description of a feature told from the perspective of the person who wants it. The standard format is:

> **As a** [type of user], **I want** [some goal] **so that** [some reason].

User stories originated in Extreme Programming ([Beck, 1999](https://www.oreilly.com/library/view/extreme-programming-explained/0201616416/)) and serve as placeholders for conversations rather than comprehensive specifications.

**Examples:**

> As a **registered user**, I want to **reset my password via email** so that **I can regain access to my account if I forget my password**.

> As a **project manager**, I want to **filter tasks by assignee** so that **I can quickly see the workload of individual team members**.

Good user stories follow the **INVEST** criteria ([Wake, 2003](https://xp123.com/articles/invest-in-good-stories-and-smart-tasks/)):

| Letter | Meaning | Description |
|---|---|---|
| **I** | Independent | Stories can be developed in any order |
| **N** | Negotiable | Details are open to discussion |
| **V** | Valuable | Delivers value to users or stakeholders |
| **E** | Estimable | The team can estimate the effort |
| **S** | Small | Completable within a single sprint |
| **T** | Testable | Acceptance criteria can be verified |

### 1.5.2 Story Points

*Story points* are a unit of measure for estimating the relative effort or complexity of user stories. They are intentionally abstract — they do not map directly to hours or days — encouraging teams to think about relative complexity rather than precise time estimates.

Teams typically use a modified Fibonacci sequence: **1, 2, 3, 5, 8, 13, 21**. The increasing gaps reflect growing uncertainty in estimating large, complex work.

**Planning Poker** is a common estimation technique ([Grenning, 2002](https://wingman-sw.com/articles/planning-poker)): each team member privately selects a card with their estimate; all cards are revealed simultaneously; significant discrepancies prompt discussion until the team reaches consensus.

Story points enable **velocity tracking** — the total points completed per sprint gives the team's *velocity*, which predicts future throughput and informs release planning.

---

## 1.6 Prioritisation: The MoSCoW Framework

The **MoSCoW framework** ([Clegg & Barker, 1994](https://www.dsdm.org/)) provides a shared vocabulary for prioritisation:

| Category | Meaning | Guideline |
|---|---|---|
| **M**ust Have | Non-negotiable; the system cannot launch without these | ~60% of effort |
| **S**hould Have | Important but not vital; workarounds exist if omitted | ~20% of effort |
| **C**ould Have | Nice to have; included only if time permits | ~20% of effort |
| **W**on't Have | Explicitly excluded from this release | Documented, not built |

The "Won't Have" category is often the most valuable: it makes explicit what is being deliberately deferred, preventing scope creep through shared agreement.

**Example — a task management application:**

| Feature | MoSCoW |
|---|---|
| Create, read, update, delete tasks | Must Have |
| Assign tasks to team members | Must Have |
| Email notifications on task assignment | Should Have |
| Drag-and-drop task reordering | Could Have |
| Integration with Slack | Won't Have (this release) |

---

## 1.7 How AI Is Reshaping the SDLC

Each SDLC phase is being transformed by AI tools. This section introduces the high-level picture; subsequent chapters examine each transformation in depth.

### Requirements

AI can help elicit requirements by analysing stakeholder interview transcripts, extracting themes from large documents, and generating draft user stories from rough natural language descriptions. However, AI cannot replace the human judgment required to resolve conflicting stakeholder needs or understand organisational context (Chapter 2).

### Design

Large language models can generate architectural diagrams, suggest design patterns, and produce code scaffolds from specifications. Chapter 3 examines how to use these capabilities critically.

### Implementation

This is where AI's current impact is most visible. AI coding assistants can complete functions, suggest API calls, translate between languages, and generate boilerplate. Studies suggest significant productivity gains for routine coding tasks ([Peng et al., 2023](https://arxiv.org/abs/2302.06590)). But AI-generated code requires the same critical evaluation as any unreviewed code — it can be subtly wrong, insecure, or incompatible with the surrounding system.

### Testing

AI can generate test cases, identify edge cases, and suggest fixes for failing tests. Chapters 4 and 7 address the critical question of how to evaluate AI-generated tests themselves.

### Deployment and Maintenance

AI-powered tools are beginning to automate parts of deployment (intelligent rollouts, anomaly detection) and maintenance (automated bug triage, code smell detection). These capabilities are still maturing.

### The Shifting Role of the Engineer

The cumulative effect is a shift in what software engineers spend their time on. The bottleneck is moving from *writing code* to *defining problems, specifying intent, and evaluating outcomes*. This is the core claim of AI-native engineering — and the reason this book focuses on thinking and evaluation skills, not just tool usage.

As Andrej Karpathy observed, we may be entering an era of ["Software 2.0"](https://karpathy.medium.com/software-2-0-a64152b37c35) — where neural networks trained on data replace hand-written code for increasing classes of problems, and the engineer's job shifts to curating data, defining objectives, and evaluating model behaviour.

---

## 1.8 Tutorial: Setting Up Your AI-Assisted Development Environment

This tutorial walks through setting up a Python development environment with AI assistance integrated at key points.

### Prerequisites

- Python 3.11 or later ([python.org](https://www.python.org/downloads/))
- Git ([git-scm.com](https://git-scm.com/))
- VS Code ([code.visualstudio.com](https://code.visualstudio.com/))
- A GitHub account ([github.com](https://github.com/))

### Step 1: Create a Virtual Environment

```bash
mkdir ai_native_project
cd ai_native_project

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # macOS/Linux
# venv\Scripts\activate         # Windows

python --version                # Confirm activation
```

### Step 2: Initialise a Git Repository

```bash
git init
cat > .gitignore << 'EOF'
venv/
__pycache__/
*.pyc
.env
EOF
git add .gitignore
git commit -m "Initial commit: add .gitignore"
```

### Step 3: Install Core Development Tools

```bash
pip install pytest ruff mypy pre-commit
pip freeze > requirements.txt
```

### Step 4: Configure Ruff and Mypy

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W"]

[tool.mypy]
python_version = "3.11"
strict = true
```

### Step 5: Set Up Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

```bash
pre-commit install
```

### Step 6: Verify the Setup

```python
# src/calculator.py
def add(a: float, b: float) -> float:
    return a + b

def divide(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```

```python
# tests/test_calculator.py
import pytest
from src.calculator import add, divide

def test_add() -> None:
    assert add(2, 3) == 5
    assert add(-1, 1) == 0

def test_divide() -> None:
    assert divide(10, 2) == 5.0

def test_divide_by_zero() -> None:
    with pytest.raises(ValueError):
        divide(1, 0)
```

```bash
pytest tests/ -v
```

Expected output:
```
tests/test_calculator.py::test_add PASSED
tests/test_calculator.py::test_divide PASSED
tests/test_calculator.py::test_divide_by_zero PASSED
3 passed in 0.12s
```

This environment — version control, dependency isolation, linting, type checking, pre-commit hooks, and a test framework — is the foundation on which every subsequent chapter builds.

---

## 1.9 Project Milestone: Define Your Course Project

### Project Brief

Throughout this course you will build a **Task Management API** — a backend system that allows users to create projects, manage tasks, assign them to team members, and track progress. This is a deliberately familiar problem domain: the focus is on *how* you build it using AI-native practices, not on inventing a novel application.

The project grows across 12 weeks:
- **Weeks 1–4**: Requirements, design, tests, and CI/CD for the core API
- **Weeks 5–8**: AI-native development of features using agents and evaluation
- **Weeks 9–12**: Security hardening, ethics review, productivity analysis, and reflection

### This Week's Deliverables

1. **Team charter** (if in a team): Names, agreed roles, and working norms.
2. **Scope statement**: One paragraph describing what your system *will* do; one paragraph explicitly describing what it *will not* do.
3. **MoSCoW list**: At least 10 features categorised as Must/Should/Could/Won't.
4. **Development environment**: A GitHub repository with virtual environment, linter, pre-commit hooks, and at least one passing test.

Submit a `README.md` in the root of your repository containing the team charter, scope statement, and MoSCoW list.

---

## Summary

This chapter traced software engineering from the 1968 NATO crisis through structured programming, object-oriented design, Agile, DevOps, and into the current AI era. Key takeaways:

- Software engineering applies disciplined processes to manage the complexity of building reliable, maintainable systems at scale.
- The SDLC provides a framework of phases that all development models address in different ways.
- Waterfall is plan-driven and sequential; Agile is iterative and adaptive. Scrum and Kanban are specific Agile frameworks with distinct practices.
- Scope creep is one of the most common causes of project failure; MoSCoW prioritisation is a practical tool for managing it.
- User stories and story points are Agile tools for capturing requirements from the user's perspective and estimating relative effort.
- AI is reshaping every phase of the SDLC, shifting the engineer's primary challenge from writing code to framing problems, specifying intent, and evaluating AI-generated outputs.

---

## Review Questions

1. What was the "software crisis" identified at the 1968 NATO conference, and what response did it prompt?
2. Compare Waterfall and Agile: in what circumstances might each be more appropriate?
3. What is the difference between Scrum and Kanban? Give an example of a team context that suits each.
4. Write a user story for: *"Users should be able to export their task list as a PDF."* Include acceptance criteria.
5. Apply the MoSCoW framework to a simple e-commerce website. List at least 8 features across the four categories.
6. In your own words, explain how AI is changing the role of the software engineer. What skills become more important? What becomes less important?

---

## References

- Naur, P., & Randell, B. (Eds.). (1969). *Software Engineering: Report on a conference sponsored by the NATO Science Committee*. [http://homepages.cs.ncl.ac.uk/brian.randell/NATO/nato1968.PDF](http://homepages.cs.ncl.ac.uk/brian.randell/NATO/nato1968.PDF)
- Brooks, F. P. (1975). *The Mythical Man-Month: Essays on Software Engineering*. Addison-Wesley. [https://en.wikipedia.org/wiki/The_Mythical_Man-Month](https://en.wikipedia.org/wiki/The_Mythical_Man-Month)
- Gamma, E., Helm, R., Johnson, R., & Vlissides, J. (1994). *Design Patterns: Elements of Reusable Object-Oriented Software*. Addison-Wesley. [https://en.wikipedia.org/wiki/Design_Patterns](https://en.wikipedia.org/wiki/Design_Patterns)
- Beck, K. et al. (2001). *Manifesto for Agile Software Development*. [https://agilemanifesto.org/](https://agilemanifesto.org/)
- Schwaber, K., & Sutherland, J. (2020). *The Scrum Guide*. [https://scrumguides.org/scrum-guide.html](https://scrumguides.org/scrum-guide.html)
- Anderson, D. J. (2010). *Kanban: Successful Evolutionary Change for Your Technology Business*. Blue Hole Press. [https://kanbanbooks.com/](https://kanbanbooks.com/)
- Royce, W. W. (1970). Managing the Development of Large Software Systems. *Proceedings of IEEE WESCON*. [http://www-scf.usc.edu/~csci201/lectures/Lecture11/royce1970.pdf](http://www-scf.usc.edu/~csci201/lectures/Lecture11/royce1970.pdf)
- Kim, G., Humble, J., Debois, P., & Willis, J. (2016). *The DevOps Handbook*. IT Revolution Press. [https://itrevolution.com/product/the-devops-handbook/](https://itrevolution.com/product/the-devops-handbook/)
- Chen, M., et al. (2021). Evaluating Large Language Models Trained on Code. *arXiv*. [https://arxiv.org/abs/2107.03374](https://arxiv.org/abs/2107.03374)
- Peng, S., et al. (2023). The Impact of AI on Developer Productivity: Evidence from GitHub Copilot. *arXiv*. [https://arxiv.org/abs/2302.06590](https://arxiv.org/abs/2302.06590)
- Wake, B. (2003). INVEST in Good Stories, and SMART Tasks. [https://xp123.com/articles/invest-in-good-stories-and-smart-tasks/](https://xp123.com/articles/invest-in-good-stories-and-smart-tasks/)
- Karpathy, A. (2017). Software 2.0. [https://karpathy.medium.com/software-2-0-a64152b37c35](https://karpathy.medium.com/software-2-0-a64152b37c35)
- PMI. (2021). Scope Creep. Project Management Institute. [https://www.pmi.org/learning/library/scope-creep-causes-effects-solutions-6181](https://www.pmi.org/learning/library/scope-creep-causes-effects-solutions-6181)
- Grenning, J. (2002). Planning Poker or How to avoid analysis paralysis while release planning. [https://wingman-sw.com/articles/planning-poker](https://wingman-sw.com/articles/planning-poker)
