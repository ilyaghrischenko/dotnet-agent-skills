# 🚀 .NET Agentic Workflows (AI Skills)

A collection of highly structured, deterministic AI skills (prompts) designed to turn your AI CLI (like OpenAI Codex) into an autonomous Senior .NET Engineer.

Unlike standard "write me code" prompts, these skills act as **state-machine pipelines**. They force the AI to read your project context, plan, branch, write code, run the compiler/tests to verify its own work, and create Pull Requests — all automatically.

## 📦 Available Skills

### 1. Vertical Slice Generator (`$vertical-slice`)
Scaffolds a complete, production-ready API endpoint using **Vertical Slice Architecture (VSA)** and Minimal APIs in ASP.NET Core.
* **Architecture:** Enforces single-file slices, strict `internal` access modifiers, and Fail-First early returns.
* **Rules:** Forbids MediatR and classic Repositories. Enforces direct `DbContext` usage and the Result pattern.
* **Autonomy:** Automatically creates a Git branch via GitHub MCP, compiles the code using `dotnet build /clp:ErrorsOnly`, self-corrects compiler errors (up to 3 times), commits, and opens a Pull Request.

### 2. Unit Tests Generator (`$unit-tests`)
Generates comprehensive, edge-case-heavy unit tests for selected C# code.
* **Stack:** `xUnit` and `FluentAssertions`.
* **Style:** Enforces `[Theory]/[InlineData]`, explicit typing (no `var`), and strict Arrange/Act/Assert structures.
* **Autonomy:** Runs `dotnet test` in the background. If tests fail, the AI reads the CLI output, fixes the test code, and retries automatically.

---

## 🛠️ Prerequisites

To get the full autonomous experience, you need:
1. **An AI CLI** (e.g., [OpenAI Codex CLI](https://github.com/openai/codex-cli)).
2. **GitHub MCP Server** enabled in your AI tool (required for `$vertical-slice` to create branches and PRs).
3. **.NET SDK** installed locally (required for the build and test execution loops).

---

## 🚀 Quick Start / Installation

### Step 1: Install the Skills
If you are using Codex CLI, you can install these skills directly from this repository:
1. Open your terminal and type `codex`.
2. Type `$` to open the skills menu.
3. Select **Skill Installer**.
4. Paste the raw URL of the skill you want to install (e.g., the raw link to `vertical-slice.md` inside the `skills/` folder).

### Step 2: Configure Your Project Context (`AGENTS.md`)
For the `$vertical-slice` skill to work perfectly, it needs to know your specific project architecture and GitHub details. 
1. Copy the `AGENTS-template.md` file from this repository and place it in the **root directory** of your target C# project.
2. Rename it to `AGENTS.md`.
3. Open it and fill in your specific details (like your exact `DbContext` class name, GitHub owner, and repo name).

### Step 3: Run the Agent
Navigate to your C# project in the terminal, start your AI CLI, and type:

**For a new feature:**
> `$vertical-slice create a new POST endpoint for Product creation. It should accept Name and Price, and optionally link a CategoryId.`

**For unit tests:**
> `$unit-tests generate tests for ProductHandler.cs`

Sit back and watch the agent branch, code, compile, and PR your feature.

---

## 🧠 Philosophy

These skills are built on the concept of **Constrained Generation**. By explicitly telling the LLM what *not* to do (e.g., "MediatR is FORBIDDEN", "Do NOT use AutoMapper"), and forcing it to run standard CLI commands (`dotnet build`) before committing, we bridge the gap between AI code generation and actual software engineering.