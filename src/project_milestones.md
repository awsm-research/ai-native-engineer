# Project Milestones

This file contains all project milestone deliverables for the 12-week course. Each milestone corresponds to one chapter and builds on the previous week's work.

---

## Week 1 — Define Your Course Project
*(Chapter 1: Software Engineering Fundamentals and Processes)*

### Project Brief

Throughout this course you will build a **Task Management API** — a backend system that allows users to create projects, manage tasks, assign them to team members, and track progress. This is a deliberately familiar problem domain: the focus is on *how* you build it using AI-native practices, not on inventing a novel application.

The project grows across 12 weeks:
- **Weeks 1–4**: Requirements, design, tests, and CI/CD for the core API
- **Weeks 5–8**: AI-native development of features using agents and evaluation
- **Weeks 9–12**: Security hardening, ethics review, productivity analysis, and reflection

### Deliverables

1. **Team charter** (if in a team): Names, agreed roles, and working norms.
2. **Scope statement**: One paragraph describing what your system *will* do; one paragraph explicitly describing what it *will not* do.
3. **MoSCoW list**: At least 10 features categorised as Must/Should/Could/Won't.
4. **Development environment**: A GitHub repository with virtual environment, linter, pre-commit hooks, and at least one passing test.

Submit a `README.md` in the root of your repository containing the team charter, scope statement, and MoSCoW list.

---

## Week 2 — Requirements Specification
*(Chapter 2: Requirements Engineering and Specification)*

### Deliverables

Produce a requirements specification document for your Task Management API. Include:

1. **Stakeholder list**: At least 3 stakeholder roles and their primary concerns.
2. **Functional requirements**: At least 15 requirements in "The system shall..." format, covering your Must Have items.
3. **Non-functional requirements**: At least 6 NFRs covering performance, security, reliability, and maintainability — all measurable.
4. **Epic and story map**: At least 3 epics, each decomposed into 3–5 user stories.
5. **Acceptance criteria**: Full Gherkin acceptance criteria for at least 5 user stories.
6. **Definition of Done**: Your team's agreed DoD checklist.

### Bonus

Run your requirements through the review pipeline from the tutorial. Include the output as an appendix to your specification. Note which requirements were flagged, whether you agreed with the AI's assessment, and what you changed (or chose not to change) — and why.

---

## Week 3 — Design and Architecture Document
*(Chapter 3: Software Design, Architecture, and Patterns)*

### Deliverables

Produce a design document for your Task Management API. Include:

1. **Architecture decision**: Which architectural pattern (layered, MVC, etc.) will you use, and why? What alternatives did you consider?
2. **Component diagram**: A diagram showing the major components and their dependencies (draw.io, Mermaid, or ASCII art all acceptable).
3. **Domain class diagram**: A class diagram showing your core domain entities (Task, Project, User, etc.) and their relationships.
4. **At least 2 sequence diagrams**: One for creating a task, one for assigning a task.
5. **Design patterns**: Identify at least 2 GoF or architectural patterns you will use, with a brief justification for each.
6. **SOLID review**: For each of the 5 SOLID principles, write one sentence explaining how your design applies it.

### Bonus

Use the AI design assistant from the tutorial to generate an initial component design. Include the AI output in an appendix, and document what you changed and why.

---

## Week 4 — Testing and CI/CD
*(Chapter 4: Testing, Quality, and CI/CD)*

### Deliverables

1. **Unit tests**: At least 20 unit tests covering your core domain logic, using pytest. Tests must include:
   - Happy path tests
   - Boundary value tests
   - Error case tests
   - At least one parametrised test using `@pytest.mark.parametrize`
2. **Integration tests**: At least 5 integration tests using your `InMemoryTaskRepository`.
3. **Code coverage**: At least 80% statement coverage on `src/`.
4. **CI/CD pipeline**: A GitHub Actions workflow that runs lint, type check, tests, and security scan on every push.
5. **Branch protection**: The `main` branch requires the CI workflow to pass before merging.

### Bonus

Generate tests for one of your service functions using an AI assistant. Review the generated tests critically, document which tests you accepted, which you modified, and which you rejected — and explain why in each case.

---

## Week 5 — Re-Examine Through the AI-Native Lens
*(Chapter 5: The AI-Native Development Paradigm)*

### Deliverables

Return to your course project specification from Weeks 1–2 and produce:

1. **AI-native specification rewrite**: Rewrite the acceptance criteria for 3 of your user stories in the "AI-native" format — context, inputs/outputs, constraints, and examples — suitable for use as a generation specification.

2. **Generate-evaluate log**: For one of your rewritten specifications, run the Spec → Generate → Evaluate → Refine cycle. Document:
   - The specification you used
   - The generated output (or a summary)
   - Your evaluation findings (what was correct, what was wrong)
   - What you changed in the specification and why
   - The final output after refinement

3. **Reflection**: In 200–300 words, reflect on how the AI-native SDLC differs from the traditional SDLC. What required more effort than you expected? What required less?

---

## Week 6 — AI-Ready Specifications
*(Chapter 6: Prompt Engineering and Context Design)*

### Deliverables

Using the specification template from Chapter 6, write AI-ready specifications for **3 core features** of your Task Management API. For each specification:

1. **Full specification**: All seven sections (Task, Signature, Context, Behaviour, Error Handling, Constraints, Examples)
2. **Generation run**: Generate an implementation using the specification
3. **Evaluation log**: Record which examples and constraints the generated code satisfies and which it violates
4. **Refinement**: Make at least one refinement to the specification and regenerate; document what changed and why

Submit the specification documents alongside your evaluation logs.

---

## Week 7 — Evaluation Suite
*(Chapter 7: Evaluation-Driven Development)*

### Deliverables

1. **Evaluation criteria**: For one complete feature of your Task Management API (e.g., the task assignment flow), define evaluation criteria across all four strategies: automated tests, static analysis, LLM-as-judge rubric, and a human review checklist.

2. **Evaluation harness**: Implement and run the evaluation harness from Chapter 7 on at least one AI-generated function from Week 6.

3. **Evaluation report**: Document:
   - Which evaluation strategy caught which issues
   - The hallucination types (if any) you observed
   - How you refined the specification in response to evaluation failures
   - Your final pass/fail verdict for each generated function

4. **Reflection**: In 200–300 words, compare EDD to TDD. What are the similarities? What is unique to the AI-generated code context?

---

## Week 8 — Agentic Feature Development
*(Chapter 8: Agentic Systems and Multi-Agent Workflows)*

### Deliverables

Introduce an agentic component into your course project:

1. **Agent design**: Choose one feature from your project backlog that requires:
   - Reading multiple existing files to understand context
   - Generating new code that integrates with existing code
   - Running tests to verify the result

   Document your agent design: goal description, tools available, stopping condition.

2. **Implementation**: Implement and run the coding agent from Chapter 8 to implement your chosen feature.

3. **Evaluation**: Compare the agentic approach to the direct generation approach from Week 6:
   - How many steps did the agent take?
   - Did the agent make any mistakes? How did it recover?
   - How did the final output quality compare?

4. **Safety review**: Review your agent implementation against the failure modes in Chapter 8. Which failure modes are relevant to your implementation, and what mitigations did you apply?

---

## Week 9 — Security Review
*(Chapter 9: AI Security Risks and Threat Modeling)*

### Deliverables

1. **Vulnerability audit**: Run Bandit and GitLeaks on your course project repository. Document every finding — do not suppress findings without understanding them.

2. **STRIDE threat model**: Complete a STRIDE analysis for two endpoints in your Task Management API. For each threat, document: the scenario, severity (high/medium/low), and your mitigation.

3. **Security specification addendum**: Add explicit security constraints to the AI specifications you wrote in Week 6. For each constraint you add, explain which vulnerability class it prevents.

4. **Security review of AI-generated code**: Run the security review function from the tutorial on at least one AI-generated function from your project. Document the findings and any changes you made.

---

## Week 10 — License and Responsible AI Review
*(Chapter 10: Licenses, Ethics, and Responsible AI)*

### Deliverables

1. **License audit**: Run `pip-licenses` on your project. Document every dependency licence. Identify any copyleft dependencies and assess whether your project's use triggers copyleft obligations.

2. **AI-generated code documentation**: Add a section to your project's `README.md` disclosing which parts of the codebase are substantially AI-generated and which AI tools were used.

3. **Responsible AI assessment**: Complete the responsible AI checklist from Chapter 10 for your project. For any unchecked items, document why (accepted risk, not applicable, or a remediation plan).

4. **Copyright decision**: Choose an appropriate licence for your course project and add a `LICENSE` file to your repository. Justify your choice in a comment in the project's `README.md`.

---

## Week 11 — Productivity Analysis and AI Workflow Documentation
*(Chapter 11: Developer Productivity and Team Practices)*

### Deliverables

1. **Productivity baseline**: Using the DORA framework, document your team's baseline metrics for the course project: deployment frequency, lead time, change failure rate (from your CI data), and estimated MTTR.

2. **AI workflow documentation**: Write a 1-page "AI workflow guide" for a hypothetical new team member joining your project. Cover:
   - Which tasks use AI assistance and which do not
   - The review process for AI-generated code
   - The specification format used (from Chapter 6)
   - Any tools configured (pre-commit hooks, CI checks, etc.)

3. **Productivity reflection**: In 300–400 words, reflect on your experience using AI tools over the past 7 weeks:
   - Which tasks produced the most reliable AI output?
   - Where did you spend more time than expected on AI-related activities?
   - What skill, if any, do you feel you exercised less because of AI assistance?
   - Based on your experience, how would you advise a colleague new to AI-native development?
