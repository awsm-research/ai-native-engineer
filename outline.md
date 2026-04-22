Title: AI-Native Engineering: Thinking, Building, and Validating in the Age of Agents

---

1. Overview

Software engineering is undergoing a fundamental shift. The bottleneck is no longer writing code—it is defining problems, specifying intent, and evaluating outcomes in collaboration with AI systems.

This book introduces a new paradigm: AI-native software engineering.

Rather than focusing on specific tools or rapidly changing frameworks, this book teaches enduring principles:

- How to frame problems for AI systems
- How to write precise, testable specifications
- How to evaluate and validate AI-generated solutions
- How to design robust, secure, and ethical AI-enabled systems

The book bridges the gap between:

- Academic software engineering foundations, and
- Real-world AI-assisted development practices

It equips readers not just to use AI, but to engineer with AI as a first-class collaborator.


2. Target Audience

Primary Audience
- Software engineers transitioning to AI-assisted workflows
- Senior developers and tech leads adapting team practices
- Graduate and advanced undergraduate students in software engineering

Secondary Audience
- Engineering managers redefining development processes
- Researchers in software engineering and AI engineering
- Bootcamp graduates seeking durable, non-tool-specific skills

Reader Profile
- Familiar with programming fundamentals
- May have used tools like GitHub Copilot or ChatGPT
- Struggling with:
    - Over-reliance on AI-generated code
    - Lack of confidence in correctness
    - Difficulty evaluating AI outputs


3. Unique Selling Proposition

This book differentiates itself by:

1. Shifting the focus from coding to thinking

    Instead of "how to prompt," it teaches:
    - Problem framing
    - Specification design
    - Evaluation strategies

2. Introducing "evaluation-driven development"

    A new paradigm where:
    - Testing and validation are central
    - AI-generated code is treated as hypotheses to verify

3. Bridging academia and practice
    - Grounded in software engineering theory
    - Applied to modern AI workflows

4. Tool-agnostic longevity

    Unlike books tied to tools like Cursor or Replit Ghostwriter, this book remains relevant as tools evolve.


---

Table of Contents

Each chapter corresponds to one week of teaching. A running course project threads through all 12 weeks, with each chapter contributing one milestone toward a complete AI-native software system.

---

Part I: SE Foundations (Weeks 1–4)

Chapter 1: Software Engineering in the Age of AI  [Week 1]
    The history and evolution of software engineering: SE 1.0 → SE 2.0
    SDLC models: Waterfall, Agile manifesto, Scrum, Kanban
    Scope creep, MoSCoW prioritization, user stories, story point estimation
    How AI is reshaping each phase of the SDLC
    Tutorial: Setting up your AI-assisted development environment
    Project milestone: Define your course project scope and team charter

Chapter 2: Requirements Engineering and Specification  [Week 2]
    Eliciting requirements: stakeholder interviews, workshops, observation
    Functional vs. non-functional requirements
    Epics, user stories, acceptance criteria, definition of done
    Quality attributes of good requirements: completeness, consistency, testability
    AI angle: Using LLMs to generate, critique, and refine requirements
    AI angle: Spec quality as a direct determinant of LLM output quality
    Tutorial: Writing and evaluating requirements with AI assistance
    Project milestone: Write the requirements specification for your course project

Chapter 3: Software Design, Architecture, and Patterns  [Week 3]
    UML diagrams: use case, class, sequence, component
    Architectural patterns: MVC, layered, event-driven, client-server, microservices, monolithic
    Design patterns (Gang of Four): singleton, factory, builder, observer, strategy, prototype
    Design principles: SOLID, DRY, composition over inheritance, Hollywood principle, encapsulate what varies
    Clean code: naming, structure, readability, and style
    AI angle: Generating architecture diagrams and code skeletons from specifications
    Tutorial: AI-assisted system design and code scaffolding
    Project milestone: Produce the design and architecture document for your course project

Chapter 4: Testing, Quality, and CI/CD  [Week 4]
    Code quality: linters, static analysis, style enforcement
    Black-box and white-box testing
    Unit testing, integration testing, code coverage
    Code review: manual and automated approaches
    CI/CD pipelines: concepts and YAML walkthrough (GitHub Actions)
    AI angle: AI-generated tests — how to trust but verify
    Tutorial: Python unit testing, code coverage, and CI/CD pipeline setup
    Project milestone: Set up automated testing and CI/CD for your course project

---

Part II: AI-Native Engineering (Weeks 5–12)

Chapter 5: The AI-Native Development Paradigm  [Week 5]
    What is AI-native engineering — distinct from AI-assisted development
    The AI-Native SDLC: Spec → Generate → Evaluate → Refine
    IDEs, copilots, and agents: GitHub Copilot, Cursor, Claude Code
    What is an AI coding agent? Tool use, planning, memory, and reflection
    Agentic protocols: MCP, A2A, and connectors
    Shifting the engineer's role: from coder to problem framer and evaluator
    Tutorial: Working with an AI coding agent end-to-end
    Project milestone: Re-examine your project spec through the AI-native lens

Chapter 6: Prompt Engineering and Context Design  [Week 6]
    Problem framing for AI systems: the art of specifying intent
    Prompt patterns: role prompting, chain-of-thought, few-shot, self-consistency
    Writing precise, testable specifications for AI generation
    Context engineering: files, memory, and what you give the model shapes what you get
    Common failure modes: vague prompts, under-specified constraints, hallucinated APIs
    Tutorial: Iterative prompt design for a realistic software feature
    Project milestone: Write AI-ready specifications for a core feature of your project

Chapter 7: Evaluation-Driven Development  [Week 7]
    AI-generated code as a hypothesis, not a solution
    Defining evaluation criteria before generating code
    Evaluation strategies: test suites, static analysis, LLM-as-judge, human review
    Measuring correctness, completeness, and hallucination
    Evaluation-driven development as a workflow: eval → generate → verify → refine
    Benchmarking AI tools on your own codebase
    Tutorial: Building an evaluation harness for AI-generated code
    Project milestone: Design and run an evaluation suite for your AI-generated project feature

Chapter 8: Agentic Systems and Multi-Agent Workflows  [Week 8]
    Agent architectures in depth: planning, tool use, memory, self-reflection
    Orchestration patterns: sequential, parallel, and hierarchical agents
    Multi-agent coordination: when agents collaborate and when they conflict
    MCP and A2A protocols: connecting agents to tools and external systems
    Designing agentic pipelines: decomposition, delegation, and error recovery
    When to use agents vs. direct generation
    Tutorial: Building a multi-step agentic workflow for a software task
    Project milestone: Introduce an agentic component into your course project

Chapter 9: AI Security Risks and Threat Modeling  [Week 9]
    Software security fundamentals: vulnerability, CVE, CWE, OWASP Top 10
    Common Python vulnerabilities and mitigations
    PII and credential detection: GitLeaks and secrets scanning
    AI-specific threats: prompt injection, data leakage, model inversion
    AI-generated vulnerabilities: how LLMs introduce insecure code
    Threat modeling for AI-enabled systems
    Tutorial: Vulnerability scanning in Python; prompt injection demonstration
    Project milestone: Conduct a security review of your course project

Chapter 10: Licenses, Ethics, and Responsible AI  [Week 10]
    Intellectual property and code ownership
    OSS licenses: proprietary, permissive (MIT, Apache), and copyleft (GPL)
    Copyright, fair use, and AI-generated code ownership ambiguity
    Responsible AI principles: fairness, transparency, accountability, privacy
    Bias in AI-generated code and outputs
    Organizational AI policies and governance frameworks
    Tutorial: License compliance audit; responsible AI checklist
    Project milestone: Apply a license and complete a responsible AI review for your project

Chapter 11: Developer Productivity and Team Practices  [Week 11]
    What "10x productivity" actually means in an AI-native context
    Measuring productivity: cycle time, defect rate, deployment frequency
    AI workflows for code review, documentation generation, and onboarding
    Team adoption patterns: pilot → practice → policy
    Managing AI tool risk: hallucination rate, over-reliance, skill atrophy
    Redefining engineering roles: from coder to AI engineer
    Tutorial: Integrating AI tools into a realistic team workflow
    Project milestone: Document your team's AI-native workflow and productivity findings

Chapter 12: The Future of AI-Native Engineering  [Week 12]
    Emerging paradigms: fully autonomous agents, self-healing systems, AI-designed architectures
    Skills that endure vs. skills that automate away
    Open research problems in AI software engineering
    Building a career in AI-native engineering
    Capstone: course project presentations and peer review
    Reflection: what it means to engineer in the age of agents

---

Appendices

Appendix A: Recommended Tools and Environments
Appendix B: Prompt Pattern Reference
Appendix C: Evaluation Rubrics for AI-Generated Code
Appendix D: OSS License Comparison Table
Appendix E: Further Reading and Research Directions
