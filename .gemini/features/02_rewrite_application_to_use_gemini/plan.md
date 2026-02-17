# Refactoring Plan: Migrating from Anthropic to Google Agent Development Kit (ADK)

This plan outlines the steps required to refactor the Course Materials RAG System to use Google's Agent Development Kit (ADK) and Gemini models, replacing the current Anthropic Claude implementation.

## 1. Dependency Updates

### 1.1 `pyproject.toml`
- **Remove**: `anthropic`
- **Add**: `google-adk`
- **Add**: `google-genai` (if not already included as a dependency of ADK)

### 1.2 Environment Variables
- Replace `ANTHROPIC_API_KEY` with `GOOGLE_API_KEY`.
- Update `.env.example` to reflect this change.

## 2. Refactor Search Tools (`backend/search_tools.py`)

Currently, `CourseSearchTool` inherits from a custom `Tool` base class and returns a manual definition. ADK simplifies this.

- **Option A (Simplified)**: Convert `CourseSearchTool.execute` into a standalone function or use `google.adk.tools.FunctionTool`.
- **Option B (Structured)**: Keep the class but ensure it complies with ADK's expectation for tools. ADK can automatically infer schemas from function docstrings or Pydantic models.

### Proposed Change:
```python
from google.adk.tools import FunctionTool

def search_course_content(query: str, course_name: str = None, lesson_number: int = None) -> str:
    """
    Search course materials with smart course name matching and lesson filtering.
    
    Args:
        query: What to search for in the course content.
        course_name: Course title (partial matches work).
        lesson_number: Specific lesson number to search within.
    """
    # ... existing implementation logic using VectorStore ...
```

## 3. Refactor AI Generator (`backend/ai_generator.py`)

The `AIGenerator` class currently handles Anthropic's message creation and manual tool execution loop. ADK's `Agent` and `InMemoryRunner` will replace this logic.

### 3.1 New `ADKGenerator` Implementation:
- Initialize `google.adk.Agent` with Gemini models (e.g., `gemini-2.0-flash`).
- Set `SYSTEM_PROMPT` as the agent's `instruction`.
- Use `InMemoryRunner` to manage execution and tool calls automatically.

### Key Snippet:
```python
from google.adk.agents import Agent
from google.adk.runners import InMemoryRunner

class ADKGenerator:
    def __init__(self, api_key: str, model: str):
        # ADK usually picks up GOOGLE_API_KEY from environment
        self.agent = Agent(
            name="CourseAssistant",
            model=model,
            instruction=SYSTEM_PROMPT,
            tools=[search_course_content_wrapped]
        )
        self.runner = InMemoryRunner(agent=self.agent)

    async def generate_response(self, query: str, session_id: str):
        # Use runner.run_async to get responses
        # Handle streaming if desired, or just collect the final text
```

## 4. Update RAG Orchestrator (`backend/rag_system.py`)

- Update `__init__` to use the new `ADKGenerator`.
- Simplify `query` method: ADK's `InMemoryRunner` handles conversation history and tool execution loops internally if configured correctly.
- Ensure `tool_manager.get_last_sources()` still works (might need to store sources in a thread-local or shared state during tool execution).

## 5. Update FastAPI App (`backend/app.py`)

- Since ADK is naturally asynchronous, update the API endpoints and `RAGSystem.query` calls to be `async` if they aren't already.

## 6. Verification Steps

1. **Verify Indexing**: Ensure documents are still indexed correctly into ChromaDB.
2. **Test General Queries**: Ask "What can you do?" to ensure Gemini responds without searching.
3. **Test Course Queries**: Ask "What is lesson 1 of MCP about?" to verify `CourseSearchTool` is triggered and results are synthesized.
4. **Verify Sources**: Check that the collapsible "Sources" section in the UI still displays the correct links.

## 7. Risks & Considerations

- **Model Naming**: Ensure correct Gemini model identifiers are used (e.g., `gemini-2.0-flash`).
- **Tool Schema**: Gemini might be more sensitive to tool descriptions than Claude; refine docstrings if needed.
- **Async Overhead**: Switching to `run_async` requires proper `await` handling throughout the backend.
