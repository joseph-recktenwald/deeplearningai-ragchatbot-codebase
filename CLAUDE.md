# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Run the application:**
```bash
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Install dependencies:**
```bash
uv sync
```

There are no tests or linting configured in this project.

## Environment Setup

Requires Python 3.13+ and uv. Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. The app runs at http://localhost:8000.

## Architecture

This is a RAG (Retrieval-Augmented Generation) chatbot that answers questions about course materials loaded from `docs/`.

### Query Flow

User question → `app.py` POST `/api/query` → `RAGSystem.query()` → `AIGenerator` sends query to Claude with tool definitions → Claude decides whether to call `search_course_content` tool → `CourseSearchTool` executes semantic search via `VectorStore` (ChromaDB) → results sent back to Claude → Claude synthesizes final answer → response returned with sources.

Key detail: Claude autonomously decides whether to search using Anthropic's tool use feature. The tool call results are sent back as a follow-up message, and Claude generates the final response *without* tools available (preventing infinite tool loops).

### Data Ingestion Flow

On startup, `app.py` triggers `RAGSystem.add_course_folder("../docs")` → `DocumentProcessor` parses each `.txt` file (extracting course metadata + lessons) → text is chunked (800 chars, 100 overlap) → chunks stored in ChromaDB `course_content` collection; course metadata stored in `course_catalog` collection. Existing courses are skipped via title deduplication.

### Component Responsibilities

- **`rag_system.py`** — Orchestrator that wires together all components. Entry point for queries and document ingestion.
- **`ai_generator.py`** — Claude API wrapper. Manages system prompt, conversation history injection, and the tool-use → tool-result → final-response cycle.
- **`search_tools.py`** — Extensible tool system. `Tool` ABC defines the interface; `CourseSearchTool` implements `search_course_content`; `ToolManager` is the registry/executor. Sources are tracked per-tool via `last_sources`.
- **`vector_store.py`** — ChromaDB wrapper with two collections: `course_catalog` (for fuzzy course name resolution via semantic search) and `course_content` (for content retrieval). Filter building supports course + lesson combinations.
- **`document_processor.py`** — Parses a specific document format (title/link/instructor header, then `Lesson N:` markers) and chunks text by sentences with overlap.
- **`session_manager.py`** — In-memory conversation history (default: last 2 exchanges per session). History is injected into the system prompt as plain text.
- **`config.py`** — Single `Config` dataclass; reads `ANTHROPIC_API_KEY` from env vars, all other settings are hardcoded defaults.

### Frontend

Vanilla HTML/CSS/JS in `frontend/`. The JS sends POST to `/api/query` and GET to `/api/courses`. Assistant messages are rendered as markdown via Marked.js. Sources are shown in a collapsible section. Static files are served by FastAPI's `StaticFiles` mount at `/`.

### Document Format

Course files in `docs/` must follow this structure:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[content...]

Lesson 1: [title]
[content...]
```

### Key Configuration (config.py)

- Model: `claude-sonnet-4-20250514`, temperature 0, max 800 tokens
- Embeddings: `all-MiniLM-L6-v2` (384-dim) via sentence-transformers
- Chunking: 800 chars with 100 char overlap
- Search: top 5 results
- ChromaDB persists to `./chroma_db` (relative to backend/)
