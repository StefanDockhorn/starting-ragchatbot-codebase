# GEMINI.md - Course Materials RAG System

This project is a full-stack Retrieval-Augmented Generation (RAG) system designed to provide intelligent, context-aware answers based on course materials.

## Project Overview

- **Purpose**: A RAG-based chatbot that allows users to query academic course materials (PDF, TXT, DOCX) and receive AI-generated responses grounded in the source documents.
- **Backend Architecture**:
    - **FastAPI**: Serves as the web framework and API layer.
    - **ChromaDB**: Vector database for storing and querying document embeddings.
    - **Anthropic Claude**: Large Language Model (LLM) used for generating answers.
    - **Sentence-Transformers**: Generates embeddings using the `all-MiniLM-L6-v2` model.
    - **RAG Orchestrator**: The `RAGSystem` class in `backend/rag_system.py` coordinates document processing, vector storage, AI generation, and session management.
- **Frontend**: A single-page application (SPA) built with HTML, CSS, and Vanilla JavaScript, served by the FastAPI backend.

## Building and Running

### Prerequisites
- **Python**: 3.13 or higher.
- **uv**: Modern Python package manager.
- **Anthropic API Key**: Required for Claude-powered responses.

### Key Commands

- **Install Dependencies**:
  ```bash
  uv sync
  ```
- **Environment Setup**:
  Create a `.env` file in the root directory with your API key:
  ```bash
  ANTHROPIC_API_KEY=your_key_here
  ```
- **Run the Application (Shell Script)**:
  ```bash
  chmod +x run.sh
  ./run.sh
  ```
- **Manual Start (Backend and Frontend)**:
  ```bash
  cd backend
  uv run uvicorn app:app --reload --port 8000
  ```
  The frontend will be automatically served at `http://localhost:8000`.

## Application Flow

```text
  [ INITIALIZATION ]               [ USER QUERY FLOW ]
  
   +--------------+                 +------------------+
   | docs/ Folder |                 |   Frontend UI    |
   +------+-------+                 +--------+---------+
          |                                  |
          v                                  v (POST /api/query)
   +--------------+                 +------------------+
   |  Doc Processor|                 |   FastAPI App    |
   +------+-------+                 +--------+---------+
          |                                  |
          v                                  v (orchestrate)
   +--------------+                 +------------------+
   | Vector Store |<---------------+|    RAG System    |
   | (ChromaDB)   |       (search)  +--------+---------+
   +--------------+                          |
                                             v (generate)
                                    +------------------+
                                    | Anthropic Claude |
                                    | (LLM + Tools)    |
                                    +------------------+
```

### 1. Initialization & Data Indexing (Startup)
1. **Server Launch**: The FastAPI backend starts and initializes the `RAGSystem` orchestrator.
2. **Document Discovery**: The system scans the `docs/` directory for `.txt`, `.pdf`, and `.docx` files.
3. **Structured Processing**: `DocumentProcessor` parses each file, extracting course metadata (Title, Instructor, Link) and splitting content into chunks based on "Lesson" markers.
4. **Vector Indexing**:
   - Course metadata is stored in the `course_catalog` ChromaDB collection.
   - Text chunks are converted into 384-dimensional embeddings (using `all-MiniLM-L6-v2`) and stored in the `course_content` collection.

### 2. User Interaction (Frontend)
1. **Dashboard Loading**: On page load, the frontend fetches course statistics and titles from `/api/courses` to populate the UI.
2. **Query Submission**: The user submits a question via the chat interface.
3. **API Request**: The frontend sends a POST request to `/api/query` containing the user's string and a `session_id` for context retention.

### 3. Intelligence & Retrieval (Backend)
1. **Context Management**: `SessionManager` retrieves up to the last 2 exchanges to provide conversation context.
2. **AI Analysis**: Anthropic's Claude receives the query and decides whether to invoke the `search_course_content` tool based on the system prompt.
3. **Semantic Search**:
   - If a search is triggered, the `CourseSearchTool` first resolves any mentioned course names against the `course_catalog`.
   - It then performs a vector similarity search in `course_content`, applying filters for specific courses or lessons if identified.
4. **Response Synthesis**: Claude synthesizes the retrieved chunks into a concise, direct answer.
5. **Source Attribution**: The system tracks the specific lessons and links used in the search and returns them as metadata.

### 4. Final Delivery
1. **Response Rendering**: The backend returns the answer (Markdown) and structured sources.
2. **UI Update**: The frontend renders the Markdown response and displays clickable source links in a collapsible menu.

## Development Conventions

### Backend Structure (`backend/`)
- `app.py`: Main FastAPI application, defines API endpoints and static file serving.
- `rag_system.py`: The core orchestrator that ties all RAG components together.
- `document_processor.py`: Handles reading files (TXT, PDF, DOCX) and chunking text.
- `vector_store.py`: Manages interactions with ChromaDB.
- `ai_generator.py`: Interfaces with the Anthropic API.
- `session_manager.py`: Manages user conversation history.
- `config.py`: Centralized configuration using `Config` dataclass and environment variables.

### Data Management
- Course materials are stored in the `docs/` directory.
- On startup, the system automatically scans the `docs/` folder and indexes any new documents into ChromaDB.
- The system supports multiple sessions, maintaining conversation history for better context in follow-up questions.

### AI and Search
- **Chunking**: Uses a default chunk size of 800 characters with 100-character overlap.
- **Search**: Retrieves the top 5 relevant chunks per query.
- **Model**: Uses `claude-sonnet-4-20250514` for high-quality responses.
