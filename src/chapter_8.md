# Chapter 8: Agentic Systems and Multi-Agent Workflows

> *"The key to building reliable agentic systems is not building smarter agents — it is building clearer interfaces between them."*

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Describe the key components of an agent architecture: planning, tool use, memory, and reflection.
2. Compare sequential, parallel, and hierarchical orchestration patterns.
3. Explain how multi-agent systems decompose complex tasks and coordinate results.
4. Identify the failure modes specific to agentic systems and how to mitigate them.
5. Design a simple multi-agent workflow for a realistic software task.
6. Implement a basic tool-using agent with the Anthropic API.

---

## 8.1 From Chatbots to Agents

A chatbot responds to each message in isolation — it has no persistent state, no ability to take actions beyond generating text, and no mechanism to verify that its responses are correct. Useful, but limited.

An *agent* differs in three fundamental ways:

1. **It can take actions**: Not just generate text, but execute code, read files, call APIs, and modify state
2. **It has a goal**: It is directed toward an outcome, not just a response
3. **It persists across steps**: It maintains state and adapts its behaviour based on what it has observed

The term "agentic AI" covers a spectrum from simple tool-using chatbots to fully autonomous systems that operate for hours or days with minimal human intervention. In this chapter, we focus on the engineering principles that apply across this spectrum.

---

## 8.2 Agent Architecture

A software agent consists of four components working together: a planning module, a tool execution layer, a memory system, and (optionally) a reflection mechanism.

### 8.2.1 Planning

Planning is the process of decomposing a high-level goal into a sequence of concrete steps. A good planning module:

- Identifies what information it needs before it can act
- Orders steps to minimise wasted work (don't implement before specifying)
- Detects when a plan needs to be revised (when a tool call returns unexpected results)
- Knows when the goal has been achieved

**ReAct: Reasoning and Acting**

The ReAct framework ([Yao et al., 2022](https://arxiv.org/abs/2210.03629)) interleaves reasoning (generating a thought about what to do next) with acting (executing a tool call):

```
Thought: I need to understand the current task assignment logic before modifying it.
Action: read_file("src/assignment_service.py")
Observation: [file contents returned]

Thought: The current code doesn't validate that the assignee is a project member.
         I should add that check. First, let me see the project membership model.
Action: read_file("src/project_service.py")
Observation: [file contents returned]

Thought: I now have enough context. I'll write the validation logic.
Action: edit_file("src/assignment_service.py", ...)
Observation: File updated successfully.

Thought: I should run the tests to verify the change is correct.
Action: run_command("pytest tests/test_assignment_service.py -v")
Observation: 5 passed, 0 failed.

Thought: All tests pass. The task is complete.
```

This interleaved pattern is significantly more reliable than generating the entire plan upfront, because each action's result informs the next step.

### 8.2.2 Tool Use

Tools are the interface between the agent's language model and the external world. In the Anthropic API, tools are defined as JSON schemas describing the function name, parameters, and expected behaviour ([Anthropic, 2024](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)):

```python
import anthropic
import json
import os

client = anthropic.Anthropic()

tools = [
    {
        "name": "read_file",
        "description": "Read the contents of a file at the given path.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The file path to read, relative to the project root.",
                }
            },
            "required": ["path"],
        },
    },
    {
        "name": "run_tests",
        "description": "Run the pytest test suite and return the output.",
        "input_schema": {
            "type": "object",
            "properties": {
                "test_path": {
                    "type": "string",
                    "description": "The test file or directory to run.",
                }
            },
            "required": ["test_path"],
        },
    },
]


def execute_tool(name: str, inputs: dict) -> str:
    """Execute a tool call and return the result as a string."""
    if name == "read_file":
        path = inputs["path"]
        try:
            with open(path) as f:
                return f.read()
        except FileNotFoundError:
            return f"Error: File not found: {path}"

    elif name == "run_tests":
        import subprocess
        result = subprocess.run(
            ["pytest", inputs["test_path"], "-v"],
            capture_output=True,
            text=True,
        )
        return result.stdout + result.stderr

    return f"Error: Unknown tool: {name}"
```

The agent loop — calling the model, executing tools, and feeding results back — looks like this:

```python
def run_agent(goal: str, max_steps: int = 10) -> str:
    """Run an agent loop until the goal is achieved or max_steps is reached."""
    messages = [{"role": "user", "content": goal}]

    for step in range(max_steps):
        response = client.messages.create(
            model="claude-opus-4-7",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )

        # If the model is done reasoning and has no tool calls, return its response
        if response.stop_reason == "end_turn":
            return response.content[0].text

        # Process any tool calls
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })

        # Append the assistant's response and tool results to the message history
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

    return "Max steps reached without completing the goal."
```

### 8.2.3 Memory

Agents need memory to operate effectively across long tasks and multiple sessions.

**In-context memory** is the conversation history maintained in the `messages` list. It is the most reliable form of memory — the model can attend to anything in context — but it is limited by the context window size and grows more expensive as the conversation lengthens.

**External memory** stores information outside the model's context, in files, databases, or vector stores. The agent reads from external memory when it needs information and writes to it when it wants to persist state.

**Episodic summarisation** compresses earlier parts of a long conversation into a summary that replaces the original detail. This allows the agent to operate on tasks longer than the context window without losing all history.

```python
def summarise_conversation(messages: list[dict]) -> str:
    """Summarise a conversation history to free up context space."""
    conversation_text = "\n".join(
        f"{m['role'].upper()}: {m['content']}" for m in messages
        if isinstance(m["content"], str)
    )

    response = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=512,
        messages=[
            {
                "role": "user",
                "content": f"Summarise the key decisions, findings, and current state "
                           f"from this agent session in 200 words:\n\n{conversation_text}",
            }
        ],
    )
    return response.content[0].text
```

### 8.2.4 Reflection

Reflection is the agent's ability to evaluate its own outputs and identify potential errors before proceeding. A reflecting agent might:

- Check that generated code compiles before writing it to disk
- Verify that a plan makes sense before executing it
- Detect when it is stuck in a loop and escalate to a human

Reflection adds latency but significantly reduces failure rates for complex tasks. It is optional for simple tool-use tasks but essential for long-running autonomous agents.

---

## 8.3 Orchestration Patterns

When a task is too complex for a single agent, it can be decomposed across multiple agents. Three orchestration patterns cover most use cases.

### 8.3.1 Sequential Orchestration

Steps execute in order, with each agent's output feeding the next agent's input.

```
User Goal
    │
    ▼
[Requirements Agent] → refined specification
    │
    ▼
[Coding Agent] → implementation
    │
    ▼
[Testing Agent] → test results
    │
    ▼
[Review Agent] → final verdict
```

**Strengths**: Simple to reason about; easy to debug (each step is independent).
**Weaknesses**: Total latency is the sum of all steps; no opportunity for parallelism; a failure in one step blocks all subsequent steps.

**Suitable for**: Tasks with clear dependencies where each step depends on the previous output.

### 8.3.2 Parallel Orchestration

Independent sub-tasks execute simultaneously, and their results are combined.

```
User Goal
    │
    ├──> [Unit Test Agent] ──────────┐
    ├──> [Integration Test Agent] ──>│ [Combiner Agent] → Final Report
    └──> [Security Scan Agent] ──────┘
```

**Strengths**: Reduces total latency by running independent tasks concurrently.
**Weaknesses**: Requires a combiner to synthesise results; sub-tasks must be truly independent.

**Suitable for**: Tasks that can be decomposed into independent work streams (e.g., running different evaluation strategies in parallel).

```python
import asyncio
import anthropic

async_client = anthropic.AsyncAnthropic()


async def run_agent_async(goal: str, agent_name: str) -> tuple[str, str]:
    """Run a single agent asynchronously."""
    response = await async_client.messages.create(
        model="claude-opus-4-7",
        max_tokens=1024,
        messages=[{"role": "user", "content": goal}],
    )
    return agent_name, response.content[0].text


async def run_parallel_evaluation(code: str, spec: str) -> dict[str, str]:
    """Run multiple evaluation agents in parallel."""
    tasks = [
        run_agent_async(
            f"Review this code for security vulnerabilities:\n{code}",
            "security",
        ),
        run_agent_async(
            f"Review this code for performance issues:\n{code}",
            "performance",
        ),
        run_agent_async(
            f"Review this code against the specification:\nSPEC:\n{spec}\nCODE:\n{code}",
            "correctness",
        ),
    ]

    results = await asyncio.gather(*tasks)
    return dict(results)


# Usage
results = asyncio.run(
    run_parallel_evaluation(code="...", spec="...")
)
```

### 8.3.3 Hierarchical Orchestration

A *coordinator* agent decomposes the goal and delegates sub-tasks to *worker* agents. Workers report back; the coordinator synthesises results and decides on next steps.

```
[Coordinator Agent]
    │
    ├── "Implement the task assignment feature"
    │       └──> [Coding Agent]
    │               └── Result: implementation files
    │
    ├── "Write tests for the assignment feature"
    │       └──> [Testing Agent]
    │               └── Result: test files + coverage report
    │
    └── "Review the implementation and tests"
            └──> [Review Agent]
                    └── Result: review findings
```

**Strengths**: Naturally handles complex, interdependent tasks; coordinator can adapt the plan based on worker results.
**Weaknesses**: Most complex to implement and debug; coordinator can become a bottleneck; error propagation is harder to trace.

**Suitable for**: Large, multi-phase software engineering tasks where the full plan cannot be specified in advance.

---

## 8.4 Failure Modes in Agentic Systems

Agentic systems introduce failure modes that do not exist in single-turn AI interactions. Understanding them is essential for building reliable systems.

### 8.4.1 Compounding Errors

In a multi-step agent, an error in step N can corrupt all subsequent steps. Unlike a single LLM call where a bad response can simply be discarded, an agent that writes incorrect code to disk, runs tests against it, and then tries to interpret the failures may compound the initial error across multiple steps.

**Mitigation**: Add checkpoints — points where the agent's output is validated before proceeding. At a minimum, run static analysis and syntax checking before executing generated code.

### 8.4.2 Hallucinated Plans

An agent may generate a plausible-looking plan that contains steps that are impossible, contradictory, or simply wrong. It will then attempt to execute these steps, failing in confusing ways.

**Mitigation**: Require the agent to state its plan before executing and provide a mechanism for human approval on plans above a certain complexity or risk level.

### 8.4.3 Tool Call Loops

An agent can get stuck calling the same tool repeatedly when it fails to make progress — for example, running tests that fail, trying to fix the code, running the tests again, failing again, and repeating.

**Mitigation**: Implement loop detection (count identical tool calls in the recent history); set a maximum step budget; escalate to a human when the budget is exhausted.

### 8.4.4 Scope Creep

A well-intentioned agent may "help" by making changes beyond the specified scope — refactoring surrounding code, updating dependencies, or adding features that were not requested.

**Mitigation**: Be explicit in goal descriptions: "Implement X. Do not modify any files outside of src/X.py and tests/test_X.py." Use file system permissions to restrict what the agent can write.

### 8.4.5 Prompt Injection via Tool Results

When an agent reads external data (files, web pages, API responses) and incorporates it into its context, a malicious payload in that data can attempt to hijack the agent's behaviour — instructing it to ignore its original goal and take a different action.

**Mitigation**: This is covered in depth in Chapter 9 (AI Security). For now: treat tool results as untrusted input; do not let tool results override system-level instructions.

---

## 8.5 When to Use Agents vs. Direct Generation

Agents add complexity and latency. They are not always the right tool.

| Use Direct Generation | Use an Agent |
|---|---|
| Single, well-specified function | Multi-file feature spanning multiple components |
| No need to read existing code | Must understand and integrate with existing code |
| Outcome is easily verified in one step | Outcome requires iterative testing and refinement |
| Low risk (reversible, reviewed before use) | High complexity (plan must adapt to findings) |
| Latency is important | Thoroughness is more important than speed |

A useful heuristic: if completing the task requires more than three tool calls, or if the agent needs to adapt its plan based on what it discovers, use an agent. Otherwise, use direct generation with a well-structured specification.

---

## 8.6 Tutorial: An Agent That Implements a Task Management Feature

This tutorial builds a coding agent that implements the `get_overdue_tasks` function from the course project, writes the implementation to the correct file, runs the existing tests, and iterates until they pass — without human intervention at each step.

**Prerequisites:** The Task Management API project from the course has `src/task_service.py` (with the `Task` dataclass) and `tests/test_overdue.py` (with failing tests for the yet-to-be-implemented function).

```python
# agent_implement_feature.py
"""
Coding agent: implements get_overdue_tasks in the Task Management API.
Run from the project root: python agent_implement_feature.py
"""
import subprocess
import os
import anthropic

client = anthropic.Anthropic()

# ── Tool definitions ──────────────────────────────────────────────────────────

tools = [
    {
        "name": "read_file",
        "description": "Read the contents of a source file.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "File path relative to project root."}
            },
            "required": ["path"],
        },
    },
    {
        "name": "write_file",
        "description": "Write (overwrite) a source file with new content.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "content": {"type": "string"},
            },
            "required": ["path", "content"],
        },
    },
    {
        "name": "run_tests",
        "description": "Run pytest on a specific test file and return the output.",
        "input_schema": {
            "type": "object",
            "properties": {
                "test_path": {"type": "string", "description": "Test file or directory."}
            },
            "required": ["test_path"],
        },
    },
]


def execute_tool(name: str, inputs: dict) -> str:
    """Dispatch a tool call and return its result as a string."""
    if name == "read_file":
        path = inputs["path"]
        if not os.path.exists(path):
            return f"ERROR: File not found: {path}"
        with open(path) as f:
            return f.read()

    elif name == "write_file":
        path = inputs["path"]
        # Safety: only allow writes within src/ and tests/
        if not (path.startswith("src/") or path.startswith("tests/")):
            return f"ERROR: Writes outside src/ and tests/ are not permitted. Got: {path}"
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, "w") as f:
            f.write(inputs["content"])
        return f"Written {len(inputs['content'])} characters to {path}"

    elif name == "run_tests":
        result = subprocess.run(
            ["pytest", inputs["test_path"], "-v", "--tb=short"],
            capture_output=True,
            text=True,
        )
        output = result.stdout + result.stderr
        return output[:4000]  # Truncate to stay within context budget

    return f"ERROR: Unknown tool: {name}"


# ── Agent loop ────────────────────────────────────────────────────────────────

GOAL = """
You are implementing a new function in an existing Python project.

YOUR TASK:
Implement `get_overdue_tasks` in src/task_service.py.

The function specification:
  def get_overdue_tasks(
      tasks: list[Task],
      today: date | None = None,
  ) -> list[Task]:

  - Returns tasks where due_date < today AND status not in ("completed", "cancelled")
  - If today is None, uses date.today()
  - Tasks with no due_date are never overdue
  - Returns empty list if no matches
  - Result sorted by due_date ascending; ties broken by priority ascending (1=highest)
  - Does NOT modify the input list
  - Raises TypeError("tasks must be a list") if tasks is not a list

PROCEDURE:
1. Read src/task_service.py to understand the existing Task dataclass and imports
2. Read tests/test_overdue.py to understand exactly what the tests expect
3. Add get_overdue_tasks to src/task_service.py (do not remove existing code)
4. Run tests/test_overdue.py
5. If tests fail, diagnose the failures and revise the implementation
6. Repeat until all tests pass, then report done

CONSTRAINTS:
- Only modify src/task_service.py
- Do not modify any test files
- Use parameterised queries; no shell=True; no external imports
"""


def run_feature_agent(max_steps: int = 10) -> None:
    messages: list[dict] = [{"role": "user", "content": GOAL}]

    for step in range(1, max_steps + 1):
        print(f"\n{'='*50}")
        print(f"Step {step}/{max_steps}")

        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )

        # Model has finished — no more tool calls
        if response.stop_reason == "end_turn":
            for block in response.content:
                if hasattr(block, "text"):
                    print(f"\nAgent: {block.text}")
            print("\n✓ Agent completed.")
            return

        # Process tool calls
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"  → {block.name}({list(block.input.keys())})")
                result = execute_tool(block.name, block.input)
                # Show a brief preview
                preview = result[:150].replace("\n", " ")
                print(f"     {preview}{'...' if len(result) > 150 else ''}")
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })

        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

    print("\n✗ Max steps reached without completing the task.")


if __name__ == "__main__":
    run_feature_agent()
```

### What to Observe

When you run this agent, watch for:

1. **Step 1–2**: The agent reads existing files before writing anything — demonstrating the "investigate before acting" planning pattern.
2. **First write**: The agent adds the function without touching existing code (respects the constraint).
3. **First test run**: If tests fail, note how the agent reads the failure output and identifies which assertion failed.
4. **Iteration**: Watch how the agent revises its implementation based on specific test failures, not by rewriting everything.
5. **Termination**: The agent declares completion only after all tests pass — not after the first write.

If the agent gets stuck in a loop (running tests, failing, making the same fix), that is a **loop failure mode** (Section 8.4.3). The `max_steps` guard prevents an infinite loop.
```

---

## 8.7 Project Milestone: Agentic Feature Development

### This Week's Deliverables

Introduce an agentic component into your course project:

1. **Agent design**: Choose one feature from your project backlog that requires:
   - Reading multiple existing files to understand context
   - Generating new code that integrates with existing code
   - Running tests to verify the result
   
   Document your agent design: goal description, tools available, stopping condition.

2. **Implementation**: Implement and run the coding agent from Section 8.6 to implement your chosen feature.

3. **Evaluation**: Compare the agentic approach to the direct generation approach from Week 6:
   - How many steps did the agent take?
   - Did the agent make any mistakes? How did it recover?
   - How did the final output quality compare?

4. **Safety review**: Review your agent implementation against the failure modes in Section 8.4. Which failure modes are relevant to your implementation, and what mitigations did you apply?

---

> **Common Mistakes**
>
> **Giving the agent too many tools.** More tools means more surface area for the agent to misuse. Start with the minimum set of tools that the task requires. Add tools only when the agent demonstrably cannot complete a step without them.
>
> **No step limit.** An agent without a `max_steps` guard will loop indefinitely when it encounters a task it cannot complete. Always set a step limit and handle the case where the agent exhausts it without completing the task.
>
> **Tool results without error handling.** Tool calls fail. The file doesn't exist, the test runner times out, the API returns 429. If your agent does not handle tool errors gracefully, one failure will cascade into an unrecoverable state. Every tool result needs an error path.
>
> **Trusting the agent's self-report.** An agent that reports "task complete" has not necessarily completed the task correctly. Verify its work with an independent check — run the tests yourself, read the file it claims to have written, call the endpoint it claims to have implemented.

---

## Summary

Agentic systems extend language models with the ability to plan, use tools, maintain memory, and reflect on their outputs. Key takeaways:

- An agent architecture consists of: a planning module (ReAct), a tool execution layer, a memory system, and optional reflection.
- Three orchestration patterns handle different task structures: sequential (for dependent steps), parallel (for independent sub-tasks), and hierarchical (for complex, adaptive tasks).
- Agentic systems introduce unique failure modes: compounding errors, hallucinated plans, tool call loops, scope creep, and prompt injection via tool results.
- Use direct generation for well-specified, single-step tasks; use agents for multi-step tasks that require adaptation to discovered information.
- The Anthropic tool use API enables building tool-using agents with explicit tool definitions and a message-based interaction loop.

---

## Review Questions

1. What are the four components of an agent architecture? Briefly describe the role of each.
2. Explain the ReAct framework. Why does interleaving reasoning and acting improve reliability over generating a full plan upfront?
3. Compare sequential and parallel orchestration. Give an example of a software task that suits each pattern.
4. What is "compounding error" in an agentic system? Give a concrete example in the context of software development.
5. You are building an agent that reads issues from a bug tracker, searches the codebase for the relevant code, and generates a fix. Which failure modes from Section 8.4 are most relevant, and what mitigations would you apply?
6. When would you choose direct generation over an agent? Give two criteria.

---

## References

- Anthropic. (2024). Tool Use — Claude API Documentation. [https://docs.anthropic.com/en/docs/build-with-claude/tool-use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- Chase, H. (2022). LangChain: Building applications with LLMs through composability. [https://github.com/langchain-ai/langchain](https://github.com/langchain-ai/langchain)
- Google. (2025). A2A: Agent-to-Agent Protocol. [https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- Anthropic. (2024). Model Context Protocol. [https://modelcontextprotocol.io/](https://modelcontextprotocol.io/)
- Russell, S., & Norvig, P. (2020). *Artificial Intelligence: A Modern Approach* (4th ed.). [https://aima.cs.berkeley.edu/](https://aima.cs.berkeley.edu/)
- Wang, L., et al. (2023). A Survey on Large Language Model based Autonomous Agents. *arXiv*. [https://arxiv.org/abs/2308.11432](https://arxiv.org/abs/2308.11432)
- Yao, S., et al. (2022). ReAct: Synergizing Reasoning and Acting in Language Models. *arXiv*. [https://arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629)
- Xi, Z., et al. (2023). The Rise and Potential of Large Language Model Based Agents: A Survey. *arXiv*. [https://arxiv.org/abs/2309.07864](https://arxiv.org/abs/2309.07864)
