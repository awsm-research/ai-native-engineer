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
6. Use AI tools to assist with requirements generation and critique — and identify where AI assistance breaks down.

---

## 2.1 What Is Requirements Engineering?

Requirements engineering (RE) is the process of defining, documenting, and maintaining the requirements for a software system. It sits at the beginning of every software project, and its quality has an outsized effect on everything that follows: design decisions, implementation choices, testing strategies, and ultimately whether the system delivers value to its users.

The cost of fixing a requirements defect grows dramatically as development progresses. Research by Boehm and Papaccio ([1988](https://ieeexplore.ieee.org/document/12516)) found that defects discovered during requirements cost roughly 1–2 units to fix; the same defect discovered during testing costs 10–100 units; discovered in production, it can cost 100–1000 units. Getting requirements right early is one of the highest-return investments in software engineering.

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

### 2.2.4 Document Analysis

Existing documents — process manuals, legacy system specifications, regulatory guidelines, error logs, support tickets — are a rich source of requirements for systems that replace or augment existing functionality. Analysing support tickets reveals the most common failure modes of a current system; regulatory guidelines reveal mandatory constraints.

### 2.2.5 Prototyping

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

Key categories of non-functional requirements ([ISO/IEC 25010, 2011](https://www.iso.org/standard/35733.html)):

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

### 2.5.3 Tasks

Each user story is implemented through one or more *tasks* — specific technical actions. Tasks are not user-visible; they are engineering sub-steps.

**Example tasks for US-03 (assign a task to a team member):**

- Design the `POST /tasks/{id}/assign` API endpoint
- Implement the assignment logic and database update
- Write unit tests for the assignment service
- Write integration tests for the assignment endpoint
- Update API documentation

---

## 2.6 Acceptance Criteria

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

## 2.7 Definition of Done

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

## 2.8 AI-Assisted Requirements Engineering

AI tools are beginning to enter the requirements engineering process in meaningful ways. This section examines where they add value and where their limitations matter most.

### 2.8.1 Generating Draft Requirements

Given a brief system description, large language models can generate a first draft of requirements. This can accelerate early project stages by providing a concrete starting point for teams to critique and refine.

A capable LLM will produce a reasonable draft, but AI-generated requirements should never be accepted uncritically. Common issues include:

- **Missing domain-specific constraints**: The model cannot know about regulatory requirements, legacy integrations, or organisational policies.
- **Hallucinated specificity**: Numbers (response times, user limits, data retention periods) will be plausibly invented rather than sourced from actual stakeholders.
- **Missing edge cases**: Models generate obvious scenarios and may miss the edge cases that matter most in production.
- **Lack of traceability**: AI-generated requirements have no stakeholder source — a critical weakness when requirements are disputed.

### 2.8.2 Critiquing and Improving Requirements

A more reliable use of AI is as a *critic* rather than an *author*. Given a set of human-written requirements, an LLM can check for common quality issues:

```python
import anthropic

client = anthropic.Anthropic()

requirements_document = """
1. The system shall be fast.
2. Users can create tasks.
3. The system should be secure.
4. Tasks can be assigned to users.
5. The system shall send notifications.
"""

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": f"""Review the following requirements for quality issues.
For each requirement, identify if it is: ambiguous, incomplete, or unverifiable.
Suggest a rewritten version that addresses any issues.

Requirements:
{requirements_document}""",
        }
    ],
)

print(response.content[0].text)
```

This use of AI is lower risk because the human author retains ownership of the requirements and uses AI feedback as one input among several.

### 2.8.3 Generating User Stories from Interview Notes

Stakeholder interviews produce messy, unstructured notes. AI can help transform these into structured user stories:

```python
import anthropic

client = anthropic.Anthropic()

interview_notes = """
Spoke with Sarah (project manager, 12-person dev team).
Main frustrations: can't see at a glance who is overloaded,
tasks fall through the cracks when someone is sick,
no way to see what was completed last week for stand-ups.
Wants to reassign tasks quickly when priorities change.
Spends 30 mins every Monday just updating statuses in a spreadsheet.
"""

response = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": f"""You are a requirements engineer. Convert the following
stakeholder interview notes into well-structured user stories using the format:
"As a [role], I want [goal] so that [reason]."
Include 2–3 acceptance criteria per story in Gherkin format.
Only generate stories directly supported by the notes — do not invent requirements.

Interview notes:
{interview_notes}""",
        }
    ],
)

print(response.content[0].text)
```

The final instruction — "do not invent requirements" — is critical. Without it, models will helpfully generate plausible but unsupported requirements, which can mislead the team.

### 2.8.4 Where AI Cannot Replace Human Judgment

Despite these capabilities, requirements engineering remains fundamentally a human activity. AI cannot:

- **Resolve political conflicts** between stakeholders with competing interests
- **Understand organisational context** — why a constraint exists, who has authority to waive it, what previous attempts failed
- **Elicit tacit knowledge** — the workarounds, unspoken norms, and tribal knowledge that experienced users hold
- **Accept legal accountability** for requirements in regulated domains (healthcare, finance, aviation)
- **Build trust** with stakeholders — the relationship component that makes people willing to share their real problems

The engineer's role is not to be replaced by AI, but to use AI for routine structural tasks (formatting, checking, drafting) while investing their own time in the high-value, human-dependent work.

---

## 2.9 Tutorial: Requirements Review Pipeline

This tutorial demonstrates a practical requirements quality review pipeline using the Anthropic API.

### Setup

```bash
pip install anthropic python-dotenv
```

```bash
# .env
ANTHROPIC_API_KEY=your_key_here
```

### Requirements Review Pipeline

```python
# requirements_review.py
import os
import anthropic
from dotenv import load_dotenv

load_dotenv()

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])


def review_requirement(requirement: str) -> dict[str, str]:
    """Review a single requirement and return issues, improvement, and verdict."""
    response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=512,
        messages=[
            {
                "role": "user",
                "content": f"""Analyse this software requirement for quality issues.
Check for: ambiguity, lack of measurability, incompleteness, missing edge cases.

Requirement: "{requirement}"

Respond in this exact format:
ISSUES: [list specific issues, or "None" if well-formed]
IMPROVED: [rewritten version, or original if no issues]
VERDICT: [GOOD / NEEDS_IMPROVEMENT / POOR]""",
            }
        ],
    )

    text = response.content[0].text
    result: dict[str, str] = {}
    for line in text.strip().split("\n"):
        for key in ("ISSUES", "IMPROVED", "VERDICT"):
            if line.startswith(f"{key}:"):
                result[key.lower()] = line[len(key) + 1 :].strip()
    return result


def review_requirements_document(requirements: list[str]) -> None:
    """Review a list of requirements and print a quality report."""
    print("=" * 60)
    print("REQUIREMENTS QUALITY REVIEW")
    print("=" * 60)

    scores: dict[str, int] = {"GOOD": 0, "NEEDS_IMPROVEMENT": 0, "POOR": 0}

    for i, req in enumerate(requirements, 1):
        print(f"\n[{i}] {req}")
        result = review_requirement(req)
        verdict = result.get("verdict", "UNKNOWN")
        scores[verdict] = scores.get(verdict, 0) + 1

        print(f"    Verdict:  {verdict}")
        if verdict != "GOOD":
            print(f"    Issues:   {result.get('issues', 'N/A')}")
            print(f"    Improved: {result.get('improved', 'N/A')}")

    print("\n" + "=" * 60)
    print("SUMMARY")
    for label, count in scores.items():
        print(f"  {label:<22} {count}")
    print("=" * 60)


if __name__ == "__main__":
    sample_requirements = [
        "The system shall be fast.",
        "The system shall respond to 95% of API requests within 200ms "
        "under a load of 1,000 concurrent users.",
        "Users can create tasks.",
        "The system shall allow authenticated users to create tasks with a title "
        "(required, max 200 characters), description (optional, max 2,000 characters), "
        "due date (optional), and priority level (low, medium, high, critical).",
        "The system should be secure.",
        "All user passwords shall be stored as bcrypt hashes with a work factor of at least 12.",
    ]

    review_requirements_document(sample_requirements)
```

**Sample output:**

```
============================================================
REQUIREMENTS QUALITY REVIEW
============================================================

[1] The system shall be fast.
    Verdict:  POOR
    Issues:   Ambiguous and unverifiable — "fast" has no defined threshold
    Improved: The system shall respond to 95% of API requests within 200ms
              under a load of 1,000 concurrent users.

[2] The system shall respond to 95% of API requests within 200ms...
    Verdict:  GOOD

[3] Users can create tasks.
    Verdict:  NEEDS_IMPROVEMENT
    Issues:   Incomplete — missing subject (who), authentication context,
              and field constraints
    Improved: The system shall allow authenticated users to create tasks...
```

---

## 2.10 Project Milestone: Requirements Specification

### This Week's Deliverables

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

## Summary

Requirements engineering is the process of discovering, documenting, and validating what a system must do. Key takeaways:

- Elicitation techniques — interviews, workshops, observation, document analysis, and prototyping — each surface different types of requirements knowledge.
- Functional requirements describe *what* the system does; non-functional requirements describe *how well* it does it. Both must be measurable.
- Agile teams organise requirements as a hierarchy: epics decompose into user stories, which decompose into implementation tasks.
- Acceptance criteria in Gherkin format make requirements directly testable and reduce ambiguity.
- The Definition of Done is a shared quality gate that prevents "almost done" from becoming permanent.
- AI can accelerate drafting and critique of requirements, but cannot replace human judgment for elicitation, conflict resolution, and accountability.

---

## Review Questions

1. Why does fixing a requirements defect become more expensive the later it is discovered? What does this imply about investment in requirements engineering?
2. What is the difference between a functional and a non-functional requirement? Give two examples of each for a ride-sharing application.
3. Write three acceptance criteria in Gherkin format for: *"As a user, I want to delete a task so that I can remove tasks that are no longer relevant."*
4. What is the Definition of Done, and how does it differ from acceptance criteria for a specific user story?
5. You are eliciting requirements for a new hospital patient record system. Which techniques would you use and why? What are the specific challenges in this domain?
6. Describe two ways AI can assist with requirements engineering and two significant limitations of AI in this context.

---

## References

- Boehm, B., & Papaccio, P. (1988). Understanding and Controlling Software Costs. *IEEE Transactions on Software Engineering*, 14(10). [https://ieeexplore.ieee.org/document/12516](https://ieeexplore.ieee.org/document/12516)
- Brooks, F. P. (1975). *The Mythical Man-Month*. Addison-Wesley. [https://en.wikipedia.org/wiki/The_Mythical_Man-Month](https://en.wikipedia.org/wiki/The_Mythical_Man-Month)
- Beyer, H., & Holtzblatt, K. (1998). *Contextual Design: Defining Customer-Centered Systems*. Morgan Kaufmann. [https://www.elsevier.com/books/contextual-design/beyer/978-0-08-050612-3](https://www.elsevier.com/books/contextual-design/beyer/978-0-08-050612-3)
- Grady, R. B. (1992). *Practical Software Metrics for Project Management and Process Improvement*. Prentice Hall. [https://dl.acm.org/doi/10.1145/155360.155361](https://dl.acm.org/doi/10.1145/155360.155361)
- IEEE. (1998). *IEEE Recommended Practice for Software Requirements Specifications* (IEEE 830). [https://standards.ieee.org/ieee/830/1222/](https://standards.ieee.org/ieee/830/1222/)
- ISO/IEC 25010. (2011). *Systems and software Quality Requirements and Evaluation (SQuaRE)*. [https://www.iso.org/standard/35733.html](https://www.iso.org/standard/35733.html)
- ISO/IEC/IEEE 29148. (2018). *Systems and software engineering — Requirements engineering*. [https://www.iso.org/standard/72052.html](https://www.iso.org/standard/72052.html)
- Schwaber, K., & Sutherland, J. (2020). *The Scrum Guide*. [https://scrumguides.org/scrum-guide.html](https://scrumguides.org/scrum-guide.html)
- Wood, J., & Silver, D. (1995). *Joint Application Development*. Wiley. [https://www.wiley.com/en-us/Joint+Application+Development-p-9780471042075](https://www.wiley.com/en-us/Joint+Application+Development-p-9780471042075)
- Wynne, M., & Hellesøy, A. (2012). *The Cucumber Book*. Pragmatic Bookshelf. [https://pragprog.com/titles/hwcuc/the-cucumber-book/](https://pragprog.com/titles/hwcuc/the-cucumber-book/)
