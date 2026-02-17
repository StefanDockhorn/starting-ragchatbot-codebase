# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Run the server** (from repo root):
```bash
./run.sh
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Manage dependencies** — always use `uv`, never `pip`:
```bash
uv sync           # install/sync all dependencies
uv add <package>  # add a new dependency
uv remove <package> # remove a dependency
```

**Environment setup** — create `.env` in the repo root:
```
ANTHROPIC_API_KEY=your-anthropic-api-key-here
```

There is no test suite in this project.

## Architecture

This is a full-stack RAG (Retrieval-Augmented Generation) chatbot. The FastAPI server is launched from inside `backend/`, so all relative paths in the Python code are relative to `backend/` — notably `../docs` for course files, `../frontend` for static assets, and `./chroma_db` for the vector DB.

### Request lifecycle

1. **Browser** sends `POST /api/query {query, session_id?}` to FastAPI (`app.py`)
2. **`RAGSystem`** (`rag_system.py`) is the central orchestrator — it holds references to all components and coordinates them
3. **`SessionManager`** looks up in-memory conversation history (max 2 exchanges, not persisted across restarts)
4. **`AIGenerator`** makes a first Claude API call with tool definitions attached
5. **Claude** may call the `search_course_content` tool — if so, `AIGenerator` hands off to `ToolManager` → `CourseSearchTool` → `VectorStore`
6. **`VectorStore`** optionally resolves a fuzzy course name via the `course_catalog` collection, then vector-searches the `course_content` collection using `SentenceTransformer` embeddings
7. Tool results are returned to Claude in a second API call, which produces the final answer
8. Sources (course + lesson labels) are collected from `CourseSearchTool.last_sources` and returned to the browser alongside the answer

### ChromaDB collections

Two persistent collections live in `backend/chroma_db/`:

- **`course_catalog`** — one entry per course; id = course title string; used for fuzzy course-name resolution via semantic search
- **`course_content`** — one entry per text chunk; id = `CourseTitle_<chunk_index>`; filtered by `course_title` and/or `lesson_number` metadata fields

Course title acts as the unique identifier throughout the system. Deduplication at ingestion is based on comparing the parsed title against `get_existing_course_titles()`.

### Course document format

Files in `docs/` must follow this exact structure:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<lesson content...>

Lesson 1: <lesson title>
...
```

`DocumentProcessor` parses lessons by the `Lesson N:` marker, chunks content sentence-by-sentence (chunk size 800 chars, 100-char overlap), and stores chunks with lesson context prepended.

### Key configuration (`backend/config.py`)

| Setting | Default | Purpose |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Model used for all AI responses |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | SentenceTransformer model for embeddings |
| `CHUNK_SIZE` | 800 | Max chars per text chunk |
| `MAX_RESULTS` | 5 | Max vector search results returned to Claude |
| `MAX_HISTORY` | 2 | Conversation exchanges kept per session |
