# Appendix C: Applying These Practices in Other Languages

All code examples in this book use Python. The AI-native engineering practices — Spec → Generate → Evaluate → Refine, evaluation-driven development, agentic workflows — are language-agnostic. This appendix maps the key tools and patterns to three other common languages: TypeScript/Node.js, Go, and Java.

---

## C.1 TypeScript / Node.js

TypeScript is the dominant language for web frontends, Node.js backends, and full-stack JavaScript applications.

### Anthropic SDK

```bash
npm install @anthropic-ai/sdk
```

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic(); // reads ANTHROPIC_API_KEY from environment

const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Write a TypeScript function to filter tasks by status." }],
});

console.log(response.content[0].type === "text" ? response.content[0].text : "");
```

### Testing (Vitest / Jest)

Equivalent to `pytest`:

```bash
npm install -D vitest
```

```typescript
// task_service.test.ts
import { describe, it, expect } from "vitest";
import { filterTasks } from "./task_service";

describe("filterTasks", () => {
  it("returns only open tasks when status is 'open'", () => {
    const tasks = [
      { id: 1, title: "Task A", status: "open" },
      { id: 2, title: "Task B", status: "done" },
    ];
    const result = filterTasks(tasks, { status: "open" });
    expect(result).toHaveLength(1);
    expect(result[0].id).toBe(1);
  });

  it("returns empty array for empty input", () => {
    expect(filterTasks([], {})).toEqual([]);
  });
});
```

Run:
```bash
npx vitest run
npx vitest run --coverage  # equivalent to pytest-cov
```

### Code Quality

| Python tool | TypeScript equivalent |
|---|---|
| `ruff` | `eslint` + `prettier` |
| `mypy` | TypeScript compiler (`tsc --noEmit`) |
| `bandit` | `eslint-plugin-security` |
| `pytest-cov` | `vitest --coverage` / `nyc` |

```bash
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin prettier
npx tsc --noEmit  # type check without emitting files
```

### CI/CD (GitHub Actions)

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: npx tsc --noEmit
      - run: npx eslint .
      - run: npx vitest run --coverage
```

### Agent Tool Use in TypeScript

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { execSync } from "child_process";
import * as fs from "fs";

const client = new Anthropic();

const tools: Anthropic.Tool[] = [
  {
    name: "read_file",
    description: "Read the contents of a file",
    input_schema: {
      type: "object",
      properties: { path: { type: "string", description: "File path" } },
      required: ["path"],
    },
  },
  {
    name: "run_tests",
    description: "Run the test suite and return results",
    input_schema: { type: "object", properties: {}, required: [] },
  },
];

function executeTool(name: string, input: Record<string, string>): string {
  switch (name) {
    case "read_file":
      return fs.readFileSync(input.path, "utf-8");
    case "run_tests":
      try {
        return execSync("npx vitest run 2>&1").toString();
      } catch (e: any) {
        return e.stdout?.toString() ?? "Test runner failed";
      }
    default:
      return `Unknown tool: ${name}`;
  }
}
```

---

## C.2 Go

Go is widely used for cloud-native services, CLIs, and infrastructure tooling.

### Anthropic SDK

The official Go SDK:

```bash
go get github.com/anthropics/anthropic-sdk-go
```

```go
package main

import (
    "context"
    "fmt"
    "github.com/anthropics/anthropic-sdk-go"
)

func main() {
    client := anthropic.NewClient() // reads ANTHROPIC_API_KEY from environment

    message, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
        Model:     anthropic.F(anthropic.ModelClaudeSonnet4_6),
        MaxTokens: anthropic.F(int64(1024)),
        Messages: anthropic.F([]anthropic.MessageParam{
            anthropic.UserMessageParam(anthropic.NewTextBlock(
                "Write a Go function to filter tasks by status.",
            )),
        }),
    })
    if err != nil {
        panic(err)
    }
    fmt.Println(message.Content[0].Text)
}
```

### Testing

Go has a built-in test framework:

```go
// task_service_test.go
package tasks_test

import (
    "testing"
    "time"
    "github.com/yourorg/tasks"
)

func TestFilterTasks_ByStatus(t *testing.T) {
    allTasks := []tasks.Task{
        {ID: 1, Title: "Task A", Status: "open"},
        {ID: 2, Title: "Task B", Status: "done"},
    }
    result := tasks.FilterTasks(allTasks, tasks.Filter{Status: "open"})
    if len(result) != 1 {
        t.Errorf("expected 1 task, got %d", len(result))
    }
    if result[0].ID != 1 {
        t.Errorf("expected task ID 1, got %d", result[0].ID)
    }
}

func TestFilterTasks_EmptyInput(t *testing.T) {
    result := tasks.FilterTasks(nil, tasks.Filter{})
    if len(result) != 0 {
        t.Errorf("expected empty result, got %d tasks", len(result))
    }
}
```

```bash
go test ./...                    # run all tests
go test -cover ./...             # with coverage
go test -coverprofile=cov.out ./...
go tool cover -html=cov.out      # HTML report
```

### Code Quality

| Python tool | Go equivalent |
|---|---|
| `ruff` | `gofmt` (built-in) + `golangci-lint` |
| `mypy` | `go vet` (built-in) |
| `bandit` | `gosec` |
| `pytest-cov` | `go test -cover` |

```bash
# Install golangci-lint
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
golangci-lint run

# gosec security scanner
go install github.com/securego/gosec/v2/cmd/gosec@latest
gosec ./...
```

### CI/CD (GitHub Actions)

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
      - run: go vet ./...
      - run: go test -cover ./...
      - run: golangci-lint run
      - run: gosec ./...
```

---

## C.3 Java / Spring Boot

Java with Spring Boot is the dominant stack for enterprise backend services.

### Anthropic SDK

The official Java SDK:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-java</artifactId>
    <version>0.8.0</version>
</dependency>
```

```java
import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import com.anthropic.models.*;

public class Example {
    public static void main(String[] args) {
        AnthropicClient client = AnthropicOkHttpClient.fromEnv();
        // reads ANTHROPIC_API_KEY from environment

        Message message = client.messages().create(
            MessageCreateParams.builder()
                .model(Model.CLAUDE_SONNET_4_6)
                .maxTokens(1024)
                .addUserMessage("Write a Java method to filter tasks by status.")
                .build()
        );

        System.out.println(message.content().get(0).text().orElse(""));
    }
}
```

### Testing (JUnit 5 + Mockito)

```java
// TaskServiceTest.java
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;
import java.util.List;

class TaskServiceTest {

    @Test
    void filterTasks_byStatus_returnsMatchingTasks() {
        var tasks = List.of(
            new Task(1L, "Task A", "open", null),
            new Task(2L, "Task B", "done", null)
        );
        var service = new TaskService();

        var result = service.filterTasks(tasks, "open", null, null);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).getId()).isEqualTo(1L);
    }

    @Test
    void filterTasks_emptyList_returnsEmpty() {
        var service = new TaskService();
        assertThat(service.filterTasks(List.of(), null, null, null)).isEmpty();
    }
}
```

```bash
# Maven
mvn test
mvn verify  # includes integration tests

# Gradle
./gradlew test
./gradlew jacocoTestReport  # coverage
```

### Code Quality

| Python tool | Java equivalent |
|---|---|
| `ruff` | `checkstyle` + `spotless` |
| `mypy` | Java compiler (type safety is built-in) |
| `bandit` | `SpotBugs` + `find-sec-bugs` |
| `pytest-cov` | `JaCoCo` |

```xml
<!-- pom.xml plugins -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
</plugin>
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.3.0</version>
</plugin>
```

### CI/CD (GitHub Actions)

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: "21"
          distribution: "temurin"
      - run: mvn --no-transfer-progress verify
      - run: mvn spotbugs:check
```

---

## C.4 Language-Agnostic Principles

Regardless of language, the following AI-native practices transfer directly:

| Practice | How it transfers |
|---|---|
| **Write the specification first** | The specification format (function signature, behaviour, constraints, examples) is language-agnostic |
| **Define acceptance criteria before generation** | Test cases can be written in any test framework before the implementation exists |
| **Evaluate, don't just run** | Code review, security analysis, and edge case testing apply in all languages |
| **Use the debugging workflow** | The 5-step debugging workflow (Chapter 7, Section 7.7) maps directly to any language |
| **Treat AI output as a hypothesis** | The EDD mindset is independent of language or framework |

The specific tools differ; the workflow is the same.
