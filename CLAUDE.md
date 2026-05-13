# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run all tests
pytest

# Run specific test suites
pytest -m unit
pytest -m integration
pytest -m websocket

# Run a single test file
pytest app/tests/test_app.py

# Run with coverage
pytest --cov=app --cov-report=html
```

There is no lint/format command configured in CI. The project uses `ruff check` for linting locally.

## Architecture

LlamaBot is an AI coding agent framework: FastAPI backend that streams LangGraph agent responses to a vanilla JS frontend over WebSocket.

### Request flow

1. Browser connects via WebSocket to `/ws`
2. `WebSocketHandler` (app/websocket/web_socket_handler.py) manages the connection lifecycle — auth, ping/pong, cancel, and dispatching messages
3. `RequestHandler.handle_request()` (app/websocket/request_handler.py) resolves the agent name from `langgraph.json`, loads the pre-compiled LangGraph workflow from `app.state.compiled_graphs`, and streams results via `app.astream(stream_mode=["updates", "messages"], subgraphs=True)`
4. Two chunk types are streamed to the frontend:
   - `AIMessageChunk` — token-by-token LLM output (with thinking/reasoning separated from text)
   - State updates — tool calls, final AI messages, token usage metadata

### Agent architecture

Agents are LangGraph workflows built with `langchain.agents.create_agent`. Each agent has:
- A `build_workflow(checkpointer)` entrypoint in `nodes.py`
- Registered in `langgraph.json` (`graphs` key maps name → module path)
- Compiled once at startup (singleton, cached in `app.state.compiled_graphs`)
- A `state_schema` extending `AgentState` with custom fields (e.g., `RailsAgentState` adds `todos`, `llm_model`, `failed_tool_calls_count`)

The main `rails_agent` has a middleware stack (order matters):
1. `SummarizationMiddleware` — triggers on token threshold, uses Gemini 3 Flash
2. `DynamicModelMiddleware` — switches LLM based on `state.llm_model` from frontend
3. `deepseek_reasoning_fix` — injects reasoning_content for multi-turn DeepSeek
4. `inject_view_context` — prepends page context to user messages
5. `check_failure_limit` — circuit breaker after 3 failed tool calls

`app/agents/leonardo/project_context.py` loads `.leonardo/LEONARDO.md` and appends it to system prompts with Anthropic prompt caching (`cache_control: ephemeral`).

### Persistence

Two separate PostgreSQL databases:
- **Auth/App DB** (`LEONARDO_DB_URI` > `AUTH_DB_URI`): User model, thread metadata, prompts, scheduled jobs, etc. via SQLModel
- **Checkpointer DB** (`CHECKPOINTER_DB_URI` > `LEONARDO_DB_URI` > `AUTH_DB_URI` > `DB_URI`): LangGraph conversation state via `AsyncPostgresSaver`

Both gracefully fall back to in-memory (`MemorySaver`) if PostgreSQL is unavailable.

### API auth

HTTP Basic Auth via `app/dependencies.py`. The `auth` dependency validates against the `User` table. Endpoints use `auth`, `get_current_user`, and `engineer_or_admin_required` dependencies.

### Key state conventions

- `app.state.compiled_graphs` — dict of pre-compiled LangGraph workflows
- `app.state.async_checkpointer` — shared `AsyncPostgresSaver` (or `MemorySaver`)
- `app.state.checkpointer_pool` — `AsyncConnectionPool` used by checkpoint cleanup
- Agent state fields like `llm_model` and `agent_mode` are passed through from the WebSocket message to LangGraph state naturally (all fields except `message`, `agent_name`, `thread_id`, `attachments` are forwarded)

### Multimodal content

`RequestHandler._build_message_content()` inspects attachments and checks `MODEL_CAPABILITIES` dict (model → supported types). Unsupported file types get a text note instead. Video only works with Gemini models.

### Frontend

Modular vanilla ES6 JavaScript under `app/frontend/chat/`:
- `index.js` — main `ChatApp` class
- `websocket/` — connection management
- `messages/` — message rendering
- `ui/` — UI components
- `threads/` — thread list management
