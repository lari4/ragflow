# RAGFlow Agent Pipelines and Workflows

This document provides comprehensive documentation of all agent workflows, processing pipelines, and data flows in the RAGFlow application. Each pipeline is documented with ASCII diagrams, data flow descriptions, and prompt orchestration details.

---

## Table of Contents

1. [Basic RAG Pipeline](#basic-rag-pipeline)
2. [Agent Workflow Pipeline](#agent-workflow-pipeline)
3. [GraphRAG Pipeline](#graphrag-pipeline)
4. [Document Processing Pipeline](#document-processing-pipeline)
5. [Deep Research Agent Pipeline](#deep-research-agent-pipeline)
6. [Agentic Reasoning Pipeline](#agentic-reasoning-pipeline)

---

## Basic RAG Pipeline

### Overview
The fundamental Retrieval-Augmented Generation pipeline that retrieves relevant documents and generates cited responses.

### Architecture Diagram
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          BASIC RAG PIPELINE                                  │
└─────────────────────────────────────────────────────────────────────────────┘

User Query
    │
    ▼
┌─────────────────────────┐
│ 1. Query Preprocessing  │ ◄── Prompt: full_question_prompt.md
├─────────────────────────┤       (Converts follow-up questions to standalone)
│ - Convert relative dates│
│ - Add conversation      │       Input: conversation history, today's date
│   context              │       Output: standalone question
│ - Rewrite follow-ups    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ 2. Query Expansion      │ ◄── Optional Prompts:
├─────────────────────────┤       - related_question.md (5-10 alternative queries)
│ - Generate related      │       - cross_languages_*.md (multilingual expansion)
│   questions            │
│ - Translate to target   │       Input: user query
│   languages            │       Output: expanded query set
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ 3. Metadata Filtering   │ ◄── Prompt: meta_filter.md
├─────────────────────────┤       (Generates filter conditions from NL query)
│ - Parse filter intent   │
│ - Generate filter JSON  │       Input: query, metadata keys, current date
│ - Handle date ranges    │       Output: JSON filter array [{"key":..,"op":..,"value":..}]
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ 4. Vector Retrieval     │
├─────────────────────────┤       No prompt - uses embedding model
│ - Embed query          │       Vector stores: Elasticsearch / Infinity
│ - Search vector DB      │
│ - Apply metadata filters│       Input: query embedding, filters
│ - Retrieve top-k chunks │       Output: ranked document chunks with IDs
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ 5. Context Assembly     │
├─────────────────────────┤       Formats context for LLM:
│ - Format chunks as      │       <context>
│   <context> blocks     │       ID: 0
│ - Add chunk IDs         │       └── Content: [chunk text]
│ - Manage token budget   │       ID: 1
└────────────┬────────────┘       └── Content: [chunk text]
             │                    </context>
             ▼
┌─────────────────────────┐
│ 6. Response Generation  │ ◄── Prompts:
├─────────────────────────┤       - ask_summary.md (basic KB answer)
│ - System: User prompts  │       - citation_prompt.md (add citations)
│ - Add context          │       - citation_plus.md (alternative citation)
│ - Generate response     │
│ - Stream output         │       Input: question, context, user prompts
└────────────┬────────────┘       Output: markdown answer with citations
             │
             ▼
┌─────────────────────────┐
│ 7. Citation Addition    │ ◄── Prompt: citation_prompt.md
├─────────────────────────┤       (Adds [ID:i] citations to response)
│ - Identify factual      │
│   claims               │       Rules:
│ - Add [ID:i] citations  │       - Cite quantitative data, dates, claims
│ - Validate citations    │       - Max 4 citations per sentence
└────────────┬────────────┘       - Place before punctuation
             │
             ▼
      Final Response
  (with citations)
```

### Data Flow

**Step 1: Query Preprocessing**
- **Function**: `full_question(tenant_id, llm_id, messages, language)`
- **Purpose**: Converts conversational queries into standalone questions
- **Example**:
  - Input: ["What is AI?", "When was it invented?"]
  - Output: "When was Artificial Intelligence invented?"

**Step 2: Query Expansion** (Optional)
- **Function**: `related_question(...)` or `cross_languages(...)`
- **Purpose**: Broadens search scope with alternative phrasings or multilingual queries
- **Example**:
  - Input: "benefits of electric vehicles"
  - Output: ["How do EVs impact environment?", "EV cost-effectiveness?", ...]

**Step 3: Metadata Filtering**
- **Function**: `gen_meta_filter(chat_mdl, meta_data, query)`
- **Purpose**: Converts natural language constraints to filter JSON
- **Example**:
  - Input: "documents from July, not blue color"
  - Output: `[{"key":"date","value":"2025-07-01","op":"≥"}, {"key":"color","value":"blue","op":"≠"}]`

**Step 4: Vector Retrieval**
- **Implementation**: `rag/svr/retriever.py`
- **Vector DB**: Elasticsearch or Infinity
- **Process**:
  1. Embed query using configured embedding model
  2. Similarity search (cosine/dot product)
  3. Apply metadata filters
  4. Return top-k chunks (default: 6-30)

**Step 5: Context Assembly**
- **Format**: Structured context with chunk IDs
- **Token management**: Respects LLM context window limits
- **Citation tracking**: Maps IDs to source documents

**Step 6: Response Generation**
- **Basic**: `ask_summary.md` prompt for simple Q&A
- **Enhanced**: User-defined system prompts + context
- **Streaming**: Supports real-time response generation

**Step 7: Citation Addition**
- **Function**: `citation_prompt(user_defined_prompts)` or `citation_plus(sources)`
- **Rules**:
  - Cite facts, not common knowledge
  - Format: `[ID:i][ID:j]` at sentence end
  - Max 4 citations per sentence

### Key Prompts Used
1. **full_question_prompt.md**: Query rewriting
2. **related_question.md**: Query expansion
3. **meta_filter.md**: Metadata filtering
4. **ask_summary.md**: Basic response generation
5. **citation_prompt.md**: Citation addition

### Configuration Points
- **Retrieval Settings**: Top-k, similarity threshold, reranking
- **Prompt Templates**: Customizable system prompts
- **Citation Mode**: Standard vs. Plus
- **Streaming**: Enabled/disabled

---

