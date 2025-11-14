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

## Agent Workflow Pipeline

### Overview
The core agentic workflow that uses task analysis, planning, tool execution, and reflection to solve complex tasks autonomously. Implements a ReAct-style (Reasoning + Acting) pattern with adaptive complexity handling.

### Architecture Diagram
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AGENT WORKFLOW PIPELINE                                 │
│                    (ReAct Pattern with Reflection)                           │
└─────────────────────────────────────────────────────────────────────────────┘

User Task Request
    │
    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ PHASE 1: TASK ANALYSIS                                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────┐                                             │
│  │ 1. Complexity Assessment│ ◄── Prompts:                                 │
│  ├─────────────────────────┤       - analyze_task_system.md              │
│  │ - Classify LOW/MED/HIGH │       - analyze_task_user.md                │
│  │ - Detect small talk     │                                             │
│  │ - Assess transmission   │     Input:                                  │
│  │   needs                │       - task: user request                  │
│  └──────────┬──────────────┘       - context: background                 │
│             │                      - agent_prompt: role hints            │
│             ▼                      - tools_desc: available tools         │
│  ┌─────────────────────────┐                                             │
│  │ 2. Adaptive Planning    │     Output Varies by Complexity:            │
│  ├─────────────────────────┤                                             │
│  │ LOW (≤50 words):       │     - LOW: 1-2 step direct approach         │
│  │   - 1-2 steps          │     - MEDIUM: 3-5 steps, probes, success    │
│  │   - Direct execution    │       criteria (80-150 words)               │
│  │                        │     - HIGH: 5-8 steps, dependencies,        │
│  │ MEDIUM (80-150 words): │       reflection hooks (150-250 words)      │
│  │   - 3-5 steps          │                                             │
│  │   - Uncertainty probes  │     Includes:                               │
│  │   - Success criteria    │     - Objective & scope                     │
│  │   - Source plan        │     - Step-by-step plan                     │
│  │                        │     - Success/failure criteria              │
│  │ HIGH (150-250 words):  │     - Task transmission (if multi-agent)    │
│  │   - 5-8 steps          │                                             │
│  │   - Dependencies       │                                             │
│  │   - Reflection hooks    │                                             │
│  │   - Source plan        │                                             │
│  └──────────┬──────────────┘                                             │
└─────────────┼────────────────────────────────────────────────────────────┘
              │
              │ [Task Analysis Output: Plan + Success Criteria]
              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ PHASE 2: PLANNING & TOOL SELECTION                                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────┐                                             │
│  │ 3. Planning Agent       │ ◄── Prompt: next_step.md                    │
│  ├─────────────────────────┤       (JSON tool call format)               │
│  │ - Review task analysis  │                                             │
│  │ - Select appropriate    │     Input:                                  │
│  │   tools                │       - task_analysis: from Phase 1         │
│  │ - Decide sequential vs  │       - desc: tool JSON schemas             │
│  │   parallel execution    │       - today: current date                 │
│  └──────────┬──────────────┘                                             │
│             │                    Available Tools:                         │
│             ▼                    - Retrieval: KB search                   │
│  ┌─────────────────────────┐    - LLM: Text generation                   │
│  │ 4. Generate Tool Calls  │    - Categorize: Classification             │
│  ├─────────────────────────┤    - Generate: Structured output            │
│  │ Parallel Execution:     │    - DeepDoc: Document parsing              │
│  │   [{                   │    - Message: Send output                   │
│  │     "name": "tool_a",  │    - WebSearch: Tavily/DuckDuckGo          │
│  │     "arguments": {...}  │    - SQL: Database queries                  │
│  │   },{                  │    - BaiduFanyi: Translation                │
│  │     "name": "tool_b",  │    - Wikipedia, GitHub, Email, etc.         │
│  │     "arguments": {...}  │    - complete_task: Finish execution        │
│  │   }]                   │                                             │
│  │                        │     Output:                                  │
│  │ OR Completion:         │     JSON array of tool calls OR             │
│  │   [{                   │     complete_task with final answer         │
│  │     "name":            │                                             │
│  │       "complete_task", │                                             │
│  │     "arguments":       │                                             │
│  │       {"answer": "..." }│                                             │
│  │   }]                   │                                             │
│  └──────────┬──────────────┘                                             │
└─────────────┼────────────────────────────────────────────────────────────┘
              │
              │ [Tool Call JSON Array]
              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ PHASE 3: EXECUTION                                                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────┐                                             │
│  │ 5. Tool Orchestration   │                                             │
│  ├─────────────────────────┤     Execution Modes:                        │
│  │ Sequential Mode:        │     - Sequential: Execute one-by-one        │
│  │   tool_a() → tool_b()   │     - Parallel: Execute concurrently        │
│  │                        │       (async/threading)                     │
│  │ Parallel Mode:         │                                             │
│  │   tool_a() ──┐         │     Result Collection:                      │
│  │   tool_b() ──┼─→ wait  │     For each tool call:                     │
│  │   tool_c() ──┘  all    │     {                                       │
│  └──────────┬──────────────┘       "name": tool_name,                    │
│             │                      "params": {...},                       │
│             ▼                      "result": output,                      │
│  ┌─────────────────────────┐       "status": success/error               │
│  │ 6. Result Summarization │     }                                       │
│  ├─────────────────────────┤                                             │
│  │ For each tool result:   │ ◄── Prompt: summary4memory.md              │
│  │   - Summarize 1-2 sent. │       (Condense to key facts)               │
│  │   - Extract status      │                                             │
│  │   - Capture constraints │     Input: tool name, params, result        │
│  └──────────┬──────────────┘     Output: "[Status] + [Outcome] +        │
│             │                            [Constraints]"                   │
└─────────────┼────────────────────────────────────────────────────────────┘
              │
              │ [Tool Results + Summaries]
              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ PHASE 4: REFLECTION                                                       │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────────────────────┐                                             │
│  │ 7. Complexity Scoring   │ ◄── Prompt: reflect.md                      │
│  ├─────────────────────────┤       (Adaptive reflection)                 │
│  │ Assess 4 dimensions:    │                                             │
│  │ - Scope Breadth (1-3)   │     Complexity Matrix:                      │
│  │ - Data Dependency (1-3) │     Each dimension scored 1-3:              │
│  │ - Decision Points (1-3) │     - Total 4-12 points                     │
│  │ - Risk Level (1-3)      │                                             │
│  │                        │     Input:                                  │
│  │ Total: 4-12 points     │     - goal: overall objective               │
│  └──────────┬──────────────┘     - tool_calls: executed tools w/results  │
│             │                                                             │
│             ▼                                                             │
│  ┌─────────────────────────┐                                             │
│  │ 8. Situational Reflection│    Reflection Depth by Score:              │
│  ├─────────────────────────┤                                             │
│  │ Simple (4-5 pts):       │    4-5: 50-100 words                        │
│  │   ~50-100 words        │      - Completion status                    │
│  │   - Status + next step  │      - Immediate next step                  │
│  │                        │                                             │
│  │ Moderate (6-8 pts):    │    6-8: 100-200 words                       │
│  │   ~100-200 words       │      - Core details                         │
│  │   - Core details + risks│      - Main risks                           │
│  │                        │      - Information adequacy                 │
│  │ Complex (9-12 pts):    │                                             │
│  │   ~200-300 words       │    9-12: 200-300 words                      │
│  │   - Full analysis      │      - Full analysis                        │
│  │   - Alternatives       │      - Alternative strategies               │
│  │   - Escalation triggers │      - Escalation conditions                │
│  └──────────┬──────────────┘                                             │
│             │                                                             │
│             ▼                    Output Sections:                         │
│  ┌─────────────────────────┐    1. Goal Achievement Status               │
│  │ 9. Reflection Output    │    2. Step Completion Check                 │
│  ├─────────────────────────┤    3. Information Adequacy                  │
│  │ 1. Goal achievement     │    4. Critical Observations                 │
│  │ 2. Completed steps      │    5. Next-Step Recommendations             │
│  │ 3. Info adequacy        │    6. Task Transmission (if needed)         │
│  │ 4. Critical observations│                                             │
│  │ 5. Next-step recommend. │                                             │
│  │ 6. Task transmission    │                                             │
│  │    (if multi-agent)     │                                             │
│  └──────────┬──────────────┘                                             │
└─────────────┼────────────────────────────────────────────────────────────┘
              │
              │ [Reflection Analysis]
              ▼
         ┌──────────┐
         │ Decision │
         └────┬─────┘
              │
      ┌───────┴────────┐
      ▼                ▼
┌──────────┐      ┌──────────────┐
│ Complete │      │ Loop Back to │
│ Task     │      │ Phase 2      │
└──────────┘      └──────────────┘
                  (Plan next tools)
```

### Data Flow Detail

**Phase 1: Task Analysis**

**Step 1-2: Complexity Assessment & Adaptive Planning**
- **Function**: `analyze_task(chat_mdl, prompt, task_name, tools_description)`
- **Complexity Classification**:
  - **LOW**: "Set a reminder", "What's 2+2?", "Hello" (small talk)
  - **MEDIUM**: "Research competitors and create report"
  - **HIGH**: "Plan marketing campaign across 5 channels with budget optimization"
- **Output Example (MEDIUM)**:
  ```
  Complexity: MEDIUM

  Objective: Research competitors and create comparison report
  Intent & Scope: Analyze top 3 competitors, compare features, pricing, market position

  Plan:
  1. Search web for competitor list in industry
  2. For each competitor: retrieve company info, products, pricing
  3. Extract key features from product pages
  4. Compare features in structured format
  5. Generate markdown report with findings

  Uncertainty & Probes:
  - Probe: Are there established industry reports? Stop if comprehensive source found.

  Success Criteria:
  - Report contains ≥3 competitors
  - Feature comparison table included
  - Pricing data current (within 6 months)

  Source Plan:
  - Web search for competitor lists (industry publications)
  - Company websites for official data
  - Product documentation for features
  ```

**Phase 2: Planning & Tool Selection**

**Step 3-4: Planning Agent & Tool Call Generation**
- **Function**: `next_step(chat_mdl, history, tools_description, task_desc)`
- **Decision Logic**:
  - Reviews task analysis plan
  - Selects tools matching current step
  - Determines if steps can run in parallel
- **Output Example** (Parallel execution):
  ```json
  [{
    "name": "WebSearch",
    "arguments": {
      "query": "top software competitors 2025",
      "max_results": 5
    }
  },{
    "name": "Retrieval",
    "arguments": {
      "query": "competitor analysis framework",
      "top_k": 3
    }
  }]
  ```

**Phase 3: Execution**

**Step 5-6: Tool Orchestration & Summarization**
- **Orchestration**: `agent/component/agent_with_tools.py`
- **Parallel Execution**: Uses async/threading for independent tools
- **Result Processing**:
  ```python
  # For each tool result:
  summary = tool_call_summary(chat_mdl, name, params, result)

  # Example summary:
  # "Success: Found 5 competitor companies including Acme Corp,
  #  BetaSoft, and GammaTech."
  ```

**Phase 4: Reflection**

**Step 7-9: Complexity Scoring & Reflection**
- **Function**: `reflect(chat_mdl, history, tool_call_res)`
- **Complexity Dimensions** (1-3 each):
  - Scope Breadth: Single-step (1) | Multi-step (2) | Multi-domain (3)
  - Data Dependency: Self-contained (1) | External (2) | Multiple sources (3)
  - Decision Points: Linear (1) | Few branches (2) | Complex logic (3)
  - Risk Level: Low (1) | Medium (2) | High (3)
- **Example Reflection** (8 points, Moderate):
  ```
  Complexity Score: 8 (Scope:2, Data:3, Decisions:2, Risk:1)

  Goal Achievement Status:
  - Partially achieved: Found 5 competitors, but pricing data incomplete
  - Gap: Only 2 of 5 competitors have current pricing

  Step Completion Check:
  - ✓ Completed: Web search, competitor identification
  - ✓ Completed: Feature extraction for Acme Corp, BetaSoft
  - ✗ Pending: Pricing data for GammaTech, DeltaInc, EpsilonSys

  Information Adequacy:
  - Current data sufficient for feature comparison
  - Need additional source for pricing (missing for 3 companies)

  Critical Observations:
  - Risk: Incomplete pricing may affect report credibility

  Next-Step Recommendations:
  - Action: Execute targeted search for missing pricing data
  - Alternative: Mark pricing as "Not publicly available" if search fails
  - Tools needed: WebSearch with specific pricing queries
  ```

**Loop Decision**:
- **If goal achieved**: Call `complete_task` with final answer
- **If more work needed**: Return to Phase 2 with reflection insights

### Key Prompts Orchestration

1. **analyze_task_system.md + analyze_task_user.md** → Task analysis
2. **next_step.md** → Tool selection (uses analysis output)
3. **summary4memory.md** → Summarize each tool result
4. **reflect.md** → Evaluate progress (uses tool summaries)
5. **rank_memory.md** → Prioritize memories (optional)
6. **Loop**: next_step uses reflection → new tools → reflect → repeat

### Memory Management

**Short-term Memory:**
- Tool call summaries (1-2 sentences each)
- Kept for current task context

**Memory Ranking** (Optional):
- **Function**: `rank_memories(chat_mdl, goal, sub_goal, tool_call_summaries)`
- **Purpose**: Prioritize most relevant information
- **Output**: `[2, 0, 1]` - indices sorted by relevance

### Configuration Points
- **Max Iterations**: Prevent infinite loops (default: 10)
- **Complexity Thresholds**: Adjust word limits for analysis
- **Tool Timeout**: Per-tool execution limits
- **Reflection Triggers**: When to force reflection vs. auto-continue

---

