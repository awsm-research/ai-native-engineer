# Chapter 5: The AI-Native Development Paradigm

> *"The question is not whether AI will change software engineering. It already has. The question is whether you are shaping that change or being shaped by it."*

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Distinguish AI-native engineering from AI-assisted development.
2. Describe the AI-Native SDLC and each of its four phases: Spec, Generate, Evaluate, Refine.
3. Explain how AI coding agents differ from copilots and what capabilities agents introduce.
4. Identify the key components of an agent: tool use, planning, and memory.
5. Explain the purpose of protocols like MCP and A2A in multi-agent systems.
6. Select an appropriate model size and estimate cost for a given engineering task.
7. Apply the AI-native lens to re-examine your course project specification.

---

## 5.1 Two Ways of Working with AI

Before 2021, AI's role in software development was largely confined to autocomplete suggestions, simple code search, and static analysis. The release of GitHub Copilot changed this: for the first time, AI could generate contextually relevant code at the function level from a comment or function signature.

Most developers initially used these tools as *accelerators* — they wrote the specification, the design, the tests, and then used AI to fill in boilerplate or suggest implementations for functions they already had in mind. This is *AI-assisted development*: the engineer's workflow is unchanged, but AI speeds up some steps.

*AI-native engineering* is a different posture. It recognises that AI has changed not just the speed of some steps, but the nature of the engineer's job itself. In AI-native engineering:

- **Specifications become the primary engineering artefact** — what you write for the AI is as important as what the AI writes back
- **Generation is a commodity** — producing code is no longer the bottleneck
- **Evaluation becomes the critical skill** — determining whether generated code is correct, secure, and appropriate requires deep engineering judgment
- **The SDLC is restructured** around the capabilities and failure modes of AI systems

This is not a prediction about some future state. It is a description of how leading software teams are working today ([Khlaaf et al., 2022](https://arxiv.org/abs/2212.09251)).

---

## 5.2 The Evolution of AI Coding Tools

Understanding where current tools came from helps calibrate what they can and cannot do.

### 5.2.1 Copilots (2021–present)

GitHub Copilot, powered by OpenAI Codex ([Chen et al., 2021](https://arxiv.org/abs/2107.03374)), was the first widely deployed AI coding tool. It operates in-editor, completing code as the developer types. It works well for:

- Boilerplate and repetitive patterns
- Simple algorithms with clear names
- Common library API usage
- Translating between languages

It works poorly for:
- Multi-file context
- Long-range dependencies
- System-level design
- Novel or domain-specific logic

### 5.2.2 Chat-Based Assistants (2022–present)

ChatGPT, Claude, and Gemini introduced multi-turn conversation with AI. Engineers could now describe a problem in natural language, receive an explanation or solution, and iterate through dialogue. Chat interfaces handle more context than inline completion and support discussion of design decisions.

### 5.2.3 AI Coding Agents (2024–present)

AI coding agents — such as Claude Code, Devin ([Cognition, 2024](https://cognition.ai/blog/introducing-devin)), and Cursor in agent mode — represent the next step. An agent is not just responding to prompts; it is *acting in the world*:

- It can read and write files
- It can run code and tests
- It can browse the web and read documentation
- It can use APIs and external tools
- It can plan a multi-step task and execute each step autonomously

This changes the nature of human-AI collaboration significantly. Rather than the engineer making every decision and using AI to execute individual steps, the engineer can delegate a whole task to an agent, monitor its progress, and intervene when it goes wrong.

---

## 5.3 The AI-Native SDLC

The traditional SDLC (Requirements → Design → Implementation → Testing → Deployment) maps awkwardly onto AI-native workflows. A more useful model for AI-native development is:

```
Spec → Generate → Evaluate → Refine
  ↑                              │
  └──────────────────────────────┘
```

This cycle is iterative and fast — a single round can take minutes rather than days.

### 5.3.1 Spec

*Specification* is the act of describing, precisely and completely, what you want the AI to produce. In AI-native engineering, specification is the primary engineering activity — not implementation.

A good specification for AI includes:

- **Context**: What is the purpose of this component? What does it fit into?
- **Inputs and outputs**: What does the function receive? What should it return?
- **Constraints**: What invariants must hold? What should the function explicitly NOT do?
- **Examples**: What are the expected input-output pairs for key cases?
- **Quality attributes**: What performance, security, or style requirements apply?

The quality of your specification directly determines the quality of what is generated. Vague inputs produce vague outputs. This is the central insight of prompt engineering, covered in depth in Chapter 6.

### 5.3.2 Generate

Generation is the act of invoking the AI with your specification to produce code, tests, documentation, or other artefacts. In the AI-native paradigm, this step is largely mechanical — the creative and intellectual work is in the specification and evaluation phases.

Key decisions at the generate step:
- **Which model to use**: Different models have different strengths, costs, and context windows
- **Temperature and sampling**: Lower temperatures produce more deterministic output; higher temperatures produce more varied output
- **Context to include**: What files, documentation, or examples should accompany the specification?

### 5.3.3 Evaluate

Evaluation is the act of determining whether the generated output meets the specification. This is where most of the engineering judgment in AI-native development lives.

Evaluation is covered in depth in Chapter 7. At a high level, it involves:

- Running automated tests
- Static analysis and type checking
- Manual code review
- Behavioural testing against real or synthetic data
- Security review

Crucially, evaluation must happen *before* the generated code is trusted. AI-generated code can pass visual inspection while containing subtle bugs, security vulnerabilities, or logical errors that only surface under specific conditions.

### 5.3.4 Refine

If evaluation reveals problems, refinement involves returning to the specification (to add constraints or correct misunderstandings) and regenerating. Refinement may involve:

- Adding failing test cases to the specification ("the function should return X when given Y")
- Adding explicit constraints that the AI violated ("do not use X, use Y instead")
- Breaking the specification into smaller, more tractable pieces
- Changing the approach entirely

The Spec → Generate → Evaluate → Refine loop typically runs multiple times before a satisfactory result is reached. The discipline is in specification quality and evaluation rigour, not in generating more.

---

## 5.4 What Is an AI Coding Agent?

An AI coding agent is a system in which a large language model can not only generate text, but also take *actions* in the world — reading and writing files, executing code, calling APIs, and browsing the web — in service of a multi-step goal.

The term "agent" comes from AI research ([Russell & Norvig, 2020](https://aima.cs.berkeley.edu/)), where an agent is any system that perceives its environment and takes actions to achieve goals. In the context of AI coding, the environment is the software development environment (the codebase, the terminal, the browser).

### 5.4.1 Tool Use

The most fundamental capability that distinguishes an agent from a chatbot is *tool use* — the ability to invoke external functions and incorporate their results into the agent's reasoning.

Common tools available to coding agents:

| Tool | Description |
|---|---|
| `read_file(path)` | Read the contents of a file |
| `write_file(path, content)` | Write content to a file |
| `run_command(cmd)` | Execute a shell command and return the output |
| `search_web(query)` | Search the web and return results |
| `fetch_url(url)` | Fetch the contents of a URL |
| `call_api(endpoint, params)` | Make an HTTP API call |

When an agent has access to these tools, it can autonomously investigate a codebase, identify a bug, write a fix, run the tests to verify it, and commit the change — all without human intervention at each step.

### 5.4.2 Planning

Planning is the ability to break a high-level goal into a sequence of sub-tasks and execute them in order, adapting the plan as new information is discovered.

A naive agent executes tasks sequentially without reflection. A more sophisticated agent uses a *plan-execute-observe* loop:

```
Goal: "Add input validation to the task creation endpoint"

Plan:
  1. Read the current task creation endpoint code
  2. Read the existing tests for this endpoint
  3. Identify which input fields currently lack validation
  4. Write validation logic for each field
  5. Write tests for the new validation
  6. Run the tests to verify

Execute step 1 → Observe result → Update plan if needed → Execute step 2 → ...
```

Modern agents use techniques like ReAct (Reason + Act) ([Yao et al., 2022](https://arxiv.org/abs/2210.03629)) to interleave reasoning and action, producing more reliable multi-step behaviour.

### 5.4.3 Memory

Agents need different types of memory to function effectively across long tasks:

- **In-context memory**: The current conversation history and tool results — limited by the model's context window
- **External memory**: Files, databases, or vector stores that the agent can read and write — persistent across sessions
- **Semantic memory**: Compressed summaries of past interactions — allows the agent to operate over longer time horizons than the context window permits

Managing what information to put in context (and what to leave out) is a significant challenge in building effective agents. Too much context causes the model to "lose focus"; too little context causes it to make decisions without relevant information.

---

## 5.5 Agentic Protocols: MCP and A2A

As agents have proliferated, the need for standard protocols for connecting them to tools and to each other has emerged.

### 5.5.1 Model Context Protocol (MCP)

The Model Context Protocol ([Anthropic, 2024](https://modelcontextprotocol.io/)) is an open standard for connecting AI models to external tools, data sources, and services. It defines a standard interface so that any MCP-compatible tool can be used by any MCP-compatible agent without custom integration code.

MCP defines three types of capabilities that a server can expose to an agent:

- **Tools**: Functions the agent can invoke (e.g., search a database, read a file)
- **Resources**: Data sources the agent can read (e.g., a file, a database record)
- **Prompts**: Pre-defined prompt templates for common tasks

With MCP, a developer can write a single MCP server for their GitHub repository, and any agent that supports MCP can read issues, create pull requests, and search code — without either the agent developer or the repository owner needing to write custom integration code.

### 5.5.2 Agent-to-Agent Protocol (A2A)

The Agent-to-Agent protocol ([Google, 2025](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)) is an open standard for agents to discover and communicate with each other. It enables multi-agent systems where a *coordinator agent* delegates sub-tasks to *worker agents*, each specialised for a particular domain.

In a multi-agent software development system, for example:
- A **coordinator agent** receives a feature request and breaks it into sub-tasks
- A **requirements agent** refines the specification
- A **coding agent** implements the feature
- A **testing agent** writes and runs tests
- A **review agent** checks the code for security and style issues

These agents communicate via A2A, passing task descriptions, results, and feedback in a standardised format. Multi-agent architectures are covered in depth in Chapter 8.

---

## 5.6 Choosing the Right Model

Not all AI models are equally suited to every task. Selecting the right model for a given purpose is an engineering decision with real consequences for quality, speed, and cost.

### 5.6.1 The Model Capability Spectrum

Modern AI providers offer models across a spectrum from small/fast/cheap to large/slow/capable. The Anthropic model family as of 2025:

| Model | Strengths | Context Window | Relative Cost | Best For |
|---|---|---|---|---|
| **Claude Haiku** | Speed, low latency | 200K tokens | Low | High-volume, simple tasks: docstring generation, lint fixes, short completions |
| **Claude Sonnet** | Balanced capability and speed | 200K tokens | Medium | Most engineering tasks: feature implementation, code review, test generation |
| **Claude Opus** | Maximum capability, complex reasoning | 200K tokens | High | Difficult tasks: architectural decisions, complex debugging, security analysis |

> **Check the current model list.** Model families evolve rapidly. Always verify available models and their capabilities at [https://docs.anthropic.com/en/docs/about-claude/models](https://docs.anthropic.com/en/docs/about-claude/models) before selecting a model for production use.

For OpenAI, the equivalent spectrum is GPT-4o-mini (fast/cheap) → GPT-4o (balanced) → o1/o3 (reasoning-heavy). For Google: Gemini Flash → Gemini Pro → Gemini Ultra. The selection principle is the same regardless of provider.

### 5.6.2 Matching Model to Task

```python
import anthropic

client = anthropic.Anthropic()

# Use a smaller, faster model for high-volume, simple tasks
def generate_docstring(function_code: str) -> str:
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Fast, cheap — appropriate for docstrings
        max_tokens=256,
        messages=[{"role": "user", "content": f"Write a one-line docstring for:\n{function_code}"}],
    )
    return response.content[0].text


# Use a capable model for complex reasoning tasks
def security_review(code: str) -> str:
    response = client.messages.create(
        model="claude-opus-4-7",  # Full capability — security analysis needs it
        max_tokens=2048,
        messages=[{"role": "user", "content": f"Security review:\n{code}"}],
    )
    return response.content[0].text


# Use a balanced model for most feature development
def implement_feature(specification: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",  # Balanced — good quality at reasonable cost
        max_tokens=4096,
        messages=[{"role": "user", "content": specification}],
    )
    return response.content[0].text
```

### 5.6.3 Context Window Considerations

Context window size — the maximum amount of text a model can process in a single call — directly affects specification design. All current Claude models support 200K tokens (roughly 150,000 words), which is sufficient for most codebases. However, larger contexts:

- Cost more (most providers charge per token)
- Are processed more slowly
- May cause the model to "lose focus" on specific instructions buried in long context ([Liu et al., 2023](https://arxiv.org/abs/2307.03172))

**Practical guideline**: Keep specification prompts under 2,000 tokens for most code generation. Reserve large context for tasks that genuinely need it — understanding an entire module before refactoring, for example.

### 5.6.4 Cost Estimation

Rough cost estimation for common tasks (prices vary; check provider pricing pages for current rates):

| Task | Typical tokens | Model tier | Approx. cost per 1,000 tasks |
|---|---|---|---|
| Docstring generation | ~500 in + ~100 out | Small | < $0.10 |
| Function implementation | ~1,000 in + ~500 out | Medium | ~$1–3 |
| Security review | ~2,000 in + ~1,000 out | Large | ~$15–30 |
| Agent task (10 steps) | ~20,000 total | Medium | ~$10–20 |

For a 12-person team running 50 AI-assisted tasks per day, monthly API costs typically range from $50–500 depending on task mix and model selection — comparable to a single SaaS tool licence.

---

## 5.7 The Shifting Role of the Engineer

The emergence of AI coding agents does not eliminate the need for software engineers — but it does fundamentally change what engineers spend their time on.

### 5.6.1 What Changes

**Less time on**: Implementing boilerplate, writing routine CRUD code, translating designs into code, looking up API documentation, writing test scaffolding.

**More time on**: Defining the problem clearly, writing precise specifications, evaluating generated outputs, architectural decisions, security review, stakeholder communication.

### 5.6.2 The Engineer as Principal

In agentic systems, the human engineer acts as a *principal* — the authority that defines goals, sets constraints, and approves outcomes. The agent acts as an *executor* — planning and carrying out the steps needed to achieve the goal.

This relationship requires a new set of skills:

- **Goal decomposition**: Breaking a complex goal into tasks small enough for an agent to handle reliably
- **Constraint specification**: Defining what the agent must NOT do, not just what it should do
- **Output evaluation**: Assessing whether the agent's output is correct, secure, and appropriate
- **Failure diagnosis**: Understanding why an agent went wrong and how to prevent recurrence

### 5.6.3 Skills That Endure

The foundational skills of software engineering — understanding algorithms, system design, testing, security, and communication — become *more* valuable in the AI-native era, not less. They are the skills needed to write good specifications, evaluate AI outputs, and diagnose agent failures.

Engineers who treat AI tools as magic boxes that produce correct code will be frustrated and vulnerable. Engineers who understand the capabilities and failure modes of AI systems will be significantly more productive.

---

## 5.8 Tutorial: Working with an AI Coding Agent End-to-End

This tutorial demonstrates the AI-Native SDLC cycle using the Anthropic API to implement a feature for the course project.

### The Task

Add a `filter_tasks` function to the task service that filters tasks by status, priority, and assignee.

### Step 1: Write the Specification

```python
# spec: filter_tasks function
# 
# Context: Part of a task management API backend (Python 3.11, no framework)
# 
# Function signature:
#   filter_tasks(tasks, status=None, priority=None, assignee=None) -> list[Task]
#
# Behaviour:
# - Returns all tasks if no filters are provided
# - Filters by status if status is provided (exact match)
# - Filters by priority if priority is provided (exact match, integer 1-4)
# - Filters by assignee if assignee is provided (exact match on assignee email)
# - Multiple filters are ANDed (all must match)
# - Returns an empty list (not None) if no tasks match
# - Does NOT modify the input list
# - Raises TypeError if tasks is not a list
# - Raises ValueError if priority is provided but not in range 1-4
#
# Examples:
# filter_tasks([task1, task2], status="open") -> [task1] (if task1.status=="open")
# filter_tasks([task1], status="open", priority=2) -> [] (if task1.priority!=2)
# filter_tasks([]) -> []
```

### Step 2: Generate an Implementation

```python
import anthropic

client = anthropic.Anthropic()

specification = """
Implement a Python function `filter_tasks` with the following specification:

Function signature:
  filter_tasks(tasks: list[Task], status: str | None = None,
               priority: int | None = None,
               assignee: str | None = None) -> list[Task]

Where Task is a dataclass with fields: id (UUID), title (str),
priority (int, 1-4), status (str), assignee (str | None).

Behaviour:
- Returns all tasks if no filters are provided
- Multiple filters are ANDed (all must match)
- Returns an empty list if no tasks match
- Does NOT modify the input list
- Raises TypeError if tasks is not a list
- Raises ValueError if priority is provided but not in range 1-4

Use Python 3.11 type hints throughout. No comments needed unless
a line would confuse an experienced Python developer.
"""

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    messages=[{"role": "user", "content": specification}],
)

print(response.content[0].text)
```

### Step 3: Evaluate the Output

Review the generated code for:

- [ ] Correct type hints on function signature and return type
- [ ] Does not modify the input list (uses a new list or generator)
- [ ] TypeError raised when tasks is not a list
- [ ] ValueError raised when priority is out of range
- [ ] Empty list returned (not None) when no matches
- [ ] All filter conditions ANDed correctly

Write tests to verify each behaviour before accepting the code.

### Step 4: Refine

If the generated code is missing the `TypeError` check, add this constraint to the specification:

```python
# Add to specification:
# - MUST raise TypeError (not just return []) if tasks is not a list
#   This is important because silent failures mask programmer errors
```

Regenerate and re-evaluate.

