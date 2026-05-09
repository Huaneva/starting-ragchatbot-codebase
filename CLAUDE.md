# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Setup**
```bash
uv sync                  # Install dependencies
cp .env.example .env     # Then add ANTHROPIC_API_KEY to .env
```

**Run the server**
```bash
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

The server must be run from the `backend/` directory (imports are relative to it). The app serves the frontend at `http://localhost:8000` and API docs at `http://localhost:8000/docs`.

There are no tests or linting configurations in this codebase.

## Architecture

The system is a RAG chatbot over structured course `.txt` files. The server (`backend/app.py`) exposes two endpoints — `POST /api/query` and `GET /api/courses` — and serves the static frontend from `frontend/`.

**Query flow:**
1. `RAGSystem.query()` builds a prompt and calls `AIGenerator.generate_response()`
2. Claude decides whether to invoke `search_course_content` (one call max per query)
3. If invoked, `CourseSearchTool.execute()` → `VectorStore.search()` → ChromaDB similarity search, optionally filtered by course name or lesson number
4. Tool result is appended to the message thread; Claude synthesizes the final answer
5. `SessionManager` stores the exchange (last `MAX_HISTORY=2` turns, serialized as a plain string injected into the system prompt)

**Key design decisions:**
- ChromaDB uses two collections: `course_catalog` (one doc per course, for fuzzy course-name resolution) and `course_content` (chunked lesson text, filtered by `course_title` and `lesson_number` metadata)
- Course name matching is semantic: a vague query like "MCP course" is resolved via a vector search against `course_catalog` before content is fetched
- Tools are abstracted behind `Tool` (ABC) + `ToolManager`; adding a new tool means subclassing `Tool`, implementing `get_tool_definition()` and `execute()`, then calling `tool_manager.register_tool()`
- Session history is stored in-memory only (lost on restart)

**Course document format** (files in `docs/`):
```
Course Title: ...
Course Link: ...
Course Instructor: ...
Lesson 0: Title
Lesson Link: ...
[lesson content...]
Lesson 1: Title
...
```
`DocumentProcessor` parses this into `Course`/`Lesson`/`CourseChunk` Pydantic models. Chunks are ~800 chars with 100-char sentence-boundary overlap. On startup, `app.py` loads all docs from `../docs/` and skips courses already present in ChromaDB.

**Config** (`backend/config.py`): all tunables (`CHUNK_SIZE`, `CHUNK_OVERLAP`, `MAX_RESULTS`, `MAX_HISTORY`, `ANTHROPIC_MODEL`, `EMBEDDING_MODEL`, `CHROMA_PATH`) live in the `Config` dataclass and are imported as the singleton `config`.
