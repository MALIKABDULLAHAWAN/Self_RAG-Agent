# Self-RAG University Course Advisory Agent

A production-quality Self-Reflective Retrieval-Augmented Generation (Self-RAG) system for XYZ National University that answers student queries using official university PDF documents.

## Features

- **Adaptive Retrieval**: Intelligently decides when document retrieval is needed
- **Strict Relevance Grading**: Filters out irrelevant documents before generation
- **Web Search Fallback**: Falls back to web search when retrieval fails
- **Hallucination Detection**: Self-corrects when unsupported claims are detected
- **Retry Loop**: Retries generation up to 3 times with stricter constraints
- **Execution Tracing**: Detailed logging of all decision points
- **Source Citations**: References to source documents in responses

## Architecture

```
┌─────────────────┐
│  route_query    │ ← Classifies query (retrieval needed?)
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────────┐
│retrieve│ │  direct   │
│_docs   │ │  response │
└───┬───┘ └───────────┘
    │
    ▼
┌─────────────┐
│grade_relevance│ ← Filters irrelevant docs
└──────┬──────┘
       │
  ┌────┴────┐
  ▼         ▼
┌───────┐ ┌──────────┐
│generate│ │ web_search│
│response│ │          │
└───┬───┘ └─────┬────┘
    │           │
    ▼           ▼
┌─────────────────────┐
│   hallucination_    │ ← Verifies answer accuracy
│      check          │
└────────┬────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌─────────┐ ┌──────────┐
│  retry  │ │  final   │
│generation│ │ response │
└────┬────┘ └──────────┘
     │
     └──────────────────┐
                        ▼
                 ┌────────────┐
                 │    END     │
                 └────────────┘
```

## Project Structure

```
project_root/
│
├── data/                           # University PDF documents
│   ├── CS_Department_Catalog.pdf
│   ├── EE_Department_Catalog.pdf
│   ├── BBA_Department_Catalog.pdf
│   ├── University_Academic_Policies.pdf
│   └── Faculty_Directory.pdf
│
├── vectorstore/                    # ChromaDB vector store (auto-created)
│
├── self_rag_agent.py              # Main CLI entry point
├── graph.py                       # LangGraph StateGraph implementation
├── tools.py                       # LangChain tools (@tool decorators)
├── ingest.py                      # PDF ingestion pipeline
├── prompts.py                     # LLM prompts
├── state.py                       # GraphState TypedDict
├── config.py                      # Configuration settings
├── utils.py                       # Utilities and logging
│
├── requirements.txt               # Python dependencies
├── .env.example                   # Environment variable template
├── README.md                      # This file
└── evaluation_results.md          # Test case documentation
```

## Setup Instructions

### 1. Install Dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure Environment Variables

Copy `.env.example` to `.env` and fill in your API keys:

```bash
cp .env.example .env
```

Edit `.env` and set at minimum **one** LLM provider key:
- `GROQ_API_KEY` (default — set `LLM_PROVIDER=groq`), **or**
- `OPENAI_API_KEY` (set `LLM_PROVIDER=openai`)

Optional:
- `TAVILY_API_KEY` — preferred web-search backend. If unset, the agent
  automatically falls back to scraping DuckDuckGo HTML (no key required).

Embeddings are computed **locally** with a bundled ONNX MiniLM-L6-v2 model
(`onnx_embeddings.py`), so no embeddings API key is needed.

### 3. Ingest Documents

Create the vector database from the PDFs:

```bash
python ingest.py
```

This will:
- Load all 5 PDF documents
- Split into intelligent chunks (800 tokens, 100 overlap)
- Create embeddings and store in ChromaDB
- Persist to `./vectorstore/`

### 4. Run the Agent

Interactive mode:

```bash
python self_rag_agent.py
```

Batch mode with specific queries:

```bash
python self_rag_agent.py "What are the prerequisites for CS401?" "Who teaches Machine Learning?"
```

## Usage Examples

### Example Queries

**Course Information:**
- "What are the prerequisites for CS401?"
- "How many credits is EE201?"
- "What does CS302 cover?"

**Faculty Information:**
- "Who teaches Machine Learning?"
- "What is Dr. Ahmed Hassan's email?"
- "Where is the CS department office?"

**Academic Policies:**
- "What is the withdrawal policy?"
- "How is GPA calculated?"
- "What are the tuition fees?"
- "When does the fall semester start?"

**General Questions (no retrieval needed):**
- "Hi!" or "Hello"
- "What does GPA stand for?"
- "Thank you!"

### Execution Trace Example

```
============================================================
Query: What are the prerequisites for CS401?
============================================================

[Route Decision]
Retrieval Needed: YES
Reason: The query asks about a university-specific course prerequisite.

[Retrieval]
Searching vector store for: 'What are the prerequisites for CS401?'
Retrieved 5 documents
  [1] CS_Department_Catalog.pdf
  [2] CS_Department_Catalog.pdf
  [3] Faculty_Directory.pdf

[Relevance Grading]
  Chunk 1 → relevant - Contains prerequisite information for CS401
  Chunk 2 → partially_relevant - Mentions CS401 but not prerequisites
  Chunk 3 → irrelevant - About faculty, not courses
Filtered to 1 relevant documents

[Response Generation]
Generating from retrieved documents
Generated: The prerequisites for CS401 Machine Learning are...

[Hallucination Check]
  ✓ Response is fully grounded in retrieved documents

[Final Response]
✓ Response finalized

============================================================
FINAL ANSWER
============================================================
The prerequisites for CS401 Machine Learning are CS201 Data Structures and Algorithms, MATH301 Linear Algebra, and STAT201 Statistics. The course is 4 credits and taught by Dr. Ahmed Hassan.

Sources:
  [1] CS_Department_Catalog.pdf [CS] (course_catalog)
============================================================
```

## Graph Explanation

### Nodes

1. **route_query**: Uses LLM to classify if retrieval is needed
2. **retrieve_documents**: Vector search using query embeddings
3. **grade_relevance**: LLM grades each retrieved document
4. **web_search**: Tavily web search as fallback
5. **generate_response**: Creates grounded answer from documents
6. **generate_direct_response**: Answers general questions directly
7. **hallucination_check**: Verifies answer against source documents
8. **retry_generation**: Regenerates with stricter constraints
9. **final_response**: Formats final answer with citations

### Conditional Edges

- `route_query` → either `retrieve_documents` or `generate_direct_response`
- `grade_relevance` → either `generate_response` or `web_search`
- `hallucination_check` → either `retry_generation` or `final_response`
- `retry_generation` → either back to `hallucination_check` or `final_response`

## State Structure

```python
class GraphState(TypedDict):
    query: str                          # User query
    needs_retrieval: bool               # Routing decision
    retrieval_reason: str               # Why retrieval is/isn't needed
    retrieved_docs: list[Document]      # Raw retrieval results
    filtered_docs: list[Document]       # After relevance grading
    relevance_grades: list[dict]        # Grading results
    web_results: list[dict]             # Web search fallback
    web_search_triggered: bool          # Was fallback used
    generated_response: str              # LLM generation
    hallucination_detected: bool       # Was hallucination found
    unsupported_claims: list[str]       # Specific bad claims
    retry_count: int                    # Current retry number
    final_answer: str                   # Validated response
    sources: list[dict]                # Citation information
    execution_trace: list[dict]         # Full execution log
```

## Configuration

All settings are in `config.py` and can be overridden via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `LLM_PROVIDER` | groq | `groq` or `openai` |
| `GROQ_API_KEY` | (required if provider=groq) | Groq API key |
| `OPENAI_API_KEY` | (required if provider=openai) | OpenAI API key |
| `TAVILY_API_KEY` | (optional) | Tavily search API (DuckDuckGo fallback works without it) |
| `LLM_MODEL` | llama-3.3-70b-versatile | LLM to use |
| `LLM_TEMPERATURE` | 0.0 | Creativity (0-1) |
| `EMBEDDING_MODEL` | all-MiniLM-L6-v2 (local ONNX) | Embedding model (no API key) |
| `CHUNK_SIZE` | 800 | Document chunk size |
| `CHUNK_OVERLAP` | 100 | Chunk overlap |
| `RETRIEVAL_K` | 5 | Documents to retrieve |
| `MAX_RETRIES` | 3 | Max retry attempts |
| `LOG_LEVEL` | INFO | Logging level |

## Troubleshooting

### Vector Store Not Found

```
ERROR: Vector store not found at: ./vectorstore
```

**Solution**: Run `python ingest.py` to create the knowledge base.

### LLM API Key Missing

```
Missing required configuration: GROQ_API_KEY
```

**Solution**: Set your Groq API key in the `.env` file (default provider),
or switch to OpenAI with `LLM_PROVIDER=openai` and `OPENAI_API_KEY=...`.

### No Relevant Documents

If you see "No relevant documents after grading" frequently, try:
1. Adjusting `CHUNK_SIZE` in config
2. Re-running `ingest.py` with different chunk settings
3. Checking that PDFs exist in `data/` directory

### Hallucination Detection Too Strict

If the agent frequently retries due to "hallucination":
1. The vector store may not contain the needed information
2. Try rephrasing your query to match document language
3. Check `evaluation_results.md` for expected behavior

## Technologies Used

- **Python 3.11+**: Core language
- **LangChain**: LLM framework and document processing
- **LangGraph**: Graph-based workflow orchestration
- **ChromaDB**: Vector database for embeddings
- **Groq** (default) **or OpenAI**: LLM provider (configurable)
- **ONNX runtime** (local, free): `all-MiniLM-L6-v2` embeddings — no API key
- **Tavily + DuckDuckGo**: Web search (DuckDuckGo fallback works without a key)
- **Pydantic**: Tool input validation
- **PyPDF**: PDF processing

## License

MIT License - For educational purposes at XYZ National University.

## Contact

For issues or questions:
- Academic Advising: advising@xyzuniversity.edu
- Registrar's Office: registrar@xyzuniversity.edu
#   S e l f _ R A G - A g e n t  
 