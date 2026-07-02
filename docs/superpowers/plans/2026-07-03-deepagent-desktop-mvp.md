# DeepAgent Desktop MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first developer-run Windows MVP of DeepAgent Desktop: a Tauri + React desktop shell connected to a Python Deep Agents runtime with Alibaba Qwen model configuration, MCP loading, approval-gated workspace tools, git commit support, event persistence, and a usable engineering workbench UI.

**Architecture:** Tauri owns the desktop window, Windows lifecycle, keychain bridge, and sidecar process management. React renders the three-column workbench and consumes a typed event stream. A Python FastAPI runtime owns sessions, Deep Agents orchestration, model routing, MCP servers, workspace tools, approvals, SQLite persistence, and WebSocket events.

**Tech Stack:** Windows, Tauri 2, Rust, React, TypeScript, Vite, npm, Python 3.11+, uv, FastAPI, Pydantic, SQLite, pytest, Deep Agents, LangChain OpenAI-compatible client, Alibaba Cloud Model Studio / Bailian / DashScope Qwen models, MCP Python SDK.

---

## Reference Documents

- Product spec: `docs/superpowers/specs/2026-07-03-local-desktop-coding-agent-design.md`
- Deep Agents Python docs: `https://docs.langchain.com/oss/python/deepagents/overview`
- Tauri sidecar docs: `https://v2.tauri.app/develop/sidecar/`
- Alibaba OpenAI-compatible DashScope docs: `https://help.aliyun.com/zh/model-studio/compatibility-of-openai-with-dashscope`
- Qwen-Coder API docs: `https://help.aliyun.com/en/model-studio/qwen-coder`
- MCP Python SDK: `https://github.com/modelcontextprotocol/python-sdk`

## Target File Structure

```text
.
  package.json
  package-lock.json
  index.html
  tsconfig.json
  tsconfig.node.json
  vite.config.ts
  src/
    App.tsx
    main.tsx
    styles.css
    api/
      runtime.ts
      types.ts
    components/
      ActivityTimeline.tsx
      ApprovalQueue.tsx
      DiffPanel.tsx
      ModelSettings.tsx
      ProjectSidebar.tsx
      RightRail.tsx
      TerminalPanel.tsx
  src-tauri/
    Cargo.toml
    build.rs
    tauri.conf.json
    capabilities/
      default.json
    src/
      main.rs
  python-runtime/
    pyproject.toml
    uv.lock
    README.md
    app/
      __init__.py
      server.py
      sessions.py
      events.py
      persistence.py
      agent/
        __init__.py
        factory.py
        prompts.py
        subagents.py
        state.py
      models/
        __init__.py
        registry.py
        router.py
        credentials.py
      tools/
        __init__.py
        approvals.py
        git.py
        mcp.py
        patch.py
        shell.py
        workspace.py
    tests/
      conftest.py
      test_approvals.py
      test_events.py
      test_git.py
      test_mcp_config.py
      test_models.py
      test_patch.py
      test_persistence.py
      test_server.py
      test_shell.py
      test_workspace.py
  samples/
    calculator-py/
      pyproject.toml
      app.py
      test_app.py
  docs/
    development.md
```

## Task 1: Scaffold Desktop and Runtime Workspaces

**Files:**
- Create: `package.json`
- Create: `index.html`
- Create: `tsconfig.json`
- Create: `tsconfig.node.json`
- Create: `vite.config.ts`
- Create: `src/main.tsx`
- Create: `src/App.tsx`
- Create: `src/styles.css`
- Create: `src-tauri/Cargo.toml`
- Create: `src-tauri/build.rs`
- Create: `src-tauri/tauri.conf.json`
- Create: `src-tauri/capabilities/default.json`
- Create: `src-tauri/src/main.rs`
- Create: `python-runtime/pyproject.toml`
- Create: `python-runtime/app/__init__.py`
- Create: `python-runtime/app/server.py`
- Create: `python-runtime/tests/test_server.py`

- [ ] **Step 1: Create the npm workspace files**

Write `package.json`:

```json
{
  "name": "deepagent-desktop",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite --host 127.0.0.1",
    "build": "tsc && vite build",
    "preview": "vite preview --host 127.0.0.1",
    "tauri": "tauri",
    "tauri:dev": "tauri dev",
    "test": "vitest run",
    "test:ui": "vitest"
  },
  "dependencies": {
    "@tauri-apps/api": "^2.0.0",
    "lucide-react": "^0.468.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@tauri-apps/cli": "^2.0.0",
    "@testing-library/jest-dom": "^6.6.3",
    "@testing-library/react": "^15.0.7",
    "@types/react": "^18.3.12",
    "@types/react-dom": "^18.3.1",
    "@vitejs/plugin-react": "^4.3.3",
    "typescript": "^5.6.3",
    "vite": "^5.4.10",
    "vitest": "^2.1.4"
  }
}
```

Write `index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>DeepAgent Desktop</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

Write `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["DOM", "DOM.Iterable", "ES2020"],
    "allowJs": false,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "ESNext",
    "moduleResolution": "Node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

Write `tsconfig.node.json`:

```json
{
  "compilerOptions": {
    "composite": true,
    "module": "ESNext",
    "moduleResolution": "Node",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts"]
}
```

Write `vite.config.ts`:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  clearScreen: false,
  server: {
    host: "127.0.0.1",
    port: 1420,
    strictPort: true,
    watch: {
      ignored: ["**/src-tauri/**", "**/python-runtime/.venv/**"],
    },
  },
});
```

- [ ] **Step 2: Create the minimal React app**

Write `src/main.tsx`:

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import "./styles.css";

ReactDOM.createRoot(document.getElementById("root") as HTMLElement).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
);
```

Write `src/App.tsx`:

```tsx
export default function App() {
  return (
    <main className="app-shell">
      <aside className="sidebar">
        <div className="brand">DeepAgent Desktop</div>
        <button className="primary-button" type="button">
          Open Project
        </button>
      </aside>
      <section className="timeline">
        <header className="topbar">
          <div>
            <strong>Local coding session</strong>
            <span>Runtime disconnected</span>
          </div>
        </header>
        <div className="empty-state">Start by opening a local repository.</div>
      </section>
      <aside className="right-rail">
        <nav className="tabs" aria-label="Workspace panels">
          <button type="button">Diff</button>
          <button type="button">Terminal</button>
          <button type="button">Approvals</button>
        </nav>
      </aside>
    </main>
  );
}
```

Write `src/styles.css`:

```css
:root {
  color: #20242a;
  background: #f3f5f7;
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
    sans-serif;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
}

button {
  font: inherit;
}

.app-shell {
  display: grid;
  grid-template-columns: 260px minmax(420px, 1fr) 360px;
  min-height: 100vh;
}

.sidebar,
.right-rail {
  background: #ffffff;
  border-color: #d7dce2;
}

.sidebar {
  border-right: 1px solid #d7dce2;
  padding: 18px;
}

.right-rail {
  border-left: 1px solid #d7dce2;
  padding: 14px;
}

.brand {
  font-size: 16px;
  font-weight: 700;
  margin-bottom: 18px;
}

.primary-button {
  width: 100%;
  border: 1px solid #1d4f91;
  border-radius: 6px;
  background: #1d4f91;
  color: #ffffff;
  min-height: 36px;
}

.timeline {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  min-height: 58px;
  border-bottom: 1px solid #d7dce2;
  background: #ffffff;
  padding: 0 18px;
}

.topbar span {
  display: block;
  color: #66717f;
  font-size: 12px;
  margin-top: 2px;
}

.empty-state {
  color: #66717f;
  padding: 24px;
}

.tabs {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 6px;
}

.tabs button {
  min-height: 32px;
  border: 1px solid #c7cdd5;
  border-radius: 6px;
  background: #f7f9fb;
}
```

- [ ] **Step 3: Create the Tauri shell**

Write `src-tauri/Cargo.toml`:

```toml
[package]
name = "deepagent-desktop"
version = "0.1.0"
description = "Local-first AI coding desktop workbench"
authors = ["hxlwd123-lang"]
edition = "2021"

[lib]
name = "deepagent_desktop_lib"
crate-type = ["staticlib", "cdylib", "rlib"]

[build-dependencies]
tauri-build = { version = "2", features = [] }

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tauri = { version = "2", features = [] }
tauri-plugin-shell = "2"
```

Write `src-tauri/build.rs`:

```rust
fn main() {
    tauri_build::build();
}
```

Write `src-tauri/src/main.rs`:

```rust
#[tauri::command]
fn runtime_url() -> &'static str {
    "http://127.0.0.1:8765"
}

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_shell::init())
        .invoke_handler(tauri::generate_handler![runtime_url])
        .run(tauri::generate_context!())
        .expect("failed to run DeepAgent Desktop");
}

fn main() {
    run();
}
```

Write `src-tauri/tauri.conf.json`:

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "DeepAgent Desktop",
  "version": "0.1.0",
  "identifier": "com.deepagent.desktop",
  "build": {
    "beforeDevCommand": "npm run dev",
    "devUrl": "http://127.0.0.1:1420",
    "beforeBuildCommand": "npm run build",
    "frontendDist": "../dist"
  },
  "app": {
    "windows": [
      {
        "title": "DeepAgent Desktop",
        "width": 1360,
        "height": 860,
        "minWidth": 1100,
        "minHeight": 720
      }
    ],
    "security": {
      "csp": null
    }
  },
  "bundle": {
    "active": false,
    "targets": "all"
  }
}
```

Write `src-tauri/capabilities/default.json`:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Default desktop permissions for DeepAgent Desktop",
  "windows": ["main"],
  "permissions": ["core:default", "shell:default"]
}
```

- [ ] **Step 4: Create the Python runtime skeleton**

Write `python-runtime/pyproject.toml`:

```toml
[project]
name = "deepagent-runtime"
version = "0.1.0"
description = "Local Python runtime for DeepAgent Desktop"
requires-python = ">=3.11"
dependencies = [
  "aiosqlite>=0.20.0",
  "deepagents>=0.0.6",
  "fastapi>=0.115.0",
  "langchain-openai>=0.2.0",
  "mcp>=1.8.0,<2.0.0",
  "pydantic>=2.9.0",
  "python-dotenv>=1.0.1",
  "uvicorn[standard]>=0.31.0"
]

[dependency-groups]
dev = [
  "httpx>=0.27.2",
  "pytest>=8.3.3",
  "pytest-asyncio>=0.24.0",
  "ruff>=0.7.0"
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 100
target-version = "py311"
```

Write `python-runtime/app/__init__.py`:

```python
"""DeepAgent Desktop Python runtime."""
```

Write `python-runtime/app/server.py`:

```python
from fastapi import FastAPI


def create_app() -> FastAPI:
    app = FastAPI(title="DeepAgent Desktop Runtime")

    @app.get("/health")
    async def health() -> dict[str, str]:
        return {"status": "ok"}

    return app


app = create_app()
```

Write `python-runtime/tests/test_server.py`:

```python
from fastapi.testclient import TestClient

from app.server import create_app


def test_health_returns_ok() -> None:
    client = TestClient(create_app())

    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

- [ ] **Step 5: Install dependencies and verify the skeleton**

Run:

```powershell
npm install
cd python-runtime
uv sync --dev
uv run pytest
cd ..
npm run build
cargo check --manifest-path src-tauri/Cargo.toml
```

Expected:

```text
python-runtime tests: 1 passed
npm build: TypeScript and Vite complete successfully
cargo check: Finished dev profile
```

- [ ] **Step 6: Commit the skeleton**

Run:

```powershell
git add package.json package-lock.json index.html tsconfig.json tsconfig.node.json vite.config.ts src src-tauri python-runtime
git commit -m "feat: scaffold desktop and runtime workspaces"
```

## Task 2: Add Runtime Event Types and SQLite Persistence

**Files:**
- Create: `python-runtime/app/events.py`
- Create: `python-runtime/app/persistence.py`
- Create: `python-runtime/tests/test_events.py`
- Create: `python-runtime/tests/test_persistence.py`

- [ ] **Step 1: Write failing tests for event serialization**

Write `python-runtime/tests/test_events.py`:

```python
from app.events import ApprovalRequest, RuntimeEvent, ToolCall


def test_runtime_event_serializes_with_type_and_payload() -> None:
    event = RuntimeEvent(
        type="tool_call",
        session_id="session-1",
        payload=ToolCall(
            tool="workspace.read_file",
            summary="Read app.py",
            status="started",
        ).model_dump(),
    )

    assert event.model_dump()["type"] == "tool_call"
    assert event.model_dump()["payload"]["tool"] == "workspace.read_file"
    assert event.model_dump()["payload"]["status"] == "started"


def test_approval_request_contains_action_preview() -> None:
    request = ApprovalRequest(
        approval_id="approval-1",
        action="shell.run",
        risk="high",
        reason="Run tests requested by the user",
        preview="uv run pytest",
    )

    assert request.action == "shell.run"
    assert request.preview == "uv run pytest"
```

- [ ] **Step 2: Run event tests and verify they fail**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_events.py -v
```

Expected:

```text
ModuleNotFoundError: No module named 'app.events'
```

- [ ] **Step 3: Implement runtime event models**

Write `python-runtime/app/events.py`:

```python
from __future__ import annotations

from datetime import UTC, datetime
from typing import Literal
from uuid import uuid4

from pydantic import BaseModel, Field

EventType = Literal[
    "session_started",
    "assistant_token",
    "plan_created",
    "todo_updated",
    "tool_call",
    "approval_required",
    "approval_resolved",
    "diff_created",
    "terminal_output",
    "subagent_update",
    "model_switched",
    "session_completed",
    "runtime_error",
]

ToolStatus = Literal["started", "succeeded", "failed", "blocked"]
RiskLevel = Literal["low", "medium", "high", "blocked"]


def utc_now() -> datetime:
    return datetime.now(UTC)


class RuntimeEvent(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid4()))
    type: EventType
    session_id: str
    timestamp: datetime = Field(default_factory=utc_now)
    payload: dict


class ToolCall(BaseModel):
    tool: str
    summary: str
    status: ToolStatus
    details: dict = Field(default_factory=dict)


class ApprovalRequest(BaseModel):
    approval_id: str
    action: str
    risk: RiskLevel
    reason: str
    preview: str
    details: dict = Field(default_factory=dict)
```

- [ ] **Step 4: Write failing tests for persistence**

Write `python-runtime/tests/test_persistence.py`:

```python
from pathlib import Path

import pytest

from app.events import RuntimeEvent
from app.persistence import RuntimeStore


@pytest.mark.asyncio
async def test_runtime_store_persists_and_loads_events(tmp_path: Path) -> None:
    store = RuntimeStore(tmp_path / "runtime.sqlite3")
    await store.initialize()

    event = RuntimeEvent(
        type="session_started",
        session_id="session-1",
        payload={"workspace": "C:/repo"},
    )
    await store.append_event(event)

    events = await store.list_events("session-1")

    assert len(events) == 1
    assert events[0].id == event.id
    assert events[0].payload == {"workspace": "C:/repo"}
```

- [ ] **Step 5: Run persistence tests and verify they fail**

Run:

```powershell
uv run pytest tests/test_persistence.py -v
```

Expected:

```text
ModuleNotFoundError: No module named 'app.persistence'
```

- [ ] **Step 6: Implement SQLite event persistence**

Write `python-runtime/app/persistence.py`:

```python
from __future__ import annotations

import json
from pathlib import Path

import aiosqlite

from app.events import RuntimeEvent


class RuntimeStore:
    def __init__(self, db_path: Path) -> None:
        self.db_path = db_path

    async def initialize(self) -> None:
        self.db_path.parent.mkdir(parents=True, exist_ok=True)
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute(
                """
                CREATE TABLE IF NOT EXISTS events (
                    id TEXT PRIMARY KEY,
                    session_id TEXT NOT NULL,
                    type TEXT NOT NULL,
                    timestamp TEXT NOT NULL,
                    payload TEXT NOT NULL
                )
                """
            )
            await db.execute(
                "CREATE INDEX IF NOT EXISTS idx_events_session ON events(session_id, timestamp)"
            )
            await db.commit()

    async def append_event(self, event: RuntimeEvent) -> None:
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute(
                """
                INSERT INTO events (id, session_id, type, timestamp, payload)
                VALUES (?, ?, ?, ?, ?)
                """,
                (
                    event.id,
                    event.session_id,
                    event.type,
                    event.timestamp.isoformat(),
                    json.dumps(event.payload),
                ),
            )
            await db.commit()

    async def list_events(self, session_id: str) -> list[RuntimeEvent]:
        async with aiosqlite.connect(self.db_path) as db:
            cursor = await db.execute(
                """
                SELECT id, session_id, type, timestamp, payload
                FROM events
                WHERE session_id = ?
                ORDER BY timestamp ASC
                """,
                (session_id,),
            )
            rows = await cursor.fetchall()
        return [
            RuntimeEvent(
                id=row[0],
                session_id=row[1],
                type=row[2],
                timestamp=row[3],
                payload=json.loads(row[4]),
            )
            for row in rows
        ]
```

- [ ] **Step 7: Run persistence tests**

Run:

```powershell
uv run pytest tests/test_events.py tests/test_persistence.py -v
```

Expected:

```text
4 passed
```

- [ ] **Step 8: Commit event and persistence foundation**

Run:

```powershell
cd ..
git add python-runtime/app/events.py python-runtime/app/persistence.py python-runtime/tests/test_events.py python-runtime/tests/test_persistence.py
git commit -m "feat: add runtime events and persistence"
```

## Task 3: Implement Approval Policy and Workspace Boundaries

**Files:**
- Create: `python-runtime/app/tools/approvals.py`
- Create: `python-runtime/app/tools/workspace.py`
- Create: `python-runtime/tests/test_approvals.py`
- Create: `python-runtime/tests/test_workspace.py`

- [ ] **Step 1: Write failing approval policy tests**

Write `python-runtime/tests/test_approvals.py`:

```python
from app.tools.approvals import ApprovalPolicy


def test_read_workspace_file_is_safe() -> None:
    policy = ApprovalPolicy()

    decision = policy.classify("workspace.read_file", {"path": "app.py"})

    assert decision.requires_approval is False
    assert decision.risk == "low"


def test_shell_command_requires_approval() -> None:
    policy = ApprovalPolicy()

    decision = policy.classify("shell.run", {"command": "uv run pytest"})

    assert decision.requires_approval is True
    assert decision.risk == "high"


def test_write_outside_workspace_is_blocked() -> None:
    policy = ApprovalPolicy()

    decision = policy.classify(
        "workspace.write_file",
        {"path": "C:/Users/32457/.ssh/id_rsa", "within_workspace": False},
    )

    assert decision.blocked is True
    assert decision.risk == "blocked"
```

- [ ] **Step 2: Implement approval policy**

Write `python-runtime/app/tools/approvals.py`:

```python
from __future__ import annotations

from pydantic import BaseModel


class ApprovalDecision(BaseModel):
    requires_approval: bool
    blocked: bool
    risk: str
    reason: str


class ApprovalPolicy:
    safe_tools = {
        "workspace.list_files",
        "workspace.read_file",
        "workspace.search",
        "git.status",
        "git.diff",
    }
    approval_tools = {
        "workspace.write_file",
        "patch.apply",
        "shell.run",
        "dependencies.install",
        "git.commit",
        "mcp.network",
    }

    def classify(self, tool: str, arguments: dict) -> ApprovalDecision:
        if arguments.get("within_workspace") is False:
            return ApprovalDecision(
                requires_approval=False,
                blocked=True,
                risk="blocked",
                reason="Action targets a path outside the selected workspace.",
            )
        if tool in self.safe_tools:
            return ApprovalDecision(
                requires_approval=False,
                blocked=False,
                risk="low",
                reason="Read-only action inside the selected workspace.",
            )
        if tool in self.approval_tools:
            return ApprovalDecision(
                requires_approval=True,
                blocked=False,
                risk="high",
                reason="Action can modify files, run commands, access network, or create commits.",
            )
        return ApprovalDecision(
            requires_approval=True,
            blocked=False,
            risk="medium",
            reason="Unknown tool requires explicit approval.",
        )
```

- [ ] **Step 3: Write failing workspace boundary tests**

Write `python-runtime/tests/test_workspace.py`:

```python
from pathlib import Path

import pytest

from app.tools.workspace import Workspace


def test_resolve_inside_workspace(tmp_path: Path) -> None:
    workspace = Workspace(tmp_path)

    resolved = workspace.resolve("src/app.py")

    assert resolved == tmp_path / "src" / "app.py"


def test_resolve_rejects_parent_escape(tmp_path: Path) -> None:
    workspace = Workspace(tmp_path)

    with pytest.raises(ValueError, match="outside workspace"):
        workspace.resolve("../secrets.txt")


def test_read_file_returns_text(tmp_path: Path) -> None:
    file_path = tmp_path / "app.py"
    file_path.write_text("print('hello')", encoding="utf-8")
    workspace = Workspace(tmp_path)

    assert workspace.read_file("app.py") == "print('hello')"
```

- [ ] **Step 4: Implement workspace tool boundaries**

Write `python-runtime/app/tools/workspace.py`:

```python
from __future__ import annotations

from pathlib import Path


class Workspace:
    def __init__(self, root: Path) -> None:
        self.root = root.resolve()

    def resolve(self, relative_path: str) -> Path:
        candidate = (self.root / relative_path).resolve()
        if candidate != self.root and self.root not in candidate.parents:
            raise ValueError(f"Path is outside workspace: {relative_path}")
        return candidate

    def read_file(self, relative_path: str) -> str:
        path = self.resolve(relative_path)
        return path.read_text(encoding="utf-8")

    def list_files(self) -> list[str]:
        return [
            str(path.relative_to(self.root)).replace("\\", "/")
            for path in self.root.rglob("*")
            if path.is_file() and ".git" not in path.parts
        ]

    def search(self, query: str) -> list[dict[str, str | int]]:
        matches: list[dict[str, str | int]] = []
        for file_path in self.root.rglob("*"):
            if not file_path.is_file() or ".git" in file_path.parts:
                continue
            try:
                lines = file_path.read_text(encoding="utf-8").splitlines()
            except UnicodeDecodeError:
                continue
            for line_number, line in enumerate(lines, start=1):
                if query in line:
                    matches.append(
                        {
                            "path": str(file_path.relative_to(self.root)).replace("\\", "/"),
                            "line": line_number,
                            "preview": line.strip(),
                        }
                    )
        return matches
```

- [ ] **Step 5: Run policy and workspace tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_approvals.py tests/test_workspace.py -v
```

Expected:

```text
6 passed
```

- [ ] **Step 6: Commit approval and workspace tools**

Run:

```powershell
cd ..
git add python-runtime/app/tools/approvals.py python-runtime/app/tools/workspace.py python-runtime/tests/test_approvals.py python-runtime/tests/test_workspace.py
git commit -m "feat: add approval policy and workspace tools"
```

## Task 4: Implement Patch, Shell, and Git Tools

**Files:**
- Create: `python-runtime/app/tools/patch.py`
- Create: `python-runtime/app/tools/shell.py`
- Create: `python-runtime/app/tools/git.py`
- Create: `python-runtime/tests/test_patch.py`
- Create: `python-runtime/tests/test_shell.py`
- Create: `python-runtime/tests/test_git.py`

- [ ] **Step 1: Write failing patch tests**

Write `python-runtime/tests/test_patch.py`:

```python
from pathlib import Path

from app.tools.patch import create_unified_diff, write_file_with_diff


def test_create_unified_diff_contains_old_and_new_lines() -> None:
    diff = create_unified_diff("app.py", "value = 1\n", "value = 2\n")

    assert "--- app.py" in diff
    assert "+++ app.py" in diff
    assert "-value = 1" in diff
    assert "+value = 2" in diff


def test_write_file_with_diff_updates_file(tmp_path: Path) -> None:
    target = tmp_path / "app.py"
    target.write_text("value = 1\n", encoding="utf-8")

    diff = write_file_with_diff(tmp_path, "app.py", "value = 2\n")

    assert target.read_text(encoding="utf-8") == "value = 2\n"
    assert "+value = 2" in diff
```

- [ ] **Step 2: Implement patch helpers**

Write `python-runtime/app/tools/patch.py`:

```python
from __future__ import annotations

import difflib
from pathlib import Path

from app.tools.workspace import Workspace


def create_unified_diff(path: str, before: str, after: str) -> str:
    return "".join(
        difflib.unified_diff(
            before.splitlines(keepends=True),
            after.splitlines(keepends=True),
            fromfile=path,
            tofile=path,
        )
    )


def write_file_with_diff(root: Path, relative_path: str, content: str) -> str:
    workspace = Workspace(root)
    target = workspace.resolve(relative_path)
    before = target.read_text(encoding="utf-8") if target.exists() else ""
    diff = create_unified_diff(relative_path, before, content)
    target.parent.mkdir(parents=True, exist_ok=True)
    target.write_text(content, encoding="utf-8")
    return diff
```

- [ ] **Step 3: Write failing shell tests**

Write `python-runtime/tests/test_shell.py`:

```python
from pathlib import Path

import pytest

from app.tools.shell import run_command


@pytest.mark.asyncio
async def test_run_command_captures_stdout(tmp_path: Path) -> None:
    result = await run_command("python -c \"print('ok')\"", cwd=tmp_path, timeout_seconds=5)

    assert result.exit_code == 0
    assert result.stdout.strip() == "ok"


@pytest.mark.asyncio
async def test_run_command_times_out(tmp_path: Path) -> None:
    result = await run_command(
        "python -c \"import time; time.sleep(3)\"",
        cwd=tmp_path,
        timeout_seconds=1,
    )

    assert result.exit_code == -1
    assert result.timed_out is True
```

- [ ] **Step 4: Implement shell command execution**

Write `python-runtime/app/tools/shell.py`:

```python
from __future__ import annotations

import asyncio
from pathlib import Path

from pydantic import BaseModel


class ShellResult(BaseModel):
    command: str
    exit_code: int
    stdout: str
    stderr: str
    timed_out: bool


async def run_command(command: str, cwd: Path, timeout_seconds: int = 60) -> ShellResult:
    process = await asyncio.create_subprocess_shell(
        command,
        cwd=str(cwd),
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout_seconds)
        return ShellResult(
            command=command,
            exit_code=process.returncode or 0,
            stdout=stdout.decode(errors="replace"),
            stderr=stderr.decode(errors="replace"),
            timed_out=False,
        )
    except TimeoutError:
        process.kill()
        stdout, stderr = await process.communicate()
        return ShellResult(
            command=command,
            exit_code=-1,
            stdout=stdout.decode(errors="replace"),
            stderr=stderr.decode(errors="replace"),
            timed_out=True,
        )
```

- [ ] **Step 5: Write failing git tests**

Write `python-runtime/tests/test_git.py`:

```python
from pathlib import Path

import pytest

from app.tools.git import git_commit, git_status


async def run(cwd: Path, command: str) -> None:
    import asyncio

    process = await asyncio.create_subprocess_shell(command, cwd=str(cwd))
    await process.wait()
    assert process.returncode == 0


@pytest.mark.asyncio
async def test_git_status_reports_changed_file(tmp_path: Path) -> None:
    await run(tmp_path, "git init")
    (tmp_path / "app.py").write_text("print('hi')\n", encoding="utf-8")

    status = await git_status(tmp_path)

    assert "app.py" in status


@pytest.mark.asyncio
async def test_git_commit_creates_commit(tmp_path: Path) -> None:
    await run(tmp_path, "git init")
    await run(tmp_path, "git config user.email test@example.com")
    await run(tmp_path, "git config user.name Test")
    (tmp_path / "app.py").write_text("print('hi')\n", encoding="utf-8")
    await run(tmp_path, "git add app.py")

    result = await git_commit(tmp_path, "test: add app")

    assert result.exit_code == 0
    assert "test: add app" in result.stdout or result.stderr == ""
```

- [ ] **Step 6: Implement git helpers**

Write `python-runtime/app/tools/git.py`:

```python
from __future__ import annotations

from pathlib import Path

from app.tools.shell import ShellResult, run_command


async def git_status(cwd: Path) -> str:
    result = await run_command("git status --short", cwd=cwd, timeout_seconds=15)
    return result.stdout


async def git_diff(cwd: Path) -> str:
    result = await run_command("git diff --", cwd=cwd, timeout_seconds=15)
    return result.stdout


async def git_commit(cwd: Path, message: str) -> ShellResult:
    escaped = message.replace('"', '\\"')
    return await run_command(f'git commit -m "{escaped}"', cwd=cwd, timeout_seconds=30)
```

- [ ] **Step 7: Run tool tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_patch.py tests/test_shell.py tests/test_git.py -v
```

Expected:

```text
6 passed
```

- [ ] **Step 8: Commit patch, shell, and git tools**

Run:

```powershell
cd ..
git add python-runtime/app/tools/patch.py python-runtime/app/tools/shell.py python-runtime/app/tools/git.py python-runtime/tests/test_patch.py python-runtime/tests/test_shell.py python-runtime/tests/test_git.py
git commit -m "feat: add patch shell and git tools"
```

## Task 5: Add Alibaba Model Registry and Router

**Files:**
- Create: `python-runtime/app/models/registry.py`
- Create: `python-runtime/app/models/router.py`
- Create: `python-runtime/app/models/credentials.py`
- Create: `python-runtime/tests/test_models.py`

- [ ] **Step 1: Write failing model registry tests**

Write `python-runtime/tests/test_models.py`:

```python
from app.models.credentials import InMemoryCredentialStore
from app.models.registry import default_registry
from app.models.router import ModelRouter


def test_default_registry_contains_alibaba_qwen_coder() -> None:
    registry = default_registry()

    model = registry.get("alibaba", "qwen3-coder-next")

    assert model.provider == "alibaba"
    assert model.base_url == "https://dashscope.aliyuncs.com/compatible-mode/v1"
    assert model.supports_tools is True


def test_router_builds_chat_model_with_credential_reference() -> None:
    registry = default_registry()
    credentials = InMemoryCredentialStore({"alibaba_default": "test-key"})
    router = ModelRouter(registry, credentials)

    chat_model = router.create_chat_model("alibaba", "qwen3-coder-next")

    assert chat_model.model_name == "qwen3-coder-next"
```

- [ ] **Step 2: Implement credential store and model registry**

Write `python-runtime/app/models/credentials.py`:

```python
from __future__ import annotations

from typing import Protocol


class CredentialStore(Protocol):
    def get_secret(self, key_ref: str) -> str:
        raise NotImplementedError


class InMemoryCredentialStore:
    def __init__(self, values: dict[str, str]) -> None:
        self.values = values

    def get_secret(self, key_ref: str) -> str:
        try:
            return self.values[key_ref]
        except KeyError as exc:
            raise KeyError(f"Missing credential: {key_ref}") from exc
```

Write `python-runtime/app/models/registry.py`:

```python
from __future__ import annotations

from pydantic import BaseModel


class ModelConfig(BaseModel):
    provider: str
    model: str
    base_url: str
    api_key_ref: str
    temperature: float = 0.2
    max_tokens: int = 64000
    reasoning_effort: str = "medium"
    supports_tools: bool = True


class ModelRegistry:
    def __init__(self, models: list[ModelConfig]) -> None:
        self.models = {(model.provider, model.model): model for model in models}

    def get(self, provider: str, model: str) -> ModelConfig:
        try:
            return self.models[(provider, model)]
        except KeyError as exc:
            raise KeyError(f"Unknown model: {provider}/{model}") from exc


def default_registry() -> ModelRegistry:
    return ModelRegistry(
        [
            ModelConfig(
                provider="alibaba",
                model="qwen3-coder-next",
                base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
                api_key_ref="alibaba_default",
            )
        ]
    )
```

- [ ] **Step 3: Implement model router**

Write `python-runtime/app/models/router.py`:

```python
from __future__ import annotations

from langchain_openai import ChatOpenAI

from app.models.credentials import CredentialStore
from app.models.registry import ModelRegistry


class ModelRouter:
    def __init__(self, registry: ModelRegistry, credentials: CredentialStore) -> None:
        self.registry = registry
        self.credentials = credentials

    def create_chat_model(self, provider: str, model: str) -> ChatOpenAI:
        config = self.registry.get(provider, model)
        api_key = self.credentials.get_secret(config.api_key_ref)
        return ChatOpenAI(
            model=config.model,
            api_key=api_key,
            base_url=config.base_url,
            temperature=config.temperature,
            max_tokens=config.max_tokens,
        )
```

- [ ] **Step 4: Run model tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_models.py -v
```

Expected:

```text
2 passed
```

- [ ] **Step 5: Commit model routing**

Run:

```powershell
cd ..
git add python-runtime/app/models python-runtime/tests/test_models.py
git commit -m "feat: add alibaba model routing"
```

## Task 6: Add MCP Configuration Loader

**Files:**
- Create: `python-runtime/app/tools/mcp.py`
- Create: `python-runtime/tests/test_mcp_config.py`

- [ ] **Step 1: Write failing MCP config tests**

Write `python-runtime/tests/test_mcp_config.py`:

```python
from pathlib import Path

from app.tools.mcp import McpConfig, load_mcp_config


def test_load_mcp_config_from_json(tmp_path: Path) -> None:
    config_path = tmp_path / ".deepagent" / "mcp.json"
    config_path.parent.mkdir()
    config_path.write_text(
        """
        {
          "servers": {
            "filesystem": {
              "command": "python",
              "args": ["-m", "example_server"],
              "env": {"MODE": "test"}
            }
          }
        }
        """,
        encoding="utf-8",
    )

    config = load_mcp_config(config_path)

    assert config.servers["filesystem"].command == "python"
    assert config.servers["filesystem"].args == ["-m", "example_server"]
    assert config.servers["filesystem"].env == {"MODE": "test"}


def test_empty_missing_mcp_config_returns_empty(tmp_path: Path) -> None:
    config = load_mcp_config(tmp_path / ".deepagent" / "mcp.json")

    assert config == McpConfig(servers={})
```

- [ ] **Step 2: Implement MCP config loader**

Write `python-runtime/app/tools/mcp.py`:

```python
from __future__ import annotations

import json
from pathlib import Path

from pydantic import BaseModel, Field


class McpServerConfig(BaseModel):
    command: str
    args: list[str] = Field(default_factory=list)
    env: dict[str, str] = Field(default_factory=dict)


class McpConfig(BaseModel):
    servers: dict[str, McpServerConfig] = Field(default_factory=dict)


def load_mcp_config(path: Path) -> McpConfig:
    if not path.exists():
        return McpConfig()
    data = json.loads(path.read_text(encoding="utf-8"))
    return McpConfig.model_validate(data)
```

- [ ] **Step 3: Run MCP config tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_mcp_config.py -v
```

Expected:

```text
2 passed
```

- [ ] **Step 4: Commit MCP configuration support**

Run:

```powershell
cd ..
git add python-runtime/app/tools/mcp.py python-runtime/tests/test_mcp_config.py
git commit -m "feat: add mcp configuration loader"
```

## Task 7: Add Session Manager and Deep Agents Factory

**Files:**
- Create: `python-runtime/app/agent/prompts.py`
- Create: `python-runtime/app/agent/subagents.py`
- Create: `python-runtime/app/agent/factory.py`
- Create: `python-runtime/app/agent/state.py`
- Create: `python-runtime/app/sessions.py`
- Modify: `python-runtime/app/server.py`
- Create: `python-runtime/tests/test_sessions.py`

- [ ] **Step 1: Write failing session tests**

Write `python-runtime/tests/test_sessions.py`:

```python
from pathlib import Path

import pytest

from app.events import RuntimeEvent
from app.persistence import RuntimeStore
from app.sessions import SessionManager


@pytest.mark.asyncio
async def test_create_session_persists_started_event(tmp_path: Path) -> None:
    store = RuntimeStore(tmp_path / "runtime.sqlite3")
    await store.initialize()
    manager = SessionManager(store)

    session = await manager.create_session(workspace=tmp_path, model_provider="alibaba", model="qwen3-coder-next")

    events = await store.list_events(session.session_id)
    assert session.workspace == tmp_path
    assert events[0].type == "session_started"
    assert events[0].payload["model"] == "qwen3-coder-next"


@pytest.mark.asyncio
async def test_emit_event_persists_event(tmp_path: Path) -> None:
    store = RuntimeStore(tmp_path / "runtime.sqlite3")
    await store.initialize()
    manager = SessionManager(store)
    session = await manager.create_session(workspace=tmp_path, model_provider="alibaba", model="qwen3-coder-next")

    await manager.emit(
        RuntimeEvent(
            type="plan_created",
            session_id=session.session_id,
            payload={"plan": ["Read files", "Create patch"]},
        )
    )

    events = await store.list_events(session.session_id)
    assert [event.type for event in events] == ["session_started", "plan_created"]
```

- [ ] **Step 2: Implement prompts and subagent definitions**

Write `python-runtime/app/agent/prompts.py`:

```python
CODING_SYSTEM_PROMPT = """You are DeepAgent Desktop, a local-first AI coding agent.

Follow Plan/Act workflow. Read project context before editing. Use safe read tools freely.
Request approval before writing files, applying patches, running shell commands, installing
dependencies, using network resources, or creating git commits. Keep changes focused on the
user goal. Summarize completed work, test evidence, and remaining risks at the end.
"""
```

Write `python-runtime/app/agent/subagents.py`:

```python
BUILT_IN_SUBAGENTS = [
    {
        "name": "planner",
        "description": "Breaks coding goals into safe implementation steps and risk notes.",
        "prompt": "Create concise implementation plans and identify approval-worthy actions.",
    },
    {
        "name": "reviewer",
        "description": "Reviews final diffs, test evidence, and likely regressions.",
        "prompt": "Review changes for correctness, missing tests, and unsafe side effects.",
    },
    {
        "name": "debugger",
        "description": "Investigates failing tests, command errors, and runtime failures.",
        "prompt": "Diagnose failures from logs and propose minimal corrective actions.",
    },
]
```

- [ ] **Step 3: Implement session state and manager**

Write `python-runtime/app/agent/state.py`:

```python
from __future__ import annotations

from pathlib import Path
from uuid import uuid4

from pydantic import BaseModel, Field


class AgentSession(BaseModel):
    session_id: str = Field(default_factory=lambda: str(uuid4()))
    workspace: Path
    model_provider: str
    model: str
    status: str = "created"
```

Write `python-runtime/app/sessions.py`:

```python
from __future__ import annotations

from pathlib import Path

from app.agent.state import AgentSession
from app.events import RuntimeEvent
from app.persistence import RuntimeStore


class SessionManager:
    def __init__(self, store: RuntimeStore) -> None:
        self.store = store
        self.sessions: dict[str, AgentSession] = {}

    async def create_session(self, workspace: Path, model_provider: str, model: str) -> AgentSession:
        session = AgentSession(
            workspace=workspace.resolve(),
            model_provider=model_provider,
            model=model,
        )
        self.sessions[session.session_id] = session
        await self.emit(
            RuntimeEvent(
                type="session_started",
                session_id=session.session_id,
                payload={
                    "workspace": str(session.workspace),
                    "model_provider": model_provider,
                    "model": model,
                },
            )
        )
        return session

    async def emit(self, event: RuntimeEvent) -> None:
        await self.store.append_event(event)
```

- [ ] **Step 4: Implement Deep Agents factory wrapper**

Write `python-runtime/app/agent/factory.py`:

```python
from __future__ import annotations

from collections.abc import Sequence
from typing import Any

from deepagents import create_deep_agent

from app.agent.prompts import CODING_SYSTEM_PROMPT
from app.agent.subagents import BUILT_IN_SUBAGENTS


def build_coding_agent(model: Any, tools: Sequence[Any]) -> Any:
    return create_deep_agent(
        model=model,
        tools=list(tools),
        system_prompt=CODING_SYSTEM_PROMPT,
        subagents=BUILT_IN_SUBAGENTS,
    )
```

- [ ] **Step 5: Run session tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_sessions.py -v
```

Expected:

```text
2 passed
```

- [ ] **Step 6: Commit session and agent factory**

Run:

```powershell
cd ..
git add python-runtime/app/agent python-runtime/app/sessions.py python-runtime/tests/test_sessions.py
git commit -m "feat: add session manager and agent factory"
```

## Task 8: Expand FastAPI HTTP and WebSocket Runtime API

**Files:**
- Modify: `python-runtime/app/server.py`
- Create: `python-runtime/tests/test_server.py` if not present from Task 1

- [ ] **Step 1: Replace server tests with runtime API coverage**

Write `python-runtime/tests/test_server.py`:

```python
from pathlib import Path

from fastapi.testclient import TestClient

from app.server import create_app


def test_health_returns_ok(tmp_path: Path) -> None:
    client = TestClient(create_app(data_dir=tmp_path))

    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}


def test_create_session_returns_session_id(tmp_path: Path) -> None:
    client = TestClient(create_app(data_dir=tmp_path))

    response = client.post(
        "/sessions",
        json={
            "workspace": str(tmp_path),
            "modelProvider": "alibaba",
            "model": "qwen3-coder-next",
        },
    )

    assert response.status_code == 200
    assert response.json()["workspace"] == str(tmp_path.resolve())
    assert response.json()["model"] == "qwen3-coder-next"


def test_list_session_events_returns_started_event(tmp_path: Path) -> None:
    client = TestClient(create_app(data_dir=tmp_path))
    session = client.post(
        "/sessions",
        json={
            "workspace": str(tmp_path),
            "modelProvider": "alibaba",
            "model": "qwen3-coder-next",
        },
    ).json()

    response = client.get(f"/sessions/{session['sessionId']}/events")

    assert response.status_code == 200
    assert response.json()[0]["type"] == "session_started"
```

- [ ] **Step 2: Implement runtime HTTP API**

Write `python-runtime/app/server.py`:

```python
from __future__ import annotations

from pathlib import Path

from fastapi import FastAPI
from pydantic import BaseModel, Field

from app.persistence import RuntimeStore
from app.sessions import SessionManager


class CreateSessionRequest(BaseModel):
    workspace: Path
    model_provider: str = Field(alias="modelProvider")
    model: str


class SessionResponse(BaseModel):
    session_id: str = Field(alias="sessionId")
    workspace: str
    model_provider: str = Field(alias="modelProvider")
    model: str


def create_app(data_dir: Path | None = None) -> FastAPI:
    app = FastAPI(title="DeepAgent Desktop Runtime")
    runtime_data = data_dir or Path.home() / ".deepagent-desktop"
    store = RuntimeStore(runtime_data / "runtime.sqlite3")
    manager = SessionManager(store)

    @app.on_event("startup")
    async def startup() -> None:
        await store.initialize()

    @app.get("/health")
    async def health() -> dict[str, str]:
        return {"status": "ok"}

    @app.post("/sessions", response_model=SessionResponse)
    async def create_session(request: CreateSessionRequest) -> SessionResponse:
        await store.initialize()
        session = await manager.create_session(
            workspace=request.workspace,
            model_provider=request.model_provider,
            model=request.model,
        )
        return SessionResponse(
            sessionId=session.session_id,
            workspace=str(session.workspace),
            modelProvider=session.model_provider,
            model=session.model,
        )

    @app.get("/sessions/{session_id}/events")
    async def list_events(session_id: str) -> list[dict]:
        await store.initialize()
        events = await store.list_events(session_id)
        return [event.model_dump(mode="json") for event in events]

    return app


app = create_app()
```

- [ ] **Step 3: Run server tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_server.py -v
```

Expected:

```text
3 passed
```

- [ ] **Step 4: Start runtime manually**

Run:

```powershell
uv run uvicorn app.server:app --host 127.0.0.1 --port 8765
```

Expected:

```text
Uvicorn running on http://127.0.0.1:8765
```

Stop the server with `Ctrl+C`.

- [ ] **Step 5: Commit runtime API**

Run:

```powershell
cd ..
git add python-runtime/app/server.py python-runtime/tests/test_server.py
git commit -m "feat: add runtime http api"
```

## Task 9: Add Tauri Runtime Bridge

**Files:**
- Modify: `src-tauri/src/main.rs`
- Modify: `src-tauri/capabilities/default.json`
- Create: `src/api/types.ts`
- Create: `src/api/runtime.ts`
- Modify: `src/App.tsx`

- [ ] **Step 1: Add shared frontend runtime types**

Write `src/api/types.ts`:

```ts
export interface RuntimeHealth {
  status: "ok";
}

export interface CreateSessionRequest {
  workspace: string;
  modelProvider: string;
  model: string;
}

export interface SessionResponse {
  sessionId: string;
  workspace: string;
  modelProvider: string;
  model: string;
}

export type JsonValue = string | number | boolean | null | JsonValue[] | { [key: string]: JsonValue };
export type JsonObject = { [key: string]: JsonValue };

export interface RuntimeEvent {
  id: string;
  type: string;
  session_id: string;
  timestamp: string;
  payload: JsonObject;
}
```

Write `src/api/runtime.ts`:

```ts
import { invoke } from "@tauri-apps/api/core";
import type { CreateSessionRequest, RuntimeEvent, RuntimeHealth, SessionResponse } from "./types";

async function runtimeBaseUrl(): Promise<string> {
  return invoke<string>("runtime_url");
}

export async function fetchHealth(): Promise<RuntimeHealth> {
  const baseUrl = await runtimeBaseUrl();
  const response = await fetch(`${baseUrl}/health`);
  if (!response.ok) {
    throw new Error(`Runtime health failed: ${response.status}`);
  }
  return response.json() as Promise<RuntimeHealth>;
}

export async function createSession(request: CreateSessionRequest): Promise<SessionResponse> {
  const baseUrl = await runtimeBaseUrl();
  const response = await fetch(`${baseUrl}/sessions`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(request),
  });
  if (!response.ok) {
    throw new Error(`Create session failed: ${response.status}`);
  }
  return response.json() as Promise<SessionResponse>;
}

export async function listEvents(sessionId: string): Promise<RuntimeEvent[]> {
  const baseUrl = await runtimeBaseUrl();
  const response = await fetch(`${baseUrl}/sessions/${sessionId}/events`);
  if (!response.ok) {
    throw new Error(`List events failed: ${response.status}`);
  }
  return response.json() as Promise<RuntimeEvent[]>;
}
```

- [ ] **Step 2: Update app to show runtime status**

Modify `src/App.tsx`:

```tsx
import { useEffect, useState } from "react";
import { fetchHealth } from "./api/runtime";

export default function App() {
  const [runtimeStatus, setRuntimeStatus] = useState("checking");

  useEffect(() => {
    fetchHealth()
      .then((health) => setRuntimeStatus(health.status))
      .catch(() => setRuntimeStatus("disconnected"));
  }, []);

  return (
    <main className="app-shell">
      <aside className="sidebar">
        <div className="brand">DeepAgent Desktop</div>
        <button className="primary-button" type="button">
          Open Project
        </button>
      </aside>
      <section className="timeline">
        <header className="topbar">
          <div>
            <strong>Local coding session</strong>
            <span>Runtime {runtimeStatus}</span>
          </div>
        </header>
        <div className="empty-state">Start by opening a local repository.</div>
      </section>
      <aside className="right-rail">
        <nav className="tabs" aria-label="Workspace panels">
          <button type="button">Diff</button>
          <button type="button">Terminal</button>
          <button type="button">Approvals</button>
        </nav>
      </aside>
    </main>
  );
}
```

- [ ] **Step 3: Verify bridge against manual runtime**

Run in terminal A:

```powershell
cd python-runtime
uv run uvicorn app.server:app --host 127.0.0.1 --port 8765
```

Run in terminal B:

```powershell
npm run build
npm run tauri:dev
```

Expected:

```text
The desktop window renders and the topbar shows Runtime ok.
```

- [ ] **Step 4: Commit runtime bridge**

Run:

```powershell
git add src/api src/App.tsx src-tauri/src/main.rs src-tauri/capabilities/default.json
git commit -m "feat: connect desktop shell to runtime"
```

## Task 10: Build Workbench UI Components

**Files:**
- Create: `src/components/ProjectSidebar.tsx`
- Create: `src/components/ActivityTimeline.tsx`
- Create: `src/components/RightRail.tsx`
- Create: `src/components/DiffPanel.tsx`
- Create: `src/components/TerminalPanel.tsx`
- Create: `src/components/ApprovalQueue.tsx`
- Create: `src/components/ModelSettings.tsx`
- Modify: `src/App.tsx`
- Modify: `src/styles.css`

- [ ] **Step 1: Create UI components**

Write `src/components/ProjectSidebar.tsx`:

```tsx
interface ProjectSidebarProps {
  model: string;
}

export function ProjectSidebar({ model }: ProjectSidebarProps) {
  return (
    <aside className="sidebar">
      <div className="brand">DeepAgent Desktop</div>
      <button className="primary-button" type="button">
        Open Project
      </button>
      <section className="sidebar-section">
        <h2>Model</h2>
        <p>{model}</p>
      </section>
      <section className="sidebar-section">
        <h2>Sessions</h2>
        <p>No saved sessions yet.</p>
      </section>
    </aside>
  );
}
```

Write `src/components/ActivityTimeline.tsx`:

```tsx
import type { RuntimeEvent } from "../api/types";

interface ActivityTimelineProps {
  runtimeStatus: string;
  events: RuntimeEvent[];
}

export function ActivityTimeline({ runtimeStatus, events }: ActivityTimelineProps) {
  return (
    <section className="timeline">
      <header className="topbar">
        <div>
          <strong>Local coding session</strong>
          <span>Runtime {runtimeStatus}</span>
        </div>
      </header>
      <div className="event-list">
        {events.length === 0 ? (
          <div className="empty-state">Start by opening a local repository.</div>
        ) : (
          events.map((event) => (
            <article className="event-row" key={event.id}>
              <span>{event.type}</span>
              <pre>{JSON.stringify(event.payload, null, 2)}</pre>
            </article>
          ))
        )}
      </div>
    </section>
  );
}
```

Write `src/components/RightRail.tsx`:

```tsx
import { ApprovalQueue } from "./ApprovalQueue";
import { DiffPanel } from "./DiffPanel";
import { TerminalPanel } from "./TerminalPanel";

export function RightRail() {
  return (
    <aside className="right-rail">
      <nav className="tabs" aria-label="Workspace panels">
        <button type="button">Diff</button>
        <button type="button">Terminal</button>
        <button type="button">Approvals</button>
      </nav>
      <DiffPanel />
      <TerminalPanel />
      <ApprovalQueue />
    </aside>
  );
}
```

Write `src/components/DiffPanel.tsx`:

```tsx
export function DiffPanel() {
  return (
    <section className="panel">
      <h2>Diff</h2>
      <p>No file changes yet.</p>
    </section>
  );
}
```

Write `src/components/TerminalPanel.tsx`:

```tsx
export function TerminalPanel() {
  return (
    <section className="panel">
      <h2>Terminal</h2>
      <p>No command output yet.</p>
    </section>
  );
}
```

Write `src/components/ApprovalQueue.tsx`:

```tsx
export function ApprovalQueue() {
  return (
    <section className="panel">
      <h2>Approvals</h2>
      <p>No pending approvals.</p>
    </section>
  );
}
```

Write `src/components/ModelSettings.tsx`:

```tsx
export function ModelSettings() {
  return (
    <section className="panel">
      <h2>Alibaba Qwen</h2>
      <label>
        API key reference
        <input value="alibaba_default" readOnly />
      </label>
    </section>
  );
}
```

- [ ] **Step 2: Compose the workbench**

Modify `src/App.tsx`:

```tsx
import { useEffect, useState } from "react";
import { fetchHealth } from "./api/runtime";
import type { RuntimeEvent } from "./api/types";
import { ActivityTimeline } from "./components/ActivityTimeline";
import { ProjectSidebar } from "./components/ProjectSidebar";
import { RightRail } from "./components/RightRail";

export default function App() {
  const [runtimeStatus, setRuntimeStatus] = useState("checking");
  const [events] = useState<RuntimeEvent[]>([]);

  useEffect(() => {
    fetchHealth()
      .then((health) => setRuntimeStatus(health.status))
      .catch(() => setRuntimeStatus("disconnected"));
  }, []);

  return (
    <main className="app-shell">
      <ProjectSidebar model="qwen3-coder-next" />
      <ActivityTimeline runtimeStatus={runtimeStatus} events={events} />
      <RightRail />
    </main>
  );
}
```

- [ ] **Step 3: Extend styles for dense workbench panels**

Append to `src/styles.css`:

```css
.sidebar-section {
  border-top: 1px solid #e1e5ea;
  margin-top: 18px;
  padding-top: 14px;
}

.sidebar-section h2,
.panel h2 {
  font-size: 13px;
  margin: 0 0 8px;
}

.sidebar-section p,
.panel p {
  color: #66717f;
  font-size: 13px;
  margin: 0;
}

.event-list {
  overflow: auto;
  padding: 14px;
}

.event-row {
  border-bottom: 1px solid #d7dce2;
  padding: 12px 4px;
}

.event-row span {
  color: #1d4f91;
  font-size: 12px;
  font-weight: 700;
  text-transform: uppercase;
}

.event-row pre {
  background: #ffffff;
  border: 1px solid #d7dce2;
  border-radius: 6px;
  overflow: auto;
  padding: 10px;
}

.panel {
  border-top: 1px solid #e1e5ea;
  margin-top: 14px;
  padding-top: 14px;
}

.panel input {
  border: 1px solid #c7cdd5;
  border-radius: 6px;
  display: block;
  margin-top: 6px;
  min-height: 32px;
  padding: 0 8px;
  width: 100%;
}
```

- [ ] **Step 4: Verify frontend build**

Run:

```powershell
npm run build
```

Expected:

```text
vite build completes successfully
```

- [ ] **Step 5: Commit workbench UI**

Run:

```powershell
git add src
git commit -m "feat: add desktop workbench ui"
```

## Task 11: Add Sample Project and Smoke Test Script

**Files:**
- Create: `samples/calculator-py/pyproject.toml`
- Create: `samples/calculator-py/app.py`
- Create: `samples/calculator-py/test_app.py`
- Create: `docs/development.md`

- [ ] **Step 1: Create sample Python project**

Write `samples/calculator-py/pyproject.toml`:

```toml
[project]
name = "calculator-py"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = []

[dependency-groups]
dev = ["pytest>=8.3.3"]
```

Write `samples/calculator-py/app.py`:

```python
def add(left: int, right: int) -> int:
    return left + right
```

Write `samples/calculator-py/test_app.py`:

```python
from app import add


def test_add() -> None:
    assert add(2, 3) == 5
```

- [ ] **Step 2: Add development instructions**

Write `docs/development.md`:

```markdown
# Development

## Requirements

- Windows
- Node.js 20 or newer
- npm
- Rust stable
- uv
- Python 3.11 or newer

## Install

```powershell
npm install
cd python-runtime
uv sync --dev
cd ..
```

## Run Runtime

```powershell
cd python-runtime
uv run uvicorn app.server:app --host 127.0.0.1 --port 8765
```

## Run Desktop

```powershell
npm run tauri:dev
```

## Test

```powershell
cd python-runtime
uv run pytest
cd ..
npm run build
cargo check --manifest-path src-tauri/Cargo.toml
```
```

- [ ] **Step 3: Verify sample project**

Run:

```powershell
cd samples/calculator-py
uv run pytest
cd ../..
```

Expected:

```text
1 passed
```

- [ ] **Step 4: Commit sample and documentation**

Run:

```powershell
git add samples docs/development.md
git commit -m "docs: add development workflow and sample project"
```

## Task 12: Add Approval Resume API and Event Fanout

**Files:**
- Modify: `python-runtime/app/sessions.py`
- Modify: `python-runtime/app/server.py`
- Modify: `python-runtime/app/events.py`
- Create: `python-runtime/tests/test_approval_flow.py`

- [ ] **Step 1: Write failing approval flow tests**

Write `python-runtime/tests/test_approval_flow.py`:

```python
from pathlib import Path

import pytest

from app.events import ApprovalRequest
from app.persistence import RuntimeStore
from app.sessions import ApprovalResolution, SessionManager


@pytest.mark.asyncio
async def test_request_approval_persists_event_and_pending_state(tmp_path: Path) -> None:
    store = RuntimeStore(tmp_path / "runtime.sqlite3")
    await store.initialize()
    manager = SessionManager(store)
    session = await manager.create_session(tmp_path, "alibaba", "qwen3-coder-next")

    approval = await manager.request_approval(
        session.session_id,
        ApprovalRequest(
            approval_id="approval-1",
            action="shell.run",
            risk="high",
            reason="Run tests",
            preview="uv run pytest",
        ),
    )

    assert approval.approval_id == "approval-1"
    assert manager.pending_approvals["approval-1"].action == "shell.run"
    events = await store.list_events(session.session_id)
    assert events[-1].type == "approval_required"


@pytest.mark.asyncio
async def test_resolve_approval_persists_decision(tmp_path: Path) -> None:
    store = RuntimeStore(tmp_path / "runtime.sqlite3")
    await store.initialize()
    manager = SessionManager(store)
    session = await manager.create_session(tmp_path, "alibaba", "qwen3-coder-next")
    await manager.request_approval(
        session.session_id,
        ApprovalRequest(
            approval_id="approval-1",
            action="git.commit",
            risk="high",
            reason="Create requested commit",
            preview="git commit -m \"test\"",
        ),
    )

    resolution = await manager.resolve_approval(
        "approval-1",
        ApprovalResolution(decision="approved", modified_preview=None),
    )

    assert resolution.decision == "approved"
    assert "approval-1" not in manager.pending_approvals
    events = await store.list_events(session.session_id)
    assert events[-1].type == "approval_resolved"
```

- [ ] **Step 2: Extend event models for approval resolution**

Modify `python-runtime/app/events.py` by adding:

```python
class ApprovalResolved(BaseModel):
    approval_id: str
    decision: Literal["approved", "rejected", "modified"]
    modified_preview: str | None = None
```

- [ ] **Step 3: Extend session manager for pending approvals**

Modify `python-runtime/app/sessions.py`:

```python
from __future__ import annotations

from pathlib import Path

from pydantic import BaseModel

from app.agent.state import AgentSession
from app.events import ApprovalRequest, ApprovalResolved, RuntimeEvent
from app.persistence import RuntimeStore


class ApprovalResolution(BaseModel):
    decision: str
    modified_preview: str | None = None


class PendingApproval(BaseModel):
    session_id: str
    request: ApprovalRequest

    @property
    def approval_id(self) -> str:
        return self.request.approval_id

    @property
    def action(self) -> str:
        return self.request.action


class SessionManager:
    def __init__(self, store: RuntimeStore) -> None:
        self.store = store
        self.sessions: dict[str, AgentSession] = {}
        self.pending_approvals: dict[str, PendingApproval] = {}

    async def create_session(self, workspace: Path, model_provider: str, model: str) -> AgentSession:
        session = AgentSession(
            workspace=workspace.resolve(),
            model_provider=model_provider,
            model=model,
        )
        self.sessions[session.session_id] = session
        await self.emit(
            RuntimeEvent(
                type="session_started",
                session_id=session.session_id,
                payload={
                    "workspace": str(session.workspace),
                    "model_provider": model_provider,
                    "model": model,
                },
            )
        )
        return session

    async def emit(self, event: RuntimeEvent) -> None:
        await self.store.append_event(event)

    async def request_approval(self, session_id: str, request: ApprovalRequest) -> ApprovalRequest:
        self.pending_approvals[request.approval_id] = PendingApproval(
            session_id=session_id,
            request=request,
        )
        await self.emit(
            RuntimeEvent(
                type="approval_required",
                session_id=session_id,
                payload=request.model_dump(),
            )
        )
        return request

    async def resolve_approval(
        self,
        approval_id: str,
        resolution: ApprovalResolution,
    ) -> ApprovalResolved:
        pending = self.pending_approvals.pop(approval_id)
        resolved = ApprovalResolved(
            approval_id=approval_id,
            decision=resolution.decision,
            modified_preview=resolution.modified_preview,
        )
        await self.emit(
            RuntimeEvent(
                type="approval_resolved",
                session_id=pending.session_id,
                payload=resolved.model_dump(),
            )
        )
        return resolved
```

- [ ] **Step 4: Add approval endpoints**

Modify `python-runtime/app/server.py` by adding request models and route handlers inside `create_app`:

```python
class ApprovalResolutionRequest(BaseModel):
    decision: str
    modified_preview: str | None = Field(default=None, alias="modifiedPreview")


@app.post("/approvals/{approval_id}")
async def resolve_approval(approval_id: str, request: ApprovalResolutionRequest) -> dict:
    resolved = await manager.resolve_approval(
        approval_id,
        ApprovalResolution(
            decision=request.decision,
            modified_preview=request.modified_preview,
        ),
    )
    return resolved.model_dump()
```

- [ ] **Step 5: Run approval flow tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_approval_flow.py -v
```

Expected:

```text
2 passed
```

- [ ] **Step 6: Commit approval flow API**

Run:

```powershell
cd ..
git add python-runtime/app/events.py python-runtime/app/sessions.py python-runtime/app/server.py python-runtime/tests/test_approval_flow.py
git commit -m "feat: add approval resume flow"
```

## Task 13: Add WebSocket Event Streaming

**Files:**
- Create: `python-runtime/app/event_bus.py`
- Modify: `python-runtime/app/sessions.py`
- Modify: `python-runtime/app/server.py`
- Modify: `src/api/runtime.ts`
- Modify: `src/api/types.ts`
- Create: `python-runtime/tests/test_event_bus.py`

- [ ] **Step 1: Write failing event bus tests**

Write `python-runtime/tests/test_event_bus.py`:

```python
import pytest

from app.event_bus import EventBus
from app.events import RuntimeEvent


@pytest.mark.asyncio
async def test_event_bus_publishes_to_session_subscriber() -> None:
    bus = EventBus()
    queue = bus.subscribe("session-1")
    event = RuntimeEvent(type="plan_created", session_id="session-1", payload={"plan": []})

    await bus.publish(event)

    received = await queue.get()
    assert received.id == event.id
    assert received.type == "plan_created"
```

- [ ] **Step 2: Implement event bus**

Write `python-runtime/app/event_bus.py`:

```python
from __future__ import annotations

import asyncio
from collections import defaultdict

from app.events import RuntimeEvent


class EventBus:
    def __init__(self) -> None:
        self.subscribers: dict[str, list[asyncio.Queue[RuntimeEvent]]] = defaultdict(list)

    def subscribe(self, session_id: str) -> asyncio.Queue[RuntimeEvent]:
        queue: asyncio.Queue[RuntimeEvent] = asyncio.Queue()
        self.subscribers[session_id].append(queue)
        return queue

    def unsubscribe(self, session_id: str, queue: asyncio.Queue[RuntimeEvent]) -> None:
        self.subscribers[session_id] = [
            subscriber for subscriber in self.subscribers[session_id] if subscriber is not queue
        ]

    async def publish(self, event: RuntimeEvent) -> None:
        for queue in self.subscribers[event.session_id]:
            await queue.put(event)
```

- [ ] **Step 3: Publish emitted session events**

Modify `python-runtime/app/sessions.py` constructor and `emit` method:

```python
from app.event_bus import EventBus


class SessionManager:
    def __init__(self, store: RuntimeStore, event_bus: EventBus | None = None) -> None:
        self.store = store
        self.event_bus = event_bus
        self.sessions: dict[str, AgentSession] = {}
        self.pending_approvals: dict[str, PendingApproval] = {}

    async def emit(self, event: RuntimeEvent) -> None:
        await self.store.append_event(event)
        if self.event_bus is not None:
            await self.event_bus.publish(event)
```

- [ ] **Step 4: Add WebSocket endpoint**

Modify `python-runtime/app/server.py` imports and `create_app` setup:

```python
from fastapi import WebSocket, WebSocketDisconnect

from app.event_bus import EventBus


event_bus = EventBus()
manager = SessionManager(store, event_bus)


@app.websocket("/sessions/{session_id}/events/ws")
async def session_events(websocket: WebSocket, session_id: str) -> None:
    await websocket.accept()
    queue = event_bus.subscribe(session_id)
    try:
        while True:
            event = await queue.get()
            await websocket.send_json(event.model_dump(mode="json"))
    except WebSocketDisconnect:
        event_bus.unsubscribe(session_id, queue)
```

- [ ] **Step 5: Add frontend WebSocket helper**

Modify `src/api/runtime.ts` by adding:

```ts
export async function subscribeToEvents(
  sessionId: string,
  onEvent: (event: RuntimeEvent) => void,
): Promise<() => void> {
  const baseUrl = await runtimeBaseUrl();
  const wsUrl = baseUrl.replace("http://", "ws://").replace("https://", "wss://");
  const socket = new WebSocket(`${wsUrl}/sessions/${sessionId}/events/ws`);
  socket.onmessage = (message) => {
    onEvent(JSON.parse(message.data) as RuntimeEvent);
  };
  return () => socket.close();
}
```

- [ ] **Step 6: Run event streaming tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_event_bus.py tests/test_server.py -v
```

Expected:

```text
All selected tests pass.
```

- [ ] **Step 7: Commit WebSocket streaming**

Run:

```powershell
cd ..
git add python-runtime/app/event_bus.py python-runtime/app/sessions.py python-runtime/app/server.py python-runtime/tests/test_event_bus.py src/api/runtime.ts src/api/types.ts
git commit -m "feat: stream runtime events over websocket"
```

## Task 14: Add Agent Run Loop and Tool Adapters

**Files:**
- Create: `python-runtime/app/agent/runner.py`
- Create: `python-runtime/app/agent/tools.py`
- Modify: `python-runtime/app/server.py`
- Create: `python-runtime/tests/test_agent_runner.py`

- [ ] **Step 1: Write failing agent runner test with a fake planner**

Write `python-runtime/tests/test_agent_runner.py`:

```python
from pathlib import Path

import pytest

from app.agent.runner import AgentRunner
from app.events import RuntimeEvent
from app.persistence import RuntimeStore
from app.sessions import SessionManager


class FakePlanner:
    async def ainvoke(self, request: dict) -> dict:
        return {
            "plan": [
                "Inspect workspace files",
                "Prepare a focused patch",
                "Request approval before writing",
            ],
            "summary": f"Prepared plan for {request['goal']}",
        }


@pytest.mark.asyncio
async def test_agent_runner_emits_plan_created(tmp_path: Path) -> None:
    store = RuntimeStore(tmp_path / "runtime.sqlite3")
    await store.initialize()
    manager = SessionManager(store)
    session = await manager.create_session(tmp_path, "alibaba", "qwen3-coder-next")
    runner = AgentRunner(manager=manager, planner=FakePlanner())

    await runner.plan(session.session_id, "Add tests")

    events = await store.list_events(session.session_id)
    assert "plan_created" in [event.type for event in events]
    assert events[-1].payload["summary"] == "Prepared plan for Add tests"
```

- [ ] **Step 2: Implement agent tool adapter registry**

Write `python-runtime/app/agent/tools.py`:

```python
from __future__ import annotations

from pathlib import Path

from app.tools.git import git_diff, git_status
from app.tools.patch import write_file_with_diff
from app.tools.shell import run_command
from app.tools.workspace import Workspace


class CodingToolset:
    def __init__(self, workspace: Path) -> None:
        self.workspace_path = workspace
        self.workspace = Workspace(workspace)

    def list_files(self) -> list[str]:
        return self.workspace.list_files()

    def read_file(self, path: str) -> str:
        return self.workspace.read_file(path)

    def search(self, query: str) -> list[dict[str, str | int]]:
        return self.workspace.search(query)

    def write_file(self, path: str, content: str) -> str:
        return write_file_with_diff(self.workspace_path, path, content)

    async def run_shell(self, command: str):
        return await run_command(command, self.workspace_path)

    async def git_status(self) -> str:
        return await git_status(self.workspace_path)

    async def git_diff(self) -> str:
        return await git_diff(self.workspace_path)
```

- [ ] **Step 3: Implement runner plan method**

Write `python-runtime/app/agent/runner.py`:

```python
from __future__ import annotations

from typing import Protocol

from app.events import RuntimeEvent
from app.sessions import SessionManager


class PlannerProtocol(Protocol):
    async def ainvoke(self, request: dict) -> dict:
        raise NotImplementedError


class AgentRunner:
    def __init__(self, manager: SessionManager, planner: PlannerProtocol) -> None:
        self.manager = manager
        self.planner = planner

    async def plan(self, session_id: str, goal: str) -> dict:
        result = await self.planner.ainvoke({"goal": goal})
        await self.manager.emit(
            RuntimeEvent(
                type="plan_created",
                session_id=session_id,
                payload={
                    "goal": goal,
                    "plan": result["plan"],
                    "summary": result["summary"],
                },
            )
        )
        return result
```

- [ ] **Step 4: Add plan endpoint**

Modify `python-runtime/app/server.py`:

```python
from app.agent.runner import AgentRunner


class PlanRequest(BaseModel):
    goal: str


class StaticPlanner:
    async def ainvoke(self, request: dict) -> dict:
        goal = request["goal"]
        return {
            "plan": [
                "Inspect relevant files.",
                "Prepare a focused implementation.",
                "Request approval before modifying files or running commands.",
                "Run approved verification commands.",
                "Summarize changes and risks.",
            ],
            "summary": f"Plan created for: {goal}",
        }


runner = AgentRunner(manager=manager, planner=StaticPlanner())


@app.post("/sessions/{session_id}/plan")
async def create_plan(session_id: str, request: PlanRequest) -> dict:
    return await runner.plan(session_id, request.goal)
```

- [ ] **Step 5: Run agent runner tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_agent_runner.py tests/test_server.py -v
```

Expected:

```text
All selected tests pass.
```

- [ ] **Step 6: Commit initial agent run loop**

Run:

```powershell
cd ..
git add python-runtime/app/agent/runner.py python-runtime/app/agent/tools.py python-runtime/app/server.py python-runtime/tests/test_agent_runner.py
git commit -m "feat: add initial agent planning loop"
```

## Task 15: Wire Session Creation, Planning, and Events in the UI

**Files:**
- Modify: `src/api/runtime.ts`
- Modify: `src/api/types.ts`
- Modify: `src/App.tsx`
- Modify: `src/components/ActivityTimeline.tsx`
- Modify: `src/components/ProjectSidebar.tsx`

- [ ] **Step 1: Add frontend plan API**

Modify `src/api/types.ts` by adding:

```ts
export interface PlanRequest {
  goal: string;
}

export interface PlanResponse {
  plan: string[];
  summary: string;
}
```

Modify `src/api/runtime.ts` by adding:

```ts
import type { PlanRequest, PlanResponse } from "./types";

export async function createPlan(sessionId: string, request: PlanRequest): Promise<PlanResponse> {
  const baseUrl = await runtimeBaseUrl();
  const response = await fetch(`${baseUrl}/sessions/${sessionId}/plan`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(request),
  });
  if (!response.ok) {
    throw new Error(`Create plan failed: ${response.status}`);
  }
  return response.json() as Promise<PlanResponse>;
}
```

- [ ] **Step 2: Add a simple session form**

Modify `src/components/ProjectSidebar.tsx`:

```tsx
interface ProjectSidebarProps {
  model: string;
  workspace: string;
  onWorkspaceChange: (workspace: string) => void;
  onCreateSession: () => void;
}

export function ProjectSidebar({
  model,
  workspace,
  onWorkspaceChange,
  onCreateSession,
}: ProjectSidebarProps) {
  return (
    <aside className="sidebar">
      <div className="brand">DeepAgent Desktop</div>
      <label className="field">
        Workspace path
        <input value={workspace} onChange={(event) => onWorkspaceChange(event.target.value)} />
      </label>
      <button className="primary-button" type="button" onClick={onCreateSession}>
        Create Session
      </button>
      <section className="sidebar-section">
        <h2>Model</h2>
        <p>{model}</p>
      </section>
      <section className="sidebar-section">
        <h2>Sessions</h2>
        <p>Current session appears after creation.</p>
      </section>
    </aside>
  );
}
```

- [ ] **Step 3: Wire app session state**

Modify `src/App.tsx`:

```tsx
import { useEffect, useState } from "react";
import { createPlan, createSession, fetchHealth, listEvents, subscribeToEvents } from "./api/runtime";
import type { RuntimeEvent, SessionResponse } from "./api/types";
import { ActivityTimeline } from "./components/ActivityTimeline";
import { ProjectSidebar } from "./components/ProjectSidebar";
import { RightRail } from "./components/RightRail";

export default function App() {
  const [runtimeStatus, setRuntimeStatus] = useState("checking");
  const [workspace, setWorkspace] = useState("C:\\\\Users\\\\32457\\\\Documents\\\\deepagent");
  const [session, setSession] = useState<SessionResponse | null>(null);
  const [goal, setGoal] = useState("Inspect this repository and propose a safe first change.");
  const [events, setEvents] = useState<RuntimeEvent[]>([]);

  useEffect(() => {
    fetchHealth()
      .then((health) => setRuntimeStatus(health.status))
      .catch(() => setRuntimeStatus("disconnected"));
  }, []);

  useEffect(() => {
    if (!session) {
      return;
    }
    let unsubscribe: (() => void) | undefined;
    listEvents(session.sessionId).then(setEvents);
    subscribeToEvents(session.sessionId, (event) => {
      setEvents((current) => [...current, event]);
    }).then((cleanup) => {
      unsubscribe = cleanup;
    });
    return () => unsubscribe?.();
  }, [session]);

  async function handleCreateSession() {
    const created = await createSession({
      workspace,
      modelProvider: "alibaba",
      model: "qwen3-coder-next",
    });
    setSession(created);
  }

  async function handleCreatePlan() {
    if (!session) {
      return;
    }
    await createPlan(session.sessionId, { goal });
  }

  return (
    <main className="app-shell">
      <ProjectSidebar
        model="qwen3-coder-next"
        workspace={workspace}
        onWorkspaceChange={setWorkspace}
        onCreateSession={handleCreateSession}
      />
      <ActivityTimeline
        runtimeStatus={runtimeStatus}
        events={events}
        goal={goal}
        onGoalChange={setGoal}
        onCreatePlan={handleCreatePlan}
        canCreatePlan={session !== null}
      />
      <RightRail />
    </main>
  );
}
```

- [ ] **Step 4: Update timeline component with goal form**

Modify `src/components/ActivityTimeline.tsx`:

```tsx
import type { RuntimeEvent } from "../api/types";

interface ActivityTimelineProps {
  runtimeStatus: string;
  events: RuntimeEvent[];
  goal: string;
  onGoalChange: (goal: string) => void;
  onCreatePlan: () => void;
  canCreatePlan: boolean;
}

export function ActivityTimeline({
  runtimeStatus,
  events,
  goal,
  onGoalChange,
  onCreatePlan,
  canCreatePlan,
}: ActivityTimelineProps) {
  return (
    <section className="timeline">
      <header className="topbar">
        <div>
          <strong>Local coding session</strong>
          <span>Runtime {runtimeStatus}</span>
        </div>
      </header>
      <div className="composer">
        <textarea value={goal} onChange={(event) => onGoalChange(event.target.value)} />
        <button type="button" onClick={onCreatePlan} disabled={!canCreatePlan}>
          Create Plan
        </button>
      </div>
      <div className="event-list">
        {events.length === 0 ? (
          <div className="empty-state">Create a session and ask for a plan.</div>
        ) : (
          events.map((event) => (
            <article className="event-row" key={event.id}>
              <span>{event.type}</span>
              <pre>{JSON.stringify(event.payload, null, 2)}</pre>
            </article>
          ))
        )}
      </div>
    </section>
  );
}
```

- [ ] **Step 5: Add composer styles**

Append to `src/styles.css`:

```css
.field {
  color: #344050;
  display: grid;
  font-size: 13px;
  gap: 6px;
  margin-bottom: 12px;
}

.field input,
.composer textarea {
  border: 1px solid #c7cdd5;
  border-radius: 6px;
  font: inherit;
  padding: 8px;
  width: 100%;
}

.composer {
  background: #ffffff;
  border-bottom: 1px solid #d7dce2;
  display: grid;
  gap: 8px;
  padding: 12px 18px;
}

.composer textarea {
  min-height: 84px;
  resize: vertical;
}

.composer button {
  border: 1px solid #1d4f91;
  border-radius: 6px;
  background: #1d4f91;
  color: #ffffff;
  justify-self: end;
  min-height: 34px;
  padding: 0 14px;
}

.composer button:disabled {
  background: #8793a1;
  border-color: #8793a1;
}
```

- [ ] **Step 6: Verify UI against runtime**

Run runtime:

```powershell
cd python-runtime
uv run uvicorn app.server:app --host 127.0.0.1 --port 8765
```

Run desktop:

```powershell
cd ..
npm run tauri:dev
```

Expected:

```text
Create Session then Create Plan shows session_started and plan_created events in the timeline.
```

- [ ] **Step 7: Commit UI session wiring**

Run:

```powershell
git add src
git commit -m "feat: wire session planning ui"
```

## Task 16: Add End-to-End Local Smoke Path

**Files:**
- Modify: `docs/development.md`
- Create: `python-runtime/tests/test_smoke_flow.py`

- [ ] **Step 1: Write backend smoke flow test**

Write `python-runtime/tests/test_smoke_flow.py`:

```python
from pathlib import Path

from fastapi.testclient import TestClient

from app.server import create_app


def test_create_session_and_plan_flow(tmp_path: Path) -> None:
    (tmp_path / "app.py").write_text("def add(a, b):\n    return a + b\n", encoding="utf-8")
    client = TestClient(create_app(data_dir=tmp_path / ".runtime"))

    session = client.post(
        "/sessions",
        json={
            "workspace": str(tmp_path),
            "modelProvider": "alibaba",
            "model": "qwen3-coder-next",
        },
    ).json()
    plan = client.post(
        f"/sessions/{session['sessionId']}/plan",
        json={"goal": "Add tests for add"},
    ).json()
    events = client.get(f"/sessions/{session['sessionId']}/events").json()

    assert "Inspect relevant files." in plan["plan"]
    assert [event["type"] for event in events] == ["session_started", "plan_created"]
```

- [ ] **Step 2: Add manual smoke checklist**

Append to `docs/development.md`:

```markdown

## Manual Smoke Test

1. Start the runtime:

```powershell
cd python-runtime
uv run uvicorn app.server:app --host 127.0.0.1 --port 8765
```

2. Start the desktop app:

```powershell
cd ..
npm run tauri:dev
```

3. Use workspace path:

```text
C:\Users\32457\Documents\deepagent\samples\calculator-py
```

4. Click `Create Session`.
5. Click `Create Plan`.
6. Confirm the timeline shows `session_started` and `plan_created`.
```

- [ ] **Step 3: Run smoke tests**

Run:

```powershell
cd python-runtime
uv run pytest tests/test_smoke_flow.py -v
```

Expected:

```text
1 passed
```

- [ ] **Step 4: Commit smoke path**

Run:

```powershell
cd ..
git add docs/development.md python-runtime/tests/test_smoke_flow.py
git commit -m "test: add local smoke flow"
```

## Task 17: Full Verification and Push

**Files:**
- No new files.

- [ ] **Step 1: Run Python test suite**

Run:

```powershell
cd python-runtime
uv run pytest
```

Expected:

```text
All Python tests pass.
```

- [ ] **Step 2: Run frontend build**

Run:

```powershell
cd ..
npm run build
```

Expected:

```text
TypeScript and Vite build complete successfully.
```

- [ ] **Step 3: Run Tauri Rust check**

Run:

```powershell
cargo check --manifest-path src-tauri/Cargo.toml
```

Expected:

```text
Finished dev profile.
```

- [ ] **Step 4: Run manual desktop smoke test**

Run:

```powershell
cd python-runtime
uv run uvicorn app.server:app --host 127.0.0.1 --port 8765
```

In a second terminal:

```powershell
npm run tauri:dev
```

Expected:

```text
DeepAgent Desktop opens on Windows and shows Runtime ok.
```

- [ ] **Step 5: Push the implementation branch**

Run:

```powershell
git status --short --branch
git push origin main
```

Expected:

```text
Working tree is clean and origin/main contains all implementation commits.
```

## Self-Review

Spec coverage:

- Local project open: covered by the Tauri/React shell and runtime session API through an editable Windows workspace path in the first build.
- Alibaba Qwen configuration: covered by Task 5.
- Plan/Act session creation: covered by Tasks 7, 14, 15, and 16.
- File read/search, diff, patch, shell, and git commit tools: covered by Tasks 3 and 4.
- Approval policy and approval resume: covered by Tasks 3 and 12.
- MCP configuration: covered by Task 6.
- Event timeline, SQLite persistence, and WebSocket stream: covered by Tasks 2, 8, 10, 13, and 15.
- Windows developer-run app: covered by Tasks 1, 9, 11, 16, and 17.

Type consistency:

- Frontend uses `modelProvider` in JSON payloads and backend maps it with Pydantic aliases.
- Runtime event fields use snake_case in Python and are consumed as received by TypeScript.
- Default model name is consistently `qwen3-coder-next`.

Execution note:

- This plan delivers a developer-run MVP with a tested planning loop, event streaming, approvals, tool foundations, MCP configuration, and UI wiring. A post-MVP plan should deepen the autonomous Act loop with real Deep Agents tool execution, reviewer/debugger dispatch, and richer diff approval controls once this vertical slice is stable.
