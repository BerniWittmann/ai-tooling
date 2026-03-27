# Castellan

> Castellan is a self-hosted control plane for orchestrating AI coding agents across multiple projects — built on top of [Sandcastle](https://github.com/mattpocock/sandcastle).

---

## The Problem

Sandcastle is a powerful tool for running AI coding agents inside sandboxed Docker containers. But it's designed to live inside a single project repo, and it uses GitHub Issues as its task tracker — which means your AI orchestration scaffolding is coupled to your codebase and your tasks are public.

**agent-hub** solves this by pulling Sandcastle out of your projects entirely, giving it a permanent home, a proper task management UI, and multi-project support.

---

## Core Idea

agent-hub is a standalone repo that lives **outside all your project repos**. It manages:

- Which projects exist and where they live on disk
- The task backlog for each project (stored in a local SQLite database)
- Per-project Sandcastle configuration (Dockerfile, prompt templates, env vars)
- Agent run history and logs
- A web UI for planning and a CLI for execution

Your actual project repos stay completely clean — no `.sandcastle/` directories, no agent tooling, no leaked tasks.

---

## Repository Structure

```
agent-hub/
├── apps/
│   ├── web/                  # React + Vite — task planning UI
│   └── cli/                  # Ink (React for terminal) — agent runner
├── packages/
│   └── core/                 # Shared types, DB schema, Sandcastle runner logic
├── data/
│   ├── agent-hub.db          # SQLite database (gitignored)
│   └── sandcastle/
│       ├── my-app/           # Per-project Sandcastle config
│       │   ├── Dockerfile
│       │   ├── prompt.md
│       │   └── .env
│       └── other-project/
├── package.json              # npm workspaces monorepo root
├── .env                      # Global secrets (ANTHROPIC_API_KEY, etc.)
└── .gitignore
```

---

## Data Model

Stored in SQLite via **Drizzle ORM**.

### `projects`
| Column | Type | Notes |
|---|---|---|
| id | text (uuid) | Primary key |
| name | text | Display name |
| repoPath | text | Absolute path on disk to the target repo |
| sandcastleImageName | text | Docker image name for this project |
| createdAt | integer | Unix timestamp |

### `tasks`
| Column | Type | Notes |
|---|---|---|
| id | text (uuid) | Primary key |
| projectId | text | Foreign key → projects |
| title | text | Short task name |
| body | text | Full task description in Markdown |
| status | text | `backlog` \| `ready` \| `running` \| `review` \| `done` |
| order | integer | Position within a status column |
| createdAt | integer | Unix timestamp |

### `runs`
| Column | Type | Notes |
|---|---|---|
| id | text (uuid) | Primary key |
| taskId | text | Foreign key → tasks |
| projectId | text | Foreign key → projects |
| branch | text | Git branch the agent worked on |
| status | text | `pending` \| `running` \| `success` \| `failed` |
| startedAt | integer | Unix timestamp |
| finishedAt | integer | Unix timestamp (nullable) |

### `run_logs`
| Column | Type | Notes |
|---|---|---|
| id | text (uuid) | Primary key |
| runId | text | Foreign key → runs |
| content | text | Log chunk (streamed in real-time) |
| createdAt | integer | Unix timestamp |

---

## Web UI (`apps/web`)

**Stack:** React + Vite + Tailwind CSS + dnd-kit

### Features

**Projects sidebar**
- List all registered projects
- Add a project by entering a name and selecting a local repo path
- Quick-jump between projects

**Kanban board**
- One board per project
- Columns: `Backlog → Ready → Running → Review → Done`
- Drag and drop cards between columns
- Click a card to open a full Markdown editor for the task body
- "Queue for agent" button moves a task to `ready` status

**Run history**
- Per-project and per-task views
- Live log streaming while an agent is running
- Show branch name, duration, and final status per run

**Task creation**
- Click "New task" to open a split-pane editor: Markdown source on the left, preview on the right
- Tasks saved directly to SQLite

---

## CLI (`apps/cli`)

**Stack:** Ink (React for terminal) + Commander.js

### Usage

```bash
# Start the interactive runner
npm run agent

# Or target a specific project directly
npm run agent -- --project my-app
```

### Interactive flow

```
agent-hub CLI

? Pick a project:
  ❯ my-app        (3 tasks ready)
    other-project (1 task ready)

? my-app — 3 tasks ready. How to proceed?
  ❯ Run all sequentially
    Pick a task

[my-app] [1/3] "Add login page"         ▶ running...
[my-app] [2/3] "Refactor auth middleware"  ⏸ queued
[my-app] [3/3] "Add password reset flow"   ⏸ queued
```

Logs stream to both the terminal and the database in real time, so the web UI stays in sync.

---

## Agent Runner (`packages/core`)

The core package wraps `@ai-hero/sandcastle` and adds:

- **Task injection:** pulls the task `body` from SQLite and passes it as the `prompt` to `sandcastle.run()`
- **Log streaming:** pipes Sandcastle output to `run_logs` as it arrives
- **Status management:** updates task and run status throughout the lifecycle
- **Concurrency control:** different projects run in parallel; tasks within the same project run sequentially to avoid branch conflicts

```typescript
// Simplified example of what core does
import { run } from "@ai-hero/sandcastle";

async function executeTask(task: Task, project: Project) {
  await updateTaskStatus(task.id, "running");

  const result = await run({
    prompt: task.body,
    branch: `agent/${task.id}`,
    // points at the project's local repo, not agent-hub
    // achieved by running with cwd set to project.repoPath
    hooks: { onSandboxReady: [{ command: "npm install" }] },
    logging: { type: "stdout" }, // we intercept and write to DB
  });

  await updateTaskStatus(task.id, result.wasCompletionSignalDetected ? "review" : "failed");
}
```

---

## Per-Project Sandcastle Configuration

Each project gets its own config directory at `data/sandcastle/<project-slug>/`:

```
data/sandcastle/my-app/
├── Dockerfile       # Sandbox environment for this project
├── prompt.md        # Base prompt template (task body injected at runtime)
└── .env             # Project-specific secrets (gitignored)
```

The `Dockerfile` is seeded from Sandcastle's default template when you add a project, and can be customized freely (e.g. to add project-specific dependencies like Python, Ruby, etc.).

The `prompt.md` is a wrapper template — at runtime, the specific task body is injected into it:

```markdown
# Context

You are working on the `my-app` repository.

!`git log --oneline -5`

# Task

{{TASK_BODY}}

When you are done, output <promise>COMPLETE</promise>.
```

---

## Tech Stack Summary

| Layer | Technology |
|---|---|
| Monorepo | npm workspaces |
| Database | SQLite + Drizzle ORM |
| Web UI | React + Vite + Tailwind CSS + dnd-kit |
| CLI | Ink + Commander.js |
| Agent runner | `@ai-hero/sandcastle` |
| Sandboxing | Docker (via Sandcastle) |

---

## Development Setup

```bash
# Clone and install
git clone <your-private-repo>/agent-hub
cd agent-hub
npm install

# Set up environment
cp .env.example .env
# Add your ANTHROPIC_API_KEY

# Run the web UI
npm run dev --workspace=apps/web

# Run the CLI
npm run agent
```

---

## Implementation Phases

### Phase 1 — Foundation
- [ ] Set up npm workspaces monorepo
- [ ] Define SQLite schema with Drizzle (`packages/core`)
- [ ] CRUD operations for projects and tasks in core
- [ ] Basic CLI: list projects, list tasks, run a single task via Sandcastle
- [ ] Log streaming from Sandcastle → database

### Phase 2 — Web UI
- [ ] Vite + React + Tailwind scaffold
- [ ] Projects sidebar with add/remove
- [ ] Kanban board with drag-and-drop (dnd-kit)
- [ ] Markdown task editor
- [ ] Run history page with log viewer

### Phase 3 — Polish
- [ ] Live log streaming in web UI (polling or WebSocket)
- [ ] Parallel multi-project runner in CLI
- [ ] Per-project Sandcastle config scaffolding on project creation
- [ ] Web UI: "Queue for agent" button triggers CLI runner (or a background process)
- [ ] Notifications (desktop or webhook) on run completion

---

## Non-Goals

- **Not a hosted SaaS** — runs entirely on your machine, no cloud dependency beyond the Anthropic API
- **Not a replacement for your git workflow** — agent branches still go through your normal PR/review process
- **Not opinionated about your project stack** — the Dockerfile is yours to customize