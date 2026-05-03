# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# First-time setup: copy and populate the API key
cp .env.example .env   # then set ANTHROPIC_API_KEY in .env

# Start the server (installs deps via uv, hot-reload on :8000)
./run.sh

# Or manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

- Web UI: `http://localhost:8000`
- Auto-generated API docs: `http://localhost:8000/docs`
- On startup, documents from `docs/` are automatically ingested into ChromaDB.

## Architecture

This is a full-stack RAG chatbot. The backend is a single FastAPI process (`backend/`) that serves both the API and the static frontend (`frontend/`).

### Request lifecycle

1. `frontend/script.js` sends `POST /api/query` with `{query, session_id}`
2. `app.py` delegates to `RAGSystem.query()`
3. `RAGSystem` fetches conversation history from `SessionManager`, then calls `AIGenerator.generate_response()` with Claude tool definitions
4. Claude (`claude-sonnet-4-20250514`) decides whether to call the `search_course_content` tool
5. If tool use: `ToolManager` → `CourseSearchTool` → `VectorStore.search()` hits ChromaDB, then a second Claude call synthesizes the result
6. Sources and the exchange are stored back in `SessionManager`

### Key design decisions

- **Two ChromaDB collections**: `course_catalog` (course-level metadata, used for fuzzy course-name resolution) and `course_content` (chunked lesson text, used for semantic search). Queries always hit `course_content`; `course_catalog` is only queried when a `course_name` filter is provided.
- **Tool-based retrieval**: Claude decides whether to search — it is not forced to retrieve on every query. General knowledge questions skip the vector store entirely.
- **Agentic loop is single-depth**: one tool call maximum per query (`_handle_tool_execution` makes exactly one follow-up Claude call; tools are stripped from that second call).
- **Session history is in-memory**: `SessionManager` holds sessions in a plain dict. History is capped at `MAX_HISTORY=2` exchanges and formatted as plain text injected into the system prompt.

### Course document format

Files in `docs/` must follow this structure for correct parsing:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <title>
Lesson Link: <url>
... lesson body ...

Lesson 2: <title>
...
```
`DocumentProcessor` parses this format, chunks lesson bodies (800 chars, 100 overlap, sentence-aware), and tags each chunk with `course_title` and `lesson_number` metadata for filtered retrieval.

## Configuration

All tunable parameters are in `backend/config.py` via environment / `.env`:

| Key | Default | Effect |
|---|---|---|
| `ANTHROPIC_API_KEY` | — | Required |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Claude model used |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Local sentence-transformer for ChromaDB |
| `CHUNK_SIZE` | `800` | Max characters per chunk |
| `CHUNK_OVERLAP` | `100` | Character overlap between chunks |
| `MAX_RESULTS` | `5` | Top-k chunks returned per search |
| `MAX_HISTORY` | `2` | Conversation exchanges retained per session |
| `CHROMA_PATH` | `./chroma_db` | ChromaDB persistence directory (relative to `backend/`) |

## Package Management

Uses `uv`. All commands must be run from the repo root or prefixed accordingly:

```bash
uv sync               # install/update dependencies
uv add <package>      # add a dependency
```

Dependencies are declared in `pyproject.toml`. There is no separate test suite or linter configured.
