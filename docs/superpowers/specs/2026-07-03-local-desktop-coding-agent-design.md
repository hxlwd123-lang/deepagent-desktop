# DeepAgent Desktop Design

## Purpose

Build DeepAgent Desktop, a local-first desktop AI coding tool similar in working style to Claude Code Desktop. The product uses Deep Agents as the Python agent harness, a multi-agent architecture, and harness-engineering principles so users can safely delegate coding work while retaining control over local files, shell commands, model choice, and approvals.

## Product Direction

The first version is a local desktop application. It opens a local project, lets the user start a coding session, asks the agent to plan before acting, shows every meaningful tool call in a visible timeline, gates risky actions through approvals, and produces reviewable diffs before applying code changes.

The first version prioritizes a reliable single-user local workflow over cloud execution, team collaboration, remote sandboxes, or plugin marketplaces.

## Product Decisions

These decisions define the first implementation milestone:

- Product name: DeepAgent Desktop.
- Repository: `hxlwd123-lang/deepagent-desktop`.
- First development and test platform: Windows.
- Desktop stack: Tauri, React, TypeScript, and npm.
- Python runtime stack: Python, uv, FastAPI, WebSocket, SQLite, and Deep Agents.
- First model provider: Alibaba Cloud Model Studio / Bailian / DashScope for Qwen models.
- Model API strategy: use Alibaba's OpenAI-compatible interface first, while keeping the model registry extensible for future providers.
- MCP: usable MCP configuration is required in the first working build.
- Git commits: the agent may create git commits in the MVP, but only through explicit approval.
- UI style: dense, calm, engineering-focused desktop workbench.
- First distribution goal: developer-run Windows app; packaged installer is a post-MVP packaging milestone after the local workflow is stable.

## Research Basis

Deep Agents is the core harness because it provides planning, file-system-oriented context management, subagent spawning, long-term memory, and model-neutral configuration. Deep Agents Code validates the coding-agent direction: it supports tool-calling LLMs, provider and model switching, skills, memory, and approval controls.

Comparable open-source products suggest the same product center:

- Cline: Plan/Act flow, explicit approval, file and terminal access, model neutrality.
- Roo Code: modes for coding, architecture, debugging, and custom workflows.
- OpenHands: agent control center, sandboxed execution, model routing, and observable agent work.
- Aider/OpenCode/Goose/Continue: local developer control, diff-centric editing, model provider flexibility, and terminal-oriented workflows.

Harness engineering is treated as the product core. The model makes decisions, while the harness owns tools, permissions, state, verification, context, logging, and recovery.

## Architecture

```text
Tauri Desktop Shell
  React UI
    Project/session navigation
    Chat, Plan, and Act timeline
    Diff viewer
    Terminal output
    Approval queue
    Subagent status
    Model settings

  Tauri Rust Backend
    Starts and stops the Python runtime
    Bridges UI requests to the runtime
    Stores local secrets through the system keychain
    Enforces local process boundaries

  Python Agent Runtime
    FastAPI HTTP API
    WebSocket event stream
    Deep Agents create_deep_agent runtime
    Coding tools
    Model router
    Approval policy engine
    Session persistence
```

The Python runtime runs as a local sidecar process. Tauri owns the desktop shell and lifecycle management. React owns the user interface. The runtime owns agent sessions, tools, policy decisions, event streaming, and persisted state.

## Desktop UI

The desktop UI is a three-column workbench rather than a chat-only interface.

```text
Left: projects, git branch, sessions, model entry
Center: user conversation, plan, act timeline, final summary
Right: files, diff, terminal, subagents, approvals
```

The center pane is the main activity feed. It shows user prompts, generated plans, todo updates, tool calls, approvals, subagent reports, terminal output summaries, and final results.

The right pane is task-focused and switchable:

- Files: files the agent read or changed.
- Diff: proposed and applied code changes.
- Terminal: command output and exit status.
- Agents: planner, reviewer, and debugger state.
- Approvals: pending write, shell, install, network, or git actions.

The UI should feel like an engineering control surface: dense, scannable, and calm. It should avoid a marketing landing page, decorative hero sections, and card-heavy promotional layouts.

## Workflow

The default workflow is Plan/Act.

1. The user opens a local project.
2. The user creates a session and enters a goal.
3. The main agent inspects safe context and produces a plan.
4. The user approves the plan.
5. The agent enters Act mode.
6. Safe read tools run automatically.
7. Risky tools emit approval requests.
8. The UI displays diffs, commands, and reasons before approval.
9. The runtime resumes the agent after approve, reject, or modified approve.
10. The reviewer subagent checks the final diff and test evidence.
11. The UI shows a final summary, changed files, test results, and remaining risks.

## Python Runtime Structure

```text
python-runtime/
  app/
    server.py
    sessions.py
    events.py
    agent/
      factory.py
      prompts.py
      subagents.py
      state.py
    models/
      registry.py
      router.py
      credentials.py
    tools/
      workspace.py
      patch.py
      shell.py
      git.py
      approvals.py
      mcp.py
```

Responsibilities:

- `server.py`: FastAPI app, health checks, HTTP endpoints, WebSocket stream.
- `sessions.py`: session lifecycle, cancellation, recovery.
- `events.py`: typed event payloads for UI and persistence.
- `agent/factory.py`: Deep Agents initialization.
- `agent/prompts.py`: coding-agent system prompt and behavior rules.
- `agent/subagents.py`: planner, reviewer, debugger configuration.
- `agent/state.py`: state serialization and restore.
- `models/registry.py`: provider and model definitions.
- `models/router.py`: model selection and session-level switching.
- `models/credentials.py`: runtime access to keychain-backed credentials.
- `tools/workspace.py`: list, read, and search workspace files.
- `tools/patch.py`: create, preview, and apply diffs.
- `tools/shell.py`: execute approved commands with timeout and cwd controls.
- `tools/git.py`: status, diff, branch, and optional commit helpers.
- `tools/approvals.py`: tool risk classification and approval suspension.
- `tools/mcp.py`: project and user MCP server loading.

## Agents

The first version includes one main coding agent and three built-in subagents.

- Main coding agent: owns the user session, plan, tool loop, and final answer.
- Planner: decomposes larger work and identifies risky execution steps.
- Reviewer: checks final diffs, missing tests, likely regressions, and unsafe edits.
- Debugger: investigates failed tests, command failures, and runtime errors.

Subagents are not allowed to write files directly in the first version. They return findings to the main agent, which then proposes changes through the normal approval path.

## Tool Policy

Tools are grouped by risk.

Safe read tools run without approval:

- list workspace files
- read files inside the workspace
- search text inside the workspace
- inspect git status and git diff

Approval tools require explicit user action:

- write files
- apply patch
- run shell command
- install dependencies
- start long-running processes
- access network resources
- create git commits

Blocked actions are denied in the first version:

- write outside the selected workspace
- delete arbitrary paths outside generated temporary directories
- read known secret files without explicit user selection
- leak configured API keys into prompts, logs, terminal output, or session records
- run background services without visible lifecycle controls
- bypass approval by nesting risky behavior inside another tool

Each approval request includes action type, target path or command, agent reason, risk level, expected effect, and a preview. File edits show a diff before application.

## Model Configuration

Model configuration has three layers:

```text
Global provider configuration
  Project default model
    Session temporary model
```

The first version supports:

- Alibaba Cloud Model Studio / Bailian / DashScope for Qwen models.
- Alibaba OpenAI-compatible Chat Completions as the default integration path.
- A generic OpenAI-compatible provider profile only where needed to support Alibaba endpoint configuration cleanly.

Post-MVP versions may add OpenAI, Anthropic, Gemini, OpenRouter, Ollama, and other local or remote providers through the same registry and router interfaces.

Each model entry stores:

```json
{
  "provider": "alibaba",
  "model": "qwen-coder",
  "baseUrl": "https://dashscope.aliyuncs.com/compatible-mode/v1",
  "apiKeyRef": "alibaba_default",
  "temperature": 0.2,
  "maxTokens": 64000,
  "reasoningEffort": "medium",
  "supportsTools": true
}
```

Alibaba configuration should allow the user to override `baseUrl` because Model Studio may require region-specific, workspace-specific, or coding-plan endpoints.

API keys are stored through the operating system keychain via Tauri. The Python runtime receives only the selected key value or a short-lived environment reference for the active provider.

Session-level model switches are allowed. Each switch is recorded as a timeline event with previous model, new model, reason, and timestamp. The runtime summarizes prior context before switching when needed.

## Persistence

Use local SQLite for non-secret state:

- projects
- sessions
- messages
- todos
- tool events
- approval decisions
- generated diffs
- model switch history
- final summaries

Use OS keychain for secrets:

- provider API keys
- optional MCP credentials
- local endpoint tokens

Session records are designed for recovery and audit. The user should be able to reopen a previous session and understand what the agent saw, what it changed, what commands it ran, and what remains unresolved.

## Error Handling

Runtime errors are surfaced as timeline events.

Model errors:

- invalid credentials
- rate limits
- context length failures
- malformed tool calls

Tool errors:

- missing files
- patch conflicts
- command timeouts
- permission failures

Approval errors:

- rejected action
- modified action
- expired approval

Subagent errors:

- failed subtask
- incomplete investigation
- conflicting recommendation

Runtime errors:

- Python sidecar crash
- WebSocket disconnect
- failed session restore

The product should explain where work stopped, what actions already happened, and whether the session can resume.

## Testing Strategy

Python tests:

- model registry and router behavior
- credential reference handling
- policy classification
- workspace path boundaries
- patch generation and application
- shell timeout and cwd handling
- session state serialization

Python integration tests:

- agent tool call to approval to patch application
- rejected approval resumes agent with rejection result
- failed shell command triggers debugger workflow
- reviewer receives final diff and returns findings

Frontend tests:

- timeline event rendering
- approval card rendering and actions
- model settings form
- diff viewer behavior
- session restore state

End-to-end smoke test:

- open a sample project
- ask agent to change a small function
- approve patch
- run tests through approved shell command
- show final summary and reviewer result

Security tests:

- deny writes outside workspace
- deny dangerous shell patterns by policy
- ensure API keys do not appear in persisted logs
- ensure rejected actions are not executed

## MVP Scope

The first version includes:

- local project open
- Alibaba Cloud Model Studio / Bailian / DashScope model configuration
- coding session creation
- Plan/Act workflow
- file read and search tools
- diff generation and patch application
- shell command approval and execution
- approved git commit creation
- planner, reviewer, and debugger subagent status
- event timeline
- local SQLite persistence
- usable MCP configuration for user and project servers
- settings page

The first version excludes:

- OpenAI, Anthropic, Gemini, OpenRouter, and Ollama provider implementations
- cloud execution
- team collaboration
- remote sandbox providers
- automatic GitHub PR creation
- plugin marketplace
- multi-project parallel agent swarms
- browser automation
- full IDE editor replacement

## Acceptance Criteria

The MVP is complete when a user can:

1. Launch the desktop app.
2. Open a local repository.
3. Configure at least one model provider.
4. Start a coding session.
5. Review and approve an agent plan.
6. See file reads, searches, diffs, approvals, and terminal commands in the timeline.
7. Approve a patch.
8. Approve a test command.
9. Receive a final summary with reviewer findings.
10. Approve agent-created git commits.
11. Configure and load an MCP server.
12. Reopen the session after restarting the app.

## Resolved Implementation Inputs

The user has resolved the required first-milestone decisions:

1. Product name: DeepAgent Desktop.
2. First target operating system: Windows.
3. Frontend package manager: npm.
4. Python dependency manager: uv.
5. Initial model provider: Alibaba Cloud Model Studio / Bailian / DashScope.
6. Local Ollama support: deferred after MVP.
7. MCP configuration: required in the first working build.
8. Git commit creation: allowed in MVP through explicit approval.
9. Visual style: dense engineering control surface.
10. Distribution goal: developer-run Windows build first, packaged installer as a post-MVP packaging milestone.
