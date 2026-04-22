# Preface

## About This Book

This book is about a shift in what software engineers actually do.

For most of the history of the profession, the primary bottleneck in software development was *writing code*: turning a clear understanding of the problem into a working implementation. Tools, languages, and processes were all designed to help engineers write code faster, more reliably, and with fewer defects.

That bottleneck is moving. AI coding systems — capable assistants that generate syntactically correct, contextually relevant code from natural language descriptions — are now widely available, fast, and good enough to handle a substantial fraction of routine implementation work. The bottleneck is shifting to the activities that surround implementation: defining problems precisely, specifying intent clearly, and evaluating whether what was generated is actually correct, secure, and appropriate.

This book teaches those surrounding activities. It is not a book about which tools to use or how to write clever prompts. It is a book about how to engineer software in an environment where AI is a first-class collaborator — and what that demands of the engineer.

---

## Who This Book Is For

**Primary readers:**
- Software engineers transitioning from traditional to AI-assisted workflows who want sustainable, tool-independent skills
- Advanced undergraduate and graduate students in software engineering
- Senior developers and tech leads adapting team practices

**Secondary readers:**
- Engineering managers redefining development processes
- Researchers in software engineering and AI engineering

**What you need to bring:**
- Comfort with at least one programming language (examples are in Python)
- Familiarity with basic programming concepts: functions, classes, loops, conditionals
- Some exposure to version control (git) and the command line

**What you do not need:**
- Prior experience with AI coding tools
- A background in machine learning or deep learning
- Advanced knowledge of Python — the examples use standard library features and widely-adopted packages

**Where prior knowledge matters more:**
- Chapter 3 (Design, Architecture, Patterns) assumes familiarity with object-oriented programming. If you are coming from a procedural or functional background, the OOP sections may require extra time.
- Chapter 9 (Security) assumes familiarity with HTTP and basic web concepts. If you are new to backend development, consider reading an introductory web security primer first.

---

## How to Use This Book

This book is written for a 12-week university course, but it is structured so that it can be used in several ways.

### Path A: 12-Week Course (Recommended)

Follow the chapters in order, one per week. Each chapter builds on the previous and contributes one milestone to the running course project — a Task Management API that grows from a scope statement (Week 1) to a complete AI-native system (Week 12).

```
Weeks 1–4:  SE Foundations (Chapters 1–4)
Weeks 5–8:  AI-Native Practice (Chapters 5–8)
Weeks 9–12: Security, Ethics, Productivity, Future (Chapters 9–12)
```

The project milestones at the end of each chapter are the primary assessment vehicle. Submit them on a weekly cadence and use peer review to compare approaches.

### Path B: Practitioner Self-Study

If you are an experienced engineer who wants to develop AI-native skills specifically, start with Chapter 5 (The AI-Native Development Paradigm) to calibrate where you are, then read Chapters 6–8 in order. Use Chapters 1–4 as reference when the foundations feel shaky, and Chapters 9–12 for the governance and strategy dimensions.

Recommended reading order: 5 → 6 → 7 → 8 → 9 → 10 → 1–4 (reference) → 11 → 12

### Path C: Team Reference

If your team is adopting AI tools and you want to use this as a shared reference, the most immediately useful chapters are:

| Need | Chapter |
|---|---|
| Writing better AI specifications | 6 |
| Evaluating AI-generated code | 7 |
| Setting up agents for development tasks | 8 |
| Security review of AI-generated code | 9 |
| AI use policies and ethics | 10 |
| Measuring team productivity | 11 |

---

## A Note on Tools and Vendors

All code examples in this book use Python and the [Anthropic API](https://docs.anthropic.com/) with Claude models. This choice is deliberate and transparent, not an endorsement.

**Why Anthropic/Claude:**
- Claude has a generous context window (up to 200K tokens) that supports the long specification prompts developed in Chapter 6
- The Anthropic API's explicit `system` / `user` / `assistant` message structure maps directly to the role-prompting patterns in Chapter 6
- The tool use API (Chapter 8) has a clean, explicit schema that makes agent mechanics easy to teach
- All examples are testable with a free or low-cost API tier

**This is not a sponsored book.** No commercial relationship exists between the author and Anthropic or any other AI provider mentioned.

**These principles apply to any LLM provider.** Every concept in this book — the AI-native SDLC, specification design, evaluation-driven development, agentic orchestration — applies equally to OpenAI GPT models, Google Gemini, Meta Llama, Mistral, and future models not yet released. The Anthropic API is the *implementation vehicle*, not the *subject*. Where examples use Anthropic-specific classes (`anthropic.Anthropic()`, `client.messages.create()`), the equivalent calls for other providers are:

| Concept | Anthropic (this book) | OpenAI equivalent | Generic pattern |
|---|---|---|---|
| Client init | `anthropic.Anthropic()` | `openai.OpenAI()` | Provider client |
| Completion | `client.messages.create(model=..., messages=[...])` | `client.chat.completions.create(model=..., messages=[...])` | Call with model + messages |
| System prompt | `system="..."` parameter | `{"role": "system", "content": "..."}` in messages | First message or system param |
| Tool definition | `tools=[{name, description, input_schema}]` | `tools=[{type, function: {name, description, parameters}}]` | JSON schema per tool |

See [Appendix C](./appendix_c.md) for provider-agnostic wrappers and guidance on applying these examples to other languages.

**Models change.** The specific model IDs used in examples (`claude-opus-4-7`, `claude-haiku-4-5-20251001`) are current as of writing. New model versions are released regularly. Always check [https://docs.anthropic.com/en/docs/about-claude/models](https://docs.anthropic.com/en/docs/about-claude/models) for the current model list. The principles in this book are model-version-independent; only the model ID strings need updating.

---

## The Running Project

Starting in Chapter 1, you will build a **Task Management API** — a backend system for software development teams to create projects, manage tasks, assign work, and track progress. This is a deliberately familiar problem domain. The focus is not on inventing a novel application but on applying AI-native engineering practices to a realistic, growing system.

By the end of Chapter 12, you will have:
- A requirements specification and design document
- A Python REST API with full test coverage
- A CI/CD pipeline with automated quality gates
- AI-generated features developed using the Spec → Generate → Evaluate → Refine cycle
- An agentic workflow that automates a development task
- Security review, licence audit, and responsible AI assessment

The project is intentionally modest in scope so that the *process* — not the product — can be the focus of each week.

---

## Companion Resources

All code examples are available at: [github.com/awsm-research/ai-native-engineer](https://github.com/awsm-research/ai-native-engineer)

For updates on regulatory changes (EU AI Act, etc.) and new tool guidance, check the repository's `UPDATES.md` file. The landscape changes faster than print allows.

---

*Associate Professor Kla Tantithamthavorn*
*Monash University*
*2026*
