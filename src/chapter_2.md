# Chapter 2: Requirements Engineering and Specification

> *"The hardest single part of building a software system is deciding precisely what to build."*
> — Fred Brooks, *The Mythical Man-Month* (1975)

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain the purpose and phases of requirements engineering.
2. Apply multiple elicitation techniques to gather requirements from stakeholders.
3. Distinguish between functional and non-functional requirements and write both clearly.
4. Define epics, user stories, and acceptance criteria, and construct each for a realistic system.
5. Write a Definition of Done for a software team.

---

## 2.1 What Is Requirements Engineering?

Requirements engineering (RE) is the process of defining, documenting, and maintaining the requirements for a software system. It sits at the beginning of every software project, and its quality has an outsized effect on everything that follows: design decisions, implementation choices, testing strategies, and ultimately whether the system delivers value to its users.

The cost of fixing a requirements defect grows dramatically as development progresses. Research by Boehm, B. W., & Papaccio, P. N. ([1988](https://acolleoni.wordpress.com/wp-content/uploads/2012/02/boehm-papaccio-understanding-software-costs.pdf)) found that defects discovered during requirements cost roughly 1–2 units to fix; the same defect discovered during testing costs 10–100 units; discovered in production, it can cost 100–1000 units. Getting requirements right early is one of the highest-return investments in software engineering.

Requirements engineering comprises four main activities:

1. **Elicitation**: Discovering what stakeholders need
2. **Analysis**: Resolving conflicts, prioritising, and checking feasibility
3. **Specification**: Documenting requirements in a clear, agreed form
4. **Validation**: Confirming that documented requirements reflect actual stakeholder needs

These activities are not strictly sequential. In practice, they iterate: elicitation reveals conflicts that require analysis; analysis raises new questions that require further elicitation; validation reveals gaps that require re-specification.

---

## 2.2 Eliciting Requirements

Elicitation is the most people-intensive phase of requirements engineering. Requirements do not simply exist waiting to be discovered — they must be actively constructed through dialogue between engineers and stakeholders.

Stakeholders include anyone with a stake in the system:

- **Users**: People who interact with the system directly
- **Clients / customers**: People or organisations paying for or commissioning the system
- **Domain experts**: People with specialist knowledge the system must encode
- **Regulators**: Bodies whose rules constrain the system
- **Developers and operators**: People who build and run the system

### 2.2.1 Interviews

One-on-one or small group interviews are the most common elicitation technique. They allow engineers to explore individual stakeholders' perspectives in depth, ask follow-up questions, and observe non-verbal cues.

**Structured interviews** use a fixed set of questions, making responses comparable across stakeholders. **Semi-structured interviews** use a prepared guide but allow the interviewer to follow interesting threads. **Unstructured interviews** are open-ended conversations — useful early in a project when the problem space is poorly understood.

Effective interview questions:
- "Walk me through a typical day in your role. Where does [the system] fit in?"
- "What is the most frustrating part of the current process?"
- "What would success look like for you, six months after this system goes live?"
- "What happens when [edge case]? How do you handle that today?"

### 2.2.2 Workshops

Requirements workshops bring multiple stakeholders together in a structured session facilitated by a trained requirements engineer. They are particularly effective for resolving conflicts between stakeholder groups and building shared understanding quickly.

Joint Application Development (JAD) sessions ([Wood & Silver, 1995](https://www.wiley.com/en-us/Joint+Application+Development-p-9780471042075)) are a formalised workshop technique in which developers and users jointly define system requirements over 1–5 days. The intensity accelerates decision-making and builds stakeholder buy-in.

### 2.2.3 Observation and Ethnography

Sometimes the best way to understand requirements is to watch people do their work. *Contextual inquiry* ([Beyer & Holtzblatt, 1998](https://www.elsevier.com/books/contextual-design/beyer/978-0-08-050612-3)) involves working alongside users in their natural environment, observing what they actually do rather than what they say they do. This often surfaces tacit knowledge — practices and workarounds that users perform automatically and would never think to mention in an interview.

### 2.2.4 Personas

Once raw data has been gathered through interviews, workshops, and observation, engineers need a way to synthesise what they have learned into a shared understanding of who the system's users actually are. [*Personas*](https://uxpressia.com/blog/how-to-create-persona-guide-examples) are fictitious but research-grounded archetypes that represent the goals, behaviours, and frustrations of distinct user groups.

A persona is not a demographic profile — it is a behavioural model. A well-formed persona captures:

- **Goals**: what the user is trying to achieve (end goals, not task goals)
- **Behaviours**: how the user currently works, including workarounds and habits
- **Pain points**: where existing systems or processes fail them
- **Context**: environment, skill level, constraints (time pressure, device, connectivity)

**Example persona for a task management system:**

> **Jordan, the Overwhelmed Project Manager** — manages 3 concurrent projects across distributed teams. Switches between a laptop and phone throughout the day. Needs to reassign tasks quickly when team members go on leave. Frustrated by notification overload and by systems that require too many clicks to complete routine actions.

Personas serve two practical functions in requirements engineering. First, they act as a *reality check* during elicitation: "would Jordan actually use this feature?" surfaces requirements that look good on paper but serve no real user. Second, they anchor user stories — each story can be written from the perspective of a named persona, keeping abstract requirements grounded in observable behaviour.

**Limitation**: personas are only as good as the research behind them. Personas invented without observational or interview data tend to reflect developer assumptions rather than user reality, and can actively mislead the team.

### 2.2.5 Document Analysis

Existing documents — process manuals, legacy system specifications, regulatory guidelines, error logs, support tickets — are a rich source of requirements for systems that replace or augment existing functionality. Analysing support tickets reveals the most common failure modes of a current system; regulatory guidelines reveal mandatory constraints.

### 2.2.6 Prototyping

Showing stakeholders a low-fidelity prototype (wireframes, paper mockups, a clickable UI mockup) is often more effective than describing a system in words. Prototypes make abstract requirements concrete and frequently reveal misunderstandings that would otherwise persist until late in development.

---

## 2.3 Functional and Non-Functional Requirements

All requirements can be classified as either functional or non-functional.

### 2.3.1 Functional Requirements

Functional requirements describe *what* the system must do — specific behaviours, functions, or features. They define the interactions between the system and its environment.

**Format**: Functional requirements are often written as:
> The system shall [action] [object] [condition/qualifier].

**Examples for a task management system:**

- The system shall allow authenticated users to create tasks with a title, description, due date, and priority level.
- The system shall allow project managers to assign tasks to one or more team members.
- The system shall send an email notification to an assignee within 5 minutes of being assigned a task.
- The system shall allow users to filter tasks by status (open, in progress, completed, cancelled).

### 2.3.2 Non-Functional Requirements

Non-functional requirements (NFRs) describe *how* the system must behave — quality attributes that constrain the system's operation. They are sometimes called quality attributes or system properties.

NFRs are often harder to specify precisely than functional requirements, but they are equally important. A system that does the right thing slowly, insecurely, or unreliably fails its users just as surely as one that does the wrong thing.

Key categories of non-functional requirements ([ISO/IEC 25002:2024](https://www.iso.org/standard/78175.html)):

| Category | Description | Example |
|---|---|---|
| **Performance** | Speed and throughput | The API shall respond to 95% of requests within 200ms under a load of 1,000 concurrent users. |
| **Reliability** | Uptime and fault tolerance | The system shall achieve 99.9% uptime (≤8.7 hours downtime per year). |
| **Security** | Protection from threats | All data at rest shall be encrypted using AES-256. |
| **Scalability** | Ability to handle growth | The system shall support up to 100,000 active users without architectural changes. |
| **Usability** | Ease of use | A new user shall be able to create their first task within 3 minutes of registering. |
| **Maintainability** | Ease of change | All modules shall have unit test coverage of at least 80%. |
| **Portability** | Ability to run in different environments | The system shall run on any Linux environment with Python 3.11+. |
| **Compliance** | Adherence to regulations | The system shall comply with GDPR requirements for personal data storage and processing. |

**The danger of vague NFRs**: Non-functional requirements must be *measurable* to be useful. "The system should be fast" is not a requirement — it is a wish. "The API shall respond to 95% of requests within 200ms under a load of 1,000 concurrent users" is testable.

### 2.3.3 The FURPS+ Model

The FURPS+ model ([Grady, 1992](https://dl.acm.org/doi/10.1145/155360.155361)) provides a checklist for ensuring requirements coverage:

- **F**unctionality: Features and capabilities
- **U**sability: User interface and user experience
- **R**eliability: Availability, fault tolerance, recoverability
- **P**erformance: Speed, throughput, capacity
- **S**upportability: Testability, maintainability, portability
- **+**: Constraints (design, implementation, interface, physical)

---

## 2.4 Quality Attributes of Good Requirements

Individual requirements should satisfy the following quality criteria. The IEEE 830 standard ([IEEE, 1998](https://standards.ieee.org/ieee/830/1222/)) and its successor ISO/IEC/IEEE 29148 ([2018](https://www.iso.org/standard/72052.html)) are the canonical references.

| Attribute | Description | Bad Example | Good Example |
|---|---|---|---|
| **Correct** | Accurately represents stakeholder needs | — | Validated with stakeholders |
| **Unambiguous** | Has only one possible interpretation | "The system shall be user-friendly" | "A new user shall create their first task in under 3 minutes" |
| **Complete** | Covers all necessary conditions | "Users can log in" | "Users can log in with email/password; failed attempts are logged; accounts lock after 5 failures" |
| **Consistent** | Does not conflict with other requirements | Two requirements with contradictory session expiry rules | All session management requirements align |
| **Verifiable** | Can be tested or inspected | "The system shall be reliable" | "The system shall achieve 99.9% uptime" |
| **Traceable** | Can be linked to its source | Requirement with no stakeholder owner | Requirement tagged to specific stakeholder interview |
| **Prioritised** | Ranked by importance | No priority information | MoSCoW category assigned |

---

## 2.5 Epics, User Stories, and Work Items

In Agile teams, requirements are typically captured as a hierarchy of work items:

```
Epic
 └── Feature / Capability
      └── User Story
           └── Task (implementation subtask)
```

### 2.5.1 Epics

An *epic* is a large body of work that can be broken down into smaller stories. Epics represent significant chunks of functionality — typically too large to complete in a single sprint.

**Example epics for a task management system:**

- User Authentication and Authorisation
- Task Lifecycle Management (create, assign, update, complete)
- Notifications and Alerts
- Reporting and Analytics

### 2.5.2 User Stories

Each epic decomposes into user stories — small, independently deliverable increments of value.

**Epic: Task Lifecycle Management**

| ID | User Story |
|---|---|
| US-01 | As a user, I want to create a task with a title and description so that I can record work that needs to be done. |
| US-02 | As a user, I want to assign a due date to a task so that I can track deadlines. |
| US-03 | As a project manager, I want to assign a task to a team member so that responsibilities are clear. |
| US-04 | As a user, I want to mark a task as complete so that the team can see progress. |
| US-05 | As a user, I want to add comments to a task so that I can communicate context without leaving the tool. |

### 2.5.3 Story Points

*Story points* are a unit of measure for estimating the relative effort or complexity of user stories. They are intentionally abstract — they do not map directly to hours or days — encouraging teams to think about relative complexity rather than precise time estimates.

Teams typically use a modified Fibonacci sequence: **1, 2, 3, 5, 8, 13, 21**. The increasing gaps reflect growing uncertainty in estimating large, complex work.

**Planning Poker** is a common estimation technique ([Grenning, 2002](https://wingman-sw.com/articles/planning-poker)): each team member privately selects a card with their estimate; all cards are revealed simultaneously; significant discrepancies prompt discussion until the team reaches consensus.

Story points enable **velocity tracking** — the total points completed per sprint gives the team's *velocity*, which predicts future throughput and informs release planning.

### 2.5.4 Tasks

Each user story is implemented through one or more *tasks* — specific technical actions. Tasks are not user-visible; they are engineering sub-steps.

**Example tasks for US-03 (assign a task to a team member):**

- Design the `POST /tasks/{id}/assign` API endpoint
- Implement the assignment logic and database update
- Write unit tests for the assignment service
- Write integration tests for the assignment endpoint
- Update API documentation

---

## 2.6 Prioritisation: The MoSCoW Framework

Once user stories are written, the team must decide which to build first. The **MoSCoW framework** ([Clegg & Barker, 1994](https://www.dsdm.org/)) provides a shared vocabulary for this:

| Category | Meaning | Guideline |
|---|---|---|
| **M**ust Have | Non-negotiable; the system cannot launch without these | ~60% of effort |
| **S**hould Have | Important but not vital; workarounds exist if omitted | ~20% of effort |
| **C**ould Have | Nice to have; included only if time permits | ~20% of effort |
| **W**on't Have | Explicitly excluded from this release | Documented, not built |

The "Won't Have" category is often the most valuable: it makes explicit what is being deliberately deferred, turning unspoken assumptions into shared agreements.

**Example — a task management application:**

| Feature | MoSCoW |
|---|---|
| Create, read, update, delete tasks | Must Have |
| Assign tasks to team members | Must Have |
| Email notifications on task assignment | Should Have |
| Drag-and-drop task reordering | Could Have |
| Integration with Slack | Won't Have (this release) |

---

## 2.7 Scope Creep

Even with user stories and prioritisation in place, projects face a persistent risk: *scope creep* — the gradual, uncontrolled expansion of scope beyond its original boundaries. It is one of the most common causes of project failure ([PMI, 2021](https://www.pmi.org/learning/library/scope-creep-causes-effects-solutions-6181)).

Scope creep happens when:

- Stakeholders request new features after the project has started
- Requirements are poorly defined, leaving room for interpretation
- The team adds features without formal approval
- External factors force new work mid-project

MoSCoW directly addresses this: by explicitly documenting what is *Won't Have*, teams create a shared boundary that makes adding new scope a visible, deliberate decision rather than a gradual drift. Combined with regular backlog grooming and formal change control, user stories, prioritisation, and scope discipline together form the core of agile requirements management.

---

## 2.8 Acceptance Criteria

*Acceptance criteria* define the specific conditions that must be satisfied for a user story to be considered done. They bridge requirements and testing: each acceptance criterion should be directly testable.

The most common format is **Gherkin** — a structured natural language syntax used by the Cucumber testing framework ([Wynne & Hellesøy, 2012](https://pragprog.com/titles/hwcuc/the-cucumber-book/)):

```gherkin
Given [some initial context]
When  [an action occurs]
Then  [an observable outcome]
```

**Example — US-03: Assign a task to a team member**

```gherkin
Scenario: Successfully assigning a task
  Given I am logged in as a project manager
  And a task with ID "123" exists in my project
  And a team member "alice@example.com" exists in my project
  When I send POST /tasks/123/assign with body {"assignee": "alice@example.com"}
  Then the response status code is 200
  And the task's assignee field is updated to "alice@example.com"
  And alice receives an email notification within 5 minutes

Scenario: Attempting to assign to a non-member
  Given I am logged in as a project manager
  And a task with ID "123" exists in my project
  When I send POST /tasks/123/assign with body {"assignee": "nonmember@example.com"}
  Then the response status code is 400
  And the response body contains {"error": "User is not a member of this project"}

Scenario: Attempting to assign without permission
  Given I am logged in as a regular user (not a project manager)
  When I send POST /tasks/123/assign with body {"assignee": "alice@example.com"}
  Then the response status code is 403
  And the response body contains {"error": "Insufficient permissions"}
```

Well-written acceptance criteria cover:
- The **happy path** (the successful scenario)
- **Error cases** (invalid input, unauthorised access)
- **Edge cases** (boundary conditions, concurrent operations)

---

## 2.9 Definition of Done

The *Definition of Done* (DoD) is a shared agreement about what "complete" means for any piece of work. It is a quality gate: a story is not done until it satisfies every item on the DoD checklist ([Schwaber & Sutherland, 2020](https://scrumguides.org/scrum-guide.html)).

**Example Definition of Done for the course project:**

- [ ] All acceptance criteria pass
- [ ] Unit tests written and passing (minimum 80% coverage for new code)
- [ ] Integration tests written and passing
- [ ] Code reviewed by at least one other team member
- [ ] Linter and type checker pass with no errors
- [ ] API documentation updated (if applicable)
- [ ] No new security vulnerabilities introduced (verified by automated scan)
- [ ] Deployed to the staging environment and manually tested

A DoD prevents "almost done" from becoming a permanent state and makes quality expectations explicit and consistent across the team.

---

## 2.10 Tutorial Activity: "AI As Your Client"

**Concepts covered:** Elicitation techniques, requirements specification, quality attributes, conflict resolution, scope management

**Format:** Individual or pairs | **Duration:** 2 hours | **Tool:** [Microsoft Copilot](https://copilot.microsoft.com/) (chat interface)

---

### Background

In this activity, [Microsoft Copilot](https://copilot.microsoft.com/) plays the role of your client. You will conduct a requirements elicitation interview, produce specification artefacts, audit their quality, resolve stakeholder conflicts, and respond to scope creep — mirroring the full requirements engineering lifecycle from §2.1.

---

### Phase 1 — Elicitation Interview (25 min)

Prime Copilot with the following system prompt at the start of your conversation:

> *"You are Jordan, the founder of a small retail business. You want to build a new online shopping application to sell your products directly to customers, replacing your current manual order-taking process via phone and email. You have opinions and preferences but are not technical. You will only answer questions I ask — do not volunteer information I haven't asked for. If I ask a vague question, give a vague answer. Stay in character for the entire conversation."*

![An example UI when using Microsoft Copilot as an AI Client](images/chapter2-copilot-ui.png)
*An example UI when using Microsoft Copilot as an AI Client*

Conduct a semi-structured interview with Jordan using the elicitation techniques from §2.2.1. Log every question and Copilot's response in a worksheet.

**Requirements:**
- Ask at least **8 questions**
- Cover at least **3 stakeholder concerns** (e.g., product browsing, checkout and payment, order management)
- Use at least **one follow-up question** that digs deeper into a vague answer

> **Tip:** Copilot will not give you everything you need unless you ask the right questions. Vague questions will produce vague answers — just as in real stakeholder interviews.

---

### Phase 2 — Produce Artefacts (20 min)

From your interview transcript, produce the following:

1. **4 functional requirements** in "The system shall…" format
2. **2 non-functional requirements** — each must be measurable (apply the test from §2.3.2)
3. **2 user stories** in "As a [role]…" format
4. **A MoSCoW table** with at least 5 features prioritised

---

### Phase 3 — Acceptance Criteria and Definition of Done (25 min)

#### Part A — Acceptance Criteria (15 min)

For each of your 2 user stories from Phase 2, write acceptance criteria in Gherkin format (§2.8). Each user story must have:

- **1 happy path scenario** — the successful case
- **1 error or edge case scenario** — invalid input, missing data, or unauthorised access

**Example structure:**

```gherkin
Scenario: [descriptive name]
  Given [initial context]
  When  [action taken]
  Then  [observable outcome]
```

> **Check:** Can each scenario be tested without ambiguity? If a tester cannot determine pass or fail from the scenario alone, rewrite it.

#### Part B — Definition of Done (10 min)

Write a **Definition of Done** (§2.9) for your online shopping application project. It must include at least **6 items** covering:

- Functional correctness (acceptance criteria)
- Code quality (testing, review)
- Non-functional validation (performance, security)
- Deployment and documentation

Compare your DoD with another pair. Identify one item they included that you missed, and add it with a one-sentence justification for why it belongs.

---

### Phase 4 — Requirements Quality Audit (20 min)

Swap your requirements artefacts with another pair. Audit each other's requirements against the IEEE quality criteria from §2.4:

| Requirement | Correct | Unambiguous | Complete | Consistent | Verifiable | Traceable | Prioritised |
|---|---|---|---|---|---|---|---|
| FR-01 | | | | | | | |
| FR-02 | | | | | | | |
| FR-03 | | | | | | | |
| FR-04 | | | | | | | |
| NFR-01 | | | | | | | |
| NFR-02 | | | | | | | |

Mark each cell ✓ (satisfies the attribute), ✗ (fails), or ? (unclear). For every ✗, write a one-sentence explanation of the flaw and a corrected version of the requirement.

---

### Phase 5 — Conflict Injection (20 min)

Start a new Copilot conversation with this persona:

> *"You are Sam, a frequent online shopper in their late 20s. You shop on your phone and expect a fast, frictionless experience — ideally guest checkout with no account required. You find long forms and mandatory registration frustrating. Stay in character."*

Interview Sam for 10 minutes, then:

1. Identify **at least 2 conflicts** between Jordan's requirements and Sam's
2. Document each conflict explicitly — which requirement from each stakeholder, and why they are incompatible
3. Propose a written resolution for each: either a requirement that satisfies both stakeholders, or a justified MoSCoW trade-off that explicitly records what was deferred and why

---

### Phase 6 — Scope Creep Simulation (15 min)

Your instructor will send the following message, simulating a client email received mid-project:

> *"Hi team — Jordan here. I forgot to mention, we'd also love the app to integrate with our Instagram and Facebook pages so customers can buy directly from our social media posts. Also, can it support a loyalty points system? Oh, and my business partner just asked if we could add a B2B wholesale portal for bulk orders."*

For each new request:

1. Classify it using MoSCoW — does it change any existing priorities?
2. Determine whether it is **scope creep** or a legitimate missed requirement, and justify your decision
3. Write a one-paragraph **change response** to Jordan that acknowledges all three requests, documents what is accepted or deferred, and explains why

---

### Phase 7 — Reflection in EdStem (15 min)

Answer the following questions individually in writing:

1. After the quality audit, which quality attribute (§2.4) was hardest to satisfy in your requirements — and why?
2. Could the conflict between Jordan and Sam have been discovered from a single stakeholder interview? What does this tell you about elicitation breadth?
3. Which of Jordan's scope creep requests was hardest to classify — the social media integration, loyalty points, or B2B portal — and why?
4. What does Copilot-as-client *cannot* replicate compared to a real stakeholder interview? Think about §2.2.3 (observation and tacit knowledge).
5. Where in this activity did Copilot add genuine value — and where did it fall short?

---
