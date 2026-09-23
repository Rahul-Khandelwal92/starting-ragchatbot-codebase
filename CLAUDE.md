# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A full-stack RAG (Retrieval-Augmented Generation) chatbot that answers questions about course materials. FastAPI backend, ChromaDB for vector storage, Anthropic Claude for generation via tool-calling, and a static HTML/CSS/JS frontend served directly by FastAPI.

## Commands

Package manager is `uv` (not pip/poetry). Python >=3.13 required.

```bash
# Install dependencies
uv sync

# Run the app (from repo root) — Windows users need Git Bash for this script
./run.sh

# Run manually (equivalent to run.sh)
cd backend && uv run uvicorn app:app --reload --port 8000
```

App serves at `http://localhost:8000`, API docs at `http://localhost:8000/docs`.

Requires a `.env` file in the repo root with `ANTHROPIC_API_KEY=...` (see `.env.example`).

There is no test suite, linter, or formatter configured in this repo.

## Architecture

### Request flow

`frontend/script.js` POSTs `{query, session_id}` to `POST /api/query` (`backend/app.py`). FastAPI creates a session if `session_id` is null, then delegates to `RAGSystem.query()` (`backend/rag_system.py`), which is the central orchestrator:

1. Pulls prior conversation turns from `SessionManager` (in-memory only, capped at `MAX_HISTORY` exchanges — lost on restart).
2. Calls `AIGenerator.generate_response()` (`backend/ai_generator.py`), which sends the query to Claude with the `search_course_content` tool available (`tool_choice: auto`).
3. If Claude emits a `tool_use` block, `AIGenerator` executes it via `ToolManager` → `CourseSearchTool` (`backend/search_tools.py`), which calls `VectorStore.search()` (`backend/vector_store.py`), then makes a **second** Claude call (tools disabled) with the tool results appended to synthesize the final answer. Claude's system prompt enforces "one search per query maximum" — this is a prompt convention, not code-enforced.
4. `RAGSystem` reads sources off the tool (`ToolManager.get_last_sources()`), resets them, saves the exchange to session history, and returns `(answer, sources)` back through FastAPI to the frontend.

### Vector storage (`backend/vector_store.py`)

ChromaDB with two separate collections, both using the `all-MiniLM-L6-v2` sentence-transformer embedding function:
- `course_catalog` — one entry per course (title as ID, instructor, lesson list as JSON). Used to semantically resolve a fuzzy course name (e.g. "MCP") to an exact title via `_resolve_course_name()`.
- `course_content` — the actual chunked text, filterable by resolved `course_title` and/or `lesson_number`.

Search is always two-step when a course name is given: resolve the name against `course_catalog`, then filter-query `course_content` with the resolved exact title.

### Document ingestion (`backend/document_processor.py`)

On startup, `app.py`'s `startup_event` loads every `.pdf`/`.docx`/`.txt` file from `docs/` into the vector store (skipping courses whose title already exists — see `RAGSystem.add_course_folder()`). Text is split into sentence-aware overlapping chunks per `CHUNK_SIZE`/`CHUNK_OVERLAP`.

### Config (`backend/config.py`)

Single `Config` dataclass, instantiated once as `config` and imported wherever needed. Key tunables: `ANTHROPIC_MODEL`, `EMBEDDING_MODEL`, `CHUNK_SIZE`/`CHUNK_OVERLAP`, `MAX_RESULTS` (search results returned), `MAX_HISTORY` (conversation turns remembered), `CHROMA_PATH`.

### Adding a new tool

Tools implement the `Tool` ABC in `search_tools.py` (`get_tool_definition()` returning an Anthropic tool schema, `execute(**kwargs)`), then get registered on `RAGSystem`'s `ToolManager` via `tool_manager.register_tool(...)`. `AIGenerator` and `ToolManager` are otherwise tool-agnostic.
