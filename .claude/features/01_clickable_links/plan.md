# Plan: Clickable Source Links in Chat UI

## Context
The chat UI shows source citations (e.g. "Course Title - Lesson 3") as plain text. The user wants each source to become a clickable link opening the lesson video in a new tab. Lesson links already exist in `Lesson.lesson_link` (parsed by `DocumentProcessor`) but are currently not stored in the `course_content` ChromaDB chunk metadata — only in the `course_catalog` collection's `lessons_json` field. This plan propagates `lesson_link` down through the chunk pipeline so it's available at search/format time, then updates the API and frontend to render clickable links.

---

## Implementation Steps

### 1. `backend/models.py` — Add `lesson_link` to `CourseChunk`

Add an optional field:
```python
class CourseChunk(BaseModel):
    content: str
    course_title: str
    lesson_number: Optional[int] = None
    chunk_index: int
    lesson_link: Optional[str] = None   # ← add this
```

### 2. `backend/document_processor.py` — Pass `lesson_link` when creating chunks

In both the mid-loop block (lines ~190–196) and the final-lesson block (lines ~236–242), pass `lesson_link` to `CourseChunk`:
```python
course_chunk = CourseChunk(
    content=chunk_with_context,
    course_title=course.title,
    lesson_number=current_lesson,
    chunk_index=chunk_counter,
    lesson_link=lesson_link,    # ← add this
)
```
`lesson_link` is already tracked as a local variable (`lesson_link = None`, then set from `Lesson Link:` line), so no additional parsing needed.

### 3. `backend/vector_store.py` — Store `lesson_link` in chunk metadata

In `add_course_chunks()`, add `lesson_link` to the metadata dict (use `""` for None since ChromaDB requires string/int/bool values):
```python
metadatas = [{
    "course_title": chunk.course_title,
    "lesson_number": chunk.lesson_number,
    "chunk_index": chunk.chunk_index,
    "lesson_link": chunk.lesson_link or "",   # ← add this
} for chunk in chunks]
```

### 4. `backend/search_tools.py` — Include `lesson_link` in source tracking

In `_format_results`, extract `lesson_link` from metadata and store structured sources (dicts instead of plain strings):
```python
def _format_results(self, results: SearchResults) -> str:
    formatted = []
    sources = []

    for doc, meta in zip(results.documents, results.metadata):
        course_title = meta.get('course_title', 'unknown')
        lesson_num = meta.get('lesson_number')
        lesson_link = meta.get('lesson_link') or None  # "" → None

        header = f"[{course_title}"
        if lesson_num is not None:
            header += f" - Lesson {lesson_num}"
        header += "]"

        label = course_title
        if lesson_num is not None:
            label += f" - Lesson {lesson_num}"

        sources.append({"label": label, "url": lesson_link})  # ← structured

        formatted.append(f"{header}\n{doc}")

    self.last_sources = sources
    return "\n\n".join(formatted)
```

### 5. `backend/app.py` — Update `QueryResponse.sources` type

Change from `List[str]` to `List[Dict[str, Any]]`:
```python
from typing import List, Dict, Any

class QueryResponse(BaseModel):
    answer: str
    sources: List[Dict[str, Any]]   # {"label": str, "url": str | None}
    session_id: str
```

### 6. `frontend/script.js` — Render sources as clickable links

Replace the `sources.join(', ')` rendering with link-aware HTML. For each source:
- If `url` is non-empty: render `<a href="{url}" target="_blank" rel="noopener">{label}</a>`
- Otherwise: render `{label}` as plain text

```javascript
if (sources && sources.length > 0) {
    const sourceItems = sources.map(s => {
        if (s.url) {
            return `<a href="${escapeHtml(s.url)}" target="_blank" rel="noopener noreferrer">${escapeHtml(s.label)}</a>`;
        }
        return escapeHtml(s.label);
    });
    html += `
        <details class="sources-collapsible">
            <summary class="sources-header">Sources</summary>
            <div class="sources-content">${sourceItems.join(', ')}</div>
        </details>
    `;
}
```

---

## Critical Files
- `backend/models.py` — `CourseChunk` model
- `backend/document_processor.py` — chunk creation (two sites: mid-loop ~line 190, last-lesson ~line 236)
- `backend/vector_store.py` — `add_course_chunks()` metadata dict ~line 162
- `backend/search_tools.py` — `_format_results()` ~line 88
- `backend/app.py` — `QueryResponse` model ~line 43
- `frontend/script.js` — `addMessage()` sources rendering ~line 123

---

## Notes
- Existing ChromaDB data won't have `lesson_link` in chunk metadata. The existing `course_catalog` data with `lessons_json` is unaffected (we're not changing catalog logic). To populate lesson links, re-ingest documents — the user may need to clear `backend/chroma_db/` and restart the server.
- The URL is not rendered as visible text — only the label is shown as the link's anchor text, matching the "embedded invisibly" requirement.
- Duplicate sources per response should be deduplicated (same label+url) to avoid redundant links.

---

## Verification
1. Clear `backend/chroma_db/` (or run with a fresh DB)
2. Start the server: `cd backend && uv run uvicorn app:app --reload --port 8000`
3. Re-ingest courses by visiting `http://localhost:8000` (ingestion runs on startup)
4. Send a query in the chat UI and expand "Sources"
5. Confirm sources appear as clickable labels (no raw URL text visible), each opening the lesson in a new tab
6. Confirm sources without lesson links render as plain text labels
