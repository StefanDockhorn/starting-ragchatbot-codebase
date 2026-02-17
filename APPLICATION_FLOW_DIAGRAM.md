# Application Flow Diagram

## System Architecture Overview

```mermaid
graph TB
    subgraph Frontend["Frontend (browser)"]
        UI[index.html / script.js]
    end

    subgraph API["FastAPI Server (app.py)"]
        QE[POST /api/query]
        CE[GET /api/courses]
    end

    subgraph RAG["RAG System (rag_system.py)"]
        RS[RAGSystem Orchestrator]
    end

    subgraph Components["Core Components"]
        DP[DocumentProcessor]
        VS[VectorStore]
        AG[AIGenerator]
        SM[SessionManager]
        TM[ToolManager]
    end

    subgraph Storage["Persistence"]
        CDB[(ChromaDB)]
        DOCS[docs/*.txt files]
    end

    subgraph External["External Services"]
        CLAUDE[Anthropic Claude API]
        ST[SentenceTransformer Model]
    end

    UI -->|HTTP POST| QE
    UI -->|HTTP GET| CE
    QE --> RS
    CE --> RS
    RS --> DP
    RS --> VS
    RS --> AG
    RS --> SM
    RS --> TM
    VS <-->|embed + store/query| CDB
    VS <-->|embeddings| ST
    AG <-->|chat completions| CLAUDE
    DP -->|parse .txt files| DOCS
```

---

## 1. Startup / Document Ingestion Flow

Runs once when the server starts (`@app.on_event("startup")`).

```mermaid
sequenceDiagram
    participant Server as FastAPI (app.py)
    participant RAG as RAGSystem
    participant DP as DocumentProcessor
    participant VS as VectorStore
    participant ST as SentenceTransformer
    participant DB as ChromaDB

    Server->>RAG: add_course_folder("../docs")
    RAG->>VS: get_existing_course_titles()
    VS->>DB: course_catalog.get()
    DB-->>VS: existing titles
    VS-->>RAG: [list of already-loaded courses]

    loop for each .txt file in /docs
        RAG->>DP: process_course_document(file_path)
        DP->>DP: read_file() → raw text
        DP->>DP: parse header (title, link, instructor)
        DP->>DP: split into lessons by "Lesson N:" markers
        DP->>DP: chunk_text() → sentence-based chunks with overlap
        DP-->>RAG: Course object + List[CourseChunk]

        alt course not yet in ChromaDB
            RAG->>VS: add_course_metadata(course)
            VS->>DB: course_catalog.add(title, instructor, lessons_json)
            RAG->>VS: add_course_content(chunks)
            VS->>ST: embed chunk texts
            ST-->>VS: vectors
            VS->>DB: course_content.add(documents, embeddings, metadata)
        else course already exists
            RAG->>RAG: skip (no duplicate ingestion)
        end
    end

    Server->>Server: log "Loaded N courses with M chunks"
```

---

## 2. Query / Response Flow

Triggered on every `POST /api/query` from the user.

```mermaid
sequenceDiagram
    participant Browser as Browser (script.js)
    participant API as FastAPI (app.py)
    participant RAG as RAGSystem
    participant SM as SessionManager
    participant AG as AIGenerator
    participant TM as ToolManager
    participant CST as CourseSearchTool
    participant VS as VectorStore
    participant DB as ChromaDB
    participant ST as SentenceTransformer
    participant Claude as Anthropic Claude API

    Browser->>API: POST /api/query {query, session_id?}
    API->>SM: create_session() if no session_id
    SM-->>API: session_id

    API->>RAG: query(user_query, session_id)

    RAG->>SM: get_conversation_history(session_id)
    SM-->>RAG: formatted prior messages (or None)

    RAG->>AG: generate_response(prompt, history, tools)

    Note over AG: Build system prompt + history context
    AG->>Claude: messages.create(model, system, messages, tools)

    alt Claude decides to use search tool
        Claude-->>AG: stop_reason="tool_use" + tool call params
        AG->>TM: execute_tool("search_course_content", query, course_name?, lesson_number?)
        TM->>CST: execute(query, course_name, lesson_number)

        CST->>VS: search(query, course_name, lesson_number)

        opt course_name provided
            VS->>DB: course_catalog.query(course_name) → resolve fuzzy match
            DB-->>VS: matched course title
        end

        VS->>ST: embed query text
        ST-->>VS: query vector
        VS->>DB: course_content.query(vector, filters)
        DB-->>VS: top-N matching chunks + metadata

        VS-->>CST: SearchResults
        CST->>CST: format results + track sources
        CST-->>TM: formatted context string
        TM-->>AG: tool result

        AG->>Claude: messages.create(model, system, messages + tool_result)
        Claude-->>AG: final answer text

    else Claude answers directly (general knowledge)
        Claude-->>AG: answer text
    end

    AG-->>RAG: response string

    RAG->>TM: get_last_sources()
    TM-->>RAG: ["Course A - Lesson 2", ...]
    RAG->>TM: reset_sources()

    RAG->>SM: add_exchange(session_id, query, response)

    RAG-->>API: (response, sources)
    API-->>Browser: {answer, sources, session_id}

    Browser->>Browser: render answer + source chips in UI
```

---

## 3. ChromaDB Collections Layout

The vector store uses two separate collections:

```
ChromaDB (persistent on disk)
│
├── course_catalog          ← one document per course
│   ├── id: "Course Title"
│   ├── document: "Course Title"   (used for fuzzy name resolution)
│   └── metadata:
│       ├── title
│       ├── instructor
│       ├── course_link
│       ├── lessons_json   (serialised array of lesson titles + links)
│       └── lesson_count
│
└── course_content          ← one document per chunk
    ├── id: "Course_Title_<chunk_index>"
    ├── document: "Lesson N content: ..."  (the actual text chunk)
    └── metadata:
        ├── course_title
        ├── lesson_number
        └── chunk_index
```

---

## 4. AI Tool Execution Loop (detail)

```mermaid
flowchart TD
    A[User query arrives] --> B[Build prompt + inject conversation history]
    B --> C[Call Claude with tool definitions]
    C --> D{stop_reason?}

    D -->|end_turn| E[Return direct text answer]
    D -->|tool_use| F[Extract tool name + input params]

    F --> G[ToolManager.execute_tool]
    G --> H[CourseSearchTool.execute]
    H --> I{course_name given?}

    I -->|yes| J[Semantic fuzzy match in course_catalog]
    I -->|no| K[Search all courses]

    J --> L[Build metadata filter]
    K --> L

    L --> M[Embed query via SentenceTransformer]
    M --> N[ChromaDB vector similarity search]
    N --> O[Format top-N chunks with course+lesson headers]
    O --> P[Store sources list for UI]

    P --> Q[Return context string to Claude]
    Q --> R[Second Claude call — no tools, generate final answer]
    R --> E
```
