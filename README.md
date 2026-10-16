# Multi-Agent DRI Architecture (work in progress)

You pitch a business idea to a root AI agent (the CEO). A hierarchical multi-agent "company" then organizes itself to work on it. You only talk to the CEO; the rest of the hierarchy is handled by the agents.

The design follows the DRI (Directly Responsible Individual) model:

- Every agent has one role and one mission.
- An agent only knows its parent and its direct children.
- A parent creates, configures, monitors and, if needed, removes its children.
- The parent decides which context (skills, constraints, budget) each child receives.

> **Status: work in progress, experimental personal project.** Expect rough edges and breaking changes. See [Status](#status-and-what-is-not-ready).

This is the development repository. A simpler, older snapshot (CLI only, no HTTP server, no dashboard) lives at `Erazor48/Multi_Agent_DRI_Architecture`.

## Architecture

### Agents

| Role | Class | Job |
|---|---|---|
| Root (CEO) | `agents/root.py` | Talks to the founder, designs the company, spawns teams |
| Manager | `agents/manager.py` | Spawns, supervises and synthesizes the work of its children |
| Worker | `agents/worker.py` | Leaf agent: executes a mission with its tools and skills |

All three extend `agents/base.py`. Every LLM call goes through `BaseAgent._call_llm()`, which enforces the token budget and logging. Orchestration is plain `asyncio` with a spawner (`orchestration/spawner.py`) that creates agents on demand and splits the token budget between parents and children. `orchestration/graph.py` is an unused LangGraph skeleton.

### Backend (`src/dri/`)

```
agents/          root (CEO), manager and worker agents
orchestration/   spawner, session executor, persistent-company executor
core/            domain models, agent registry, budget, memory, messages
llm/             providers: Gemini, Anthropic, OpenAI-compatible (+ factory)
skills/          skill catalog: prompt fragments injected into agents
tools/           web search, files, shell, code execution, external actions, TTS
connectors/      email (SMTP, SendGrid), SMS, Slack, LinkedIn, GitHub, YouTube, webhook
storage/         SQLAlchemy 2 async (SQLite by default) + repositories
config/          pydantic-settings configuration
api/cli.py       Typer CLI (`dri`)
api/server.py    FastAPI HTTP server (`dri serve`)
```

- **LLM providers**: `LLM_PROVIDER` selects `gemini`, `anthropic` or `openai_compatible` (OpenAI, OpenRouter, Groq, Ollama, ...). All sit behind one `BaseLLMProvider` interface.
- **Skills** (`skills/catalog.py`): web research, writing, Python development, code review, data analysis, market research, team management, strategic planning, TanStack Start / Convex / Better Auth development, Manim and video production, cold outreach, financial modeling, and a few more.
- **Tools**: `web_search` (Tavily or Brave), `code_exec`, `shell_exec` (executable allowlist, sandboxed to the workspace), file read/write/list/delete with role-based access (RBAC), `tts_generate` (Google Cloud TTS), `propose_external_action`.
- **Approval flow**: agents never act on the outside world directly. External actions (email, SMS, Slack, LinkedIn, GitHub, YouTube, webhook, bulk file delete) are written as proposals that the founder approves or rejects (`dri company approvals ...` or the API).
- **Memory**: persistent per-agent expertise files, a company knowledge base, task history and conversation summarization. `.claude/architecture-memory.md` describes the six layers.
- **Persistence**: companies, messages and agent identities in SQLite; each company has its own workspace directory.

### HTTP API

`uv run dri serve` starts a FastAPI app (default `127.0.0.1:8000`). Routes under `/api/companies`: list, create, get, delete, `chat`, `task`, `approvals` (list / approve / reject), `team`, `status`, `errors`.

### Dashboard (`dashboard/`)

A web UI built with TanStack Start (React 19, Vite, Tailwind 4), Convex and Better Auth. It calls the FastAPI backend (`VITE_API_URL`) for company operations. Convex currently holds one table (`companies`, linking a user to a DRI company id). Pages: login, company list, company detail with chat, team view and approvals.

## Install

Requirements: Python 3.12+, [uv](https://docs.astral.sh/uv/), and [bun](https://bun.sh) for the dashboard.

### Backend

```bash
uv sync --extra dev
cp .env.example .env     # set LLM_PROVIDER and the matching API key
# or: uv run dri setup   # interactive wizard
```

`manim` is a declared dependency and builds `pycairo` / `manimpango`, which need the system cairo and pango development headers. If `uv sync` fails on those, install the headers for your OS first.

```bash
uv run dri --help
uv run dri run --pitch "My idea" --budget 500000   # one-shot session
uv run dri company create --pitch "My idea"        # persistent company
uv run dri company chat --id <ID>
uv run dri company task --task "Produce a market analysis report"
uv run dri company approvals list
uv run dri serve                                   # HTTP API on :8000
```

`.claude/CLI-reference.md` has the full command reference. `.env.example` documents every setting (providers, budget, connectors).

### Dashboard

```bash
cd dashboard
cp .env.example .env     # Convex and Better Auth values, see below
bun install
bunx convex dev          # creates/links a Convex project and prints the values for .env
bun run dev              # http://localhost:3000
```

`dashboard/.env.example` lists `VITE_API_URL`, `CONVEX_DEPLOYMENT`, `VITE_CONVEX_URL`, `BETTER_AUTH_SECRET` and `BETTER_AUTH_URL`. Generate the secret yourself (`openssl rand -hex 32`). The backend must be running for the dashboard to be useful. Full deployment setup: to be completed.

## Tests

```bash
uv run pytest tests/unit          # backend unit tests, no network
cd dashboard && bun run test      # dashboard API client tests (vitest)
```

`tests/integration` calls real LLM providers and connectors and needs valid credentials; do not run it without them.

## Status and what is not ready

- Work in progress: APIs, CLI flags and data schemas may change without notice.
- The dashboard is early (Sprint 20): company list, chat, team and approvals only. Its Convex schema is minimal.
- `dri org` is a placeholder ("coming soon"). `orchestration/graph.py` (LangGraph) is unused.
- The HTTP API identifies the user from an `X-User-Id` request header and does not verify it (no token check). Do not expose `dri serve` to an untrusted network.
- Agents can run shell commands and call real services with your credentials. Review proposed actions before approving them, and keep API keys in `.env` only (git-ignored).
- Unit tests: 204 passed, 2 failed on the last run on Linux without `manim` installed. `test_shell_bonus_accumulates` is timing-dependent (fixed in the older public repo), and `test_recover_scan_ignores_known_company` fails in the full run but passes alone (probably test ordering). To be investigated.
- Documentation of the internals is mostly in French in `CLAUDE.md` and `.claude/`, written as instructions for AI coding agents.

## License

MIT, see [LICENSE](LICENSE).
