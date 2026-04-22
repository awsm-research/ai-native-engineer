<div align="center">
  <img src="src/cover.svg" alt="The AI-Native Engineer: Building Software with Coding Agents" width="340"/>
</div>

<h1 align="center">The AI-Native Engineer<br><sub>Building Software with Coding Agents</sub></h1>

<p align="center">
  <strong>Kla Tantithamthavorn</strong><br>
  Associate Professor · Monash University
</p>

<p align="center">
  <a href="https://awsm-research.github.io/ai-native-engineer/"><strong>Read the Book →</strong></a>
</p>

---

## About

*The AI-Native Engineer* is a 12-week textbook and online course for the next generation of software engineers. It covers the full stack of modern software development — from classical software engineering foundations to prompt engineering, evaluation-driven development, agentic systems, security, ethics, and team productivity — through the lens of AI-native practice.

The book is built around a single running project (a Task Management API) that grows chapter by chapter, giving readers a concrete, end-to-end illustration of every concept.

## Contents

### Part I: SE Foundations
| Week | Chapter |
|------|---------|
| 1 | Software Engineering in the Age of AI |
| 2 | Requirements Engineering and Specification |
| 3 | Software Design, Architecture, and Patterns |
| 4 | Testing, Quality, and CI/CD |

### Part II: AI-Native Engineering
| Week | Chapter |
|------|---------|
| 5 | The AI-Native Development Paradigm |
| 6 | Prompt Engineering and Context Design |
| 7 | Evaluation-Driven Development |
| 8 | Agentic Systems and Multi-Agent Workflows |

### Part III: Governance and Practice
| Week | Chapter |
|------|---------|
| 9  | AI Security Risks and Threat Modeling |
| 10 | Licenses, Ethics, and Responsible AI |
| 11 | Developer Productivity and Team Practices |
| 12 | The Future of AI-Native Engineering |

### Appendices
- A: Recommended Tools and Environments
- B: Design Pattern Reference
- C: Applying These Practices in Other Languages
- D: Prompt Pattern Reference

## Key Concepts

- **Spec → Generate → Evaluate → Refine** — the AI-native SDLC
- **Evaluation-Driven Development (EDD)** — treat AI-generated code as a hypothesis
- **Agentic workflows** — ReAct agents, tool use, multi-agent orchestration
- **AI security** — prompt injection, STRIDE threat modelling, AI-generated vulnerabilities
- **Responsible AI** — licensing, GDPR, EU AI Act, accountability

## About the Author

**Kla Tantithamthavorn** is Associate Professor in Software Engineering at Monash University. His research spans AI-enabled software engineering, Explainable AI for SE, and LLM-based security testing. He has published 80+ papers with 8,600+ citations (h-index 44), is a World Top 2% Scientist (Stanford), and has secured over $2M in competitive research funding.

→ [chakkrit.com](https://chakkrit.com) · [Google Scholar](https://scholar.google.com.au/citations?user=idShgcoAAAAJ)

## Building Locally

Requires [mdBook](https://rust-lang.github.io/mdBook/) v0.5.2+.

```bash
# Install mdBook
cargo install mdbook

# Serve with live reload
mdbook serve

# Build static site
mdbook build
```

## License

Book content © Kla Tantithamthavorn, Monash University. All rights reserved.  
Code examples in the book are released under the [MIT License](LICENSE).
