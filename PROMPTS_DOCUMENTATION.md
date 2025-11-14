# RAGFlow AI Prompts Documentation

This document provides comprehensive documentation of all AI prompts used in the RAGFlow application, organized by theme and functionality.

---

## Table of Contents

1. [RAG & Chat Prompts](#rag--chat-prompts)
2. [Agent Workflow Prompts](#agent-workflow-prompts)
3. [Document Processing Prompts](#document-processing-prompts)
4. [GraphRAG Prompts](#graphrag-prompts)
5. [Agent Template Prompts](#agent-template-prompts)
6. [Agentic Reasoning Prompts](#agentic-reasoning-prompts)

---

## RAG & Chat Prompts

### 1. Citation Prompt

**Purpose:** Adds citations to AI-generated responses using source references from the RAG context. Ensures factual accuracy by requiring citations for quantitative data, temporal claims, causal relationships, comparisons, technical definitions, direct attributions, predictions, and controversial claims.

**Location:** `rag/prompts/citation_prompt.md`

**Usage:** Applied to final responses to add `[ID:i]` style citations from retrieved documents.

```markdown
Based on the provided document or chat history, add citations to the input text using the format specified later.

# Citation Requirements:

## Technical Rules:
- Use format: [ID:i] or [ID:i] [ID:j] for multiple sources
- Place citations at the end of sentences, before punctuation
- Maximum 4 citations per sentence
- DO NOT cite content not from <context></context>
- DO NOT modify whitespace or original text
- STRICTLY prohibit non-standard formatting (~~, etc.)

## What MUST Be Cited:
1. **Quantitative data**: Numbers, percentages, statistics, measurements
2. **Temporal claims**: Dates, timeframes, sequences of events
3. **Causal relationships**: Claims about cause and effect
4. **Comparative statements**: Rankings, comparisons, superlatives
5. **Technical definitions**: Specialized terms, concepts, methodologies
6. **Direct attributions**: What someone said, did, or believes
7. **Predictions/forecasts**: Future projections, trend analyses
8. **Controversial claims**: Disputed facts, minority opinions

## What Should NOT Be Cited:
- Common knowledge (e.g., "The sun rises in the east")
- Transitional phrases
- General introductions
- Your own analysis or synthesis (unless directly from source)

# Comprehensive Examples:

## Example 1: Data and Statistics
<context>
ID: 45
└── Content: The global smartphone market grew by 7.8% in Q3 2024, with Samsung holding 20.6% market share and Apple at 15.8%.

ID: 46
└── Content: 5G adoption reached 1.5 billion users worldwide by October 2024.
</context>

USER: How is the smartphone market performing?

ASSISTANT:
The smartphone industry is showing strong recovery. The global smartphone market grew by 7.8% in Q3 2024 [ID:45]. Samsung leads with 20.6% market share, followed by Apple at 15.8% [ID:45]. This growth is partly driven by 5G technology, which reached 1.5 billion users worldwide by October 2024 [ID:46].

## Example 2: Causal Relationships and Predictions
<context>
ID: 78
└── Content: Rising interest rates have led to a 15% decline in housing starts. Economists predict this trend will continue through 2025.

ID: 79
└── Content: The Federal Reserve's aggressive monetary policy is the primary driver behind the real estate slowdown.
</context>

USER: What's happening in the real estate market?

ASSISTANT:
The real estate market is experiencing significant challenges. Rising interest rates have led to a 15% decline in housing starts [ID:78]. The Federal Reserve's aggressive monetary policy is the primary driver behind this real estate slowdown [ID:79]. Economists predict this trend will continue through 2025 [ID:78]. These factors suggest a prolonged period of market adjustment ahead.

## Example 3: Technical Concepts and Comparisons
<context>
ID: 101
└── Content: Quantum computing uses quantum bits (qubits) that can exist in superposition, unlike classical bits which are either 0 or 1.

ID: 102
└── Content: Current quantum computers can solve certain problems 158 million times faster than classical supercomputers.
</context>

USER: Explain quantum computing advantages.

ASSISTANT:
Quantum computing represents a fundamental shift in computational capability. Unlike classical computers, quantum computing uses quantum bits (qubits) that can exist in superposition, rather than being limited to just 0 or 1 [ID:101]. This unique property enables extraordinary performance gains. Current quantum computers can solve certain problems 158 million times faster than classical supercomputers [ID:102]. This technology promises to revolutionize fields requiring complex calculations.

## Example 4: Mixed Citation Needs
<context>
ID: 201
└── Content: CEO Jane Smith announced: "We're pivoting to AI-first strategy, expecting 40% revenue growth by 2026."

ID: 202
└── Content: Industry analysts remain skeptical, with Morgan Stanley rating the stock as "underperform."

ID: 203
└── Content: The company's AI division currently represents only 5% of total revenue.
</context>

USER: What's the company's new strategy?

ASSISTANT:
The company is undergoing a major strategic transformation. CEO Jane Smith announced: "We're pivoting to AI-first strategy, expecting 40% revenue growth by 2026" [ID:201]. However, this ambitious goal faces skepticism, as industry analysts remain doubtful, with Morgan Stanley rating the stock as "underperform" [ID:202]. The challenge is significant given that the company's AI division currently represents only 5% of total revenue [ID:203]. Success will depend on execution of this bold vision.

## Example 5: When NOT to Cite
<context>
ID: 301
└── Content: Paris is the capital of France. The Eiffel Tower was completed in 1889.
</context>

USER: Tell me about Paris.

ASSISTANT:
Paris is the capital of France. It's known for its rich history, culture, and architecture. The Eiffel Tower was completed in 1889 [ID:301]. The city attracts millions of tourists annually. Paris remains one of the world's most visited destinations.
(Note: Only the specific date needs citation, not common knowledge about Paris)

--- Examples END ---

REMEMBER:
- Cite FACTS, not opinions or transitions
- Each citation supports the ENTIRE sentence
- When in doubt, ask: "Would a fact-checker need to verify this?"
- Place citations at sentence end, before punctuation
- Format likes this is FORBIDDEN: [ID:0, ID:5, ID:...]. It MUST be seperated like, [ID:0][ID:5]...
```

---

### 2. Citation Plus

**Purpose:** Simplified citation prompt for adding citations to text that was generated based on provided sources but lacks proper references. Works with example-based learning.

**Location:** `rag/prompts/citation_plus.md`

**Usage:** Alternative citation approach using Jinja2 templates for sources and examples.

```markdown
You are an agent for adding correct citations to the given text by user.
You are given a piece of text within [ID:<ID>] tags, which was generated based on the provided sources.
However, the sources are not cited in the [ID:<ID>].
Your task is to enhance user trust by generating correct, appropriate citations for this report.

{{ example }}

<context>

{{ sources }}

</context>
```

---

### 3. Full Question Prompt

**Purpose:** Converts conversational follow-up questions into complete, standalone questions by incorporating context from chat history. Handles relative dates (yesterday, tomorrow) and converts them to absolute dates.

**Location:** `rag/prompts/full_question_prompt.md`

**Usage:** Query preprocessing for RAG retrieval to ensure context-independent searches.

```markdown
## Role
A helpful assistant.

## Task & Steps
1. Generate a full user question that would follow the conversation.
2. If the user's question involves relative dates, convert them into absolute dates based on today ({{ today }}).
   - "yesterday" = {{ yesterday }}, "tomorrow" = {{ tomorrow }}

## Requirements & Restrictions
- If the user's latest question is already complete, don't do anything — just return the original question.
- DON'T generate anything except a refined question.
{% if language %}
- Text generated MUST be in {{ language }}.
{% else %}
- Text generated MUST be in the same language as the original user's question.
{% endif %}

---

## Examples

### Example 1
**Conversation:**

USER: What is the name of Donald Trump's father?
ASSISTANT: Fred Trump.
USER: And his mother?

**Output:** What's the name of Donald Trump's mother?

---

### Example 2
**Conversation:**

USER: What is the name of Donald Trump's father?
ASSISTANT: Fred Trump.
USER: And his mother?
ASSISTANT: Mary Trump.
USER: What's her full name?

**Output:** What's the full name of Donald Trump's mother Mary Trump?

---

### Example 3
**Conversation:**

USER: What's the weather today in London?
ASSISTANT: Cloudy.
USER: What's about tomorrow in Rochester?

**Output:** What's the weather in Rochester on {{ tomorrow }}?

---

## Real Data

**Conversation:**

{{ conversation }}
```

---

### 4. Keyword Extraction Prompt

**Purpose:** Extracts the most important keywords/phrases from text content for search optimization and indexing. Ensures keywords maintain the original language.

**Location:** `rag/prompts/keyword_prompt.md`

**Usage:** Used for document indexing, query expansion, and semantic search enhancement.

```markdown
## Role
You are a text analyzer.

## Task
Extract the most important keywords/phrases of a given piece of text content.

## Requirements
- Summarize the text content, and give the top {{ topn }} important keywords/phrases.
- The keywords MUST be in the same language as the given piece of text content.
- The keywords are delimited by ENGLISH COMMA.
- Output keywords ONLY.

---

## Text Content
{{ content }}
```

---

### 5. Question Generation Prompt

**Purpose:** Generates important questions from text content to facilitate question-answering systems and content understanding. Questions should cover main content without overlapping meanings.

**Location:** `rag/prompts/question_prompt.md`

**Usage:** Creates question-answer pairs for knowledge base enrichment and testing.

```markdown
## Role
You are a text analyzer.

## Task
Propose {{ topn }} questions about a given piece of text content.

## Requirements
- Understand and summarize the text content, and propose the top {{ topn }} important questions.
- The questions SHOULD NOT have overlapping meanings.
- The questions SHOULD cover the main content of the text as much as possible.
- The questions MUST be in the same language as the given piece of text content.
- One question per line.
- Output questions ONLY.

---

## Text Content
{{ content }}
```

---

### 6. Related Questions Prompt

**Purpose:** Generates 5-10 related questions based on a user's query to expand search scope and improve retrieval relevance. Helps retrieve broader range of relevant documents from vector database.

**Location:** `rag/prompts/related_question.md`

**Usage:** Query expansion for multi-query retrieval strategies.

```markdown
# Role
You are an AI language model assistant tasked with generating **5-10 related questions** based on a user's original query.
These questions should help **expand the search query scope** and **improve search relevance**.

---

## Instructions

**Input:**
You are provided with a **user's question**.

**Output:**
Generate **5-10 alternative questions** that are **related** to the original user question.
These alternatives should help retrieve a **broader range of relevant documents** from a vector database.

**Context:**
Focus on **rephrasing** the original question in different ways, ensuring the alternative questions are **diverse but still connected** to the topic of the original query.
Do **not** create overly obscure, irrelevant, or unrelated questions.

**Fallback:**
If you cannot generate any relevant alternatives, do **not** return any questions.

---

## Guidance

1. Each alternative should be **unique** but still **relevant** to the original query.
2. Keep the phrasing **clear, concise, and easy to understand**.
3. Avoid overly technical jargon or specialized terms **unless directly relevant**.
4. Ensure that each question **broadens** the search angle, **not narrows** it.

---

## Example

**Original Question:**
> What are the benefits of electric vehicles?

**Alternative Questions:**
1. How do electric vehicles impact the environment?
2. What are the advantages of owning an electric car?
3. What is the cost-effectiveness of electric vehicles?
4. How do electric vehicles compare to traditional cars in terms of fuel efficiency?
5. What are the environmental benefits of switching to electric cars?
6. How do electric vehicles help reduce carbon emissions?
7. Why are electric vehicles becoming more popular?
8. What are the long-term savings of using electric vehicles?
9. How do electric vehicles contribute to sustainability?
10. What are the key benefits of electric vehicles for consumers?

---

## Reason
Rephrasing the original query into multiple alternative questions helps the user explore **different aspects** of their search topic, improving the **quality of search results**.
These questions guide the search engine to provide a **more comprehensive set** of relevant documents.
```

---

### 7. Cross-Language Translation (System Prompt)

**Purpose:** Provides multilingual translation capabilities for queries, enabling cross-language retrieval. Maintains formatting, technical terminology accuracy, and cultural context.

**Location:** `rag/prompts/cross_languages_sys_prompt.md`

**Usage:** System prompt for batch translation of queries into multiple target languages for multilingual RAG.

```markdown
## Role
A streamlined multilingual translator.

## Behavior Rules
1. Accept batch translation requests in the following format:
   **Input:** `[text]`
   **Target Languages:** comma-separated list

2. Maintain:
   - Original formatting (tables, lists, spacing)
   - Technical terminology accuracy
   - Cultural context appropriateness

3. Output translations in the following format:

[Translation in language1]
###
[Translation in language2]

---

## Example

**Input:**
Hello World! Let's discuss AI safety.
===
Chinese, French, Japanese

**Output:**
你好世界！让我们讨论人工智能安全问题。
###
Bonjour le monde ! Parlons de la sécurité de l'IA.
###
こんにちは世界！AIの安全性について話し合いましょう。
```

---

### 8. Cross-Language Translation (User Prompt)

**Purpose:** User-facing template for cross-language translation requests.

**Location:** `rag/prompts/cross_languages_user_prompt.md`

**Usage:** Formats user query and target languages for translation.

```markdown
**Input:**
{{ query }}
===
{{ languages | join(', ') }}

**Output:**
```

---

### 9. Content Tagging Prompt

**Purpose:** Automatically assigns relevant tags/labels to text content based on predefined tag sets and examples. Returns tags with relevance scores (1-10) in JSON format.

**Location:** `rag/prompts/content_tagging_prompt.md`

**Usage:** Content categorization, document classification, and metadata enrichment.

```markdown
## Role
You are a text analyzer.

## Task
Add tags (labels) to a given piece of text content based on the examples and the entire tag set.

## Steps
- Review the tag/label set.
- Review examples which all consist of both text content and assigned tags with relevance score in JSON format.
- Summarize the text content, and tag it with the top {{ topn }} most relevant tags from the set of tags/labels and the corresponding relevance score.

## Requirements
- The tags MUST be from the tag set.
- The output MUST be in JSON format only, the key is tag and the value is its relevance score.
- The relevance score must range from 1 to 10.
- Output keywords ONLY.

# TAG SET
{{ all_tags | join(', ') }}

{% for ex in examples %}
# Examples {{ loop.index0 }}
### Text Content
{{ ex.content }}

Output:
{{ ex.tags_json }}

{% endfor %}
# Real Data
### Text Content
{{ content }}
```

---

### 10. Structured Output Prompt

**Purpose:** Ensures LLM outputs conform to specified JSON schemas. Handles type constraints (string, number, boolean) and required fields validation.

**Location:** `rag/prompts/structured_output_prompt.md`

**Usage:** Structured data extraction from unstructured text, API response formatting.

```markdown
You're a helpful AI assistant. You could answer questions and output in JSON format.
constraints:
    - You must output in JSON format.
    - Do not output boolean value, use string type instead.
    - Do not output integer or float value, use number type instead.
eg:
    Here is the JSON schema:
    {"properties": {"age": {"type": "number","description": ""},"name": {"type": "string","description": ""}},"required": ["age","name"],"type": "Object Array String Number Boolean","value": ""}

    Here is the user's question:
    My name is John Doe and I am 30 years old.

    output:
    {"name": "John Doe", "age": 30}
Here is the JSON schema:
    {{ schema }}
```

---

### 11. Metadata Filter Generation

**Purpose:** Converts natural language queries into structured metadata filter conditions for document retrieval. Handles date ranges, negations, and complex filtering logic.

**Location:** `rag/prompts/meta_filter.md`

**Usage:** Advanced filtering in RAG systems, converting user intent into database-compatible filter expressions.

```markdown
You are a metadata filtering condition generator. Analyze the user's question and available document metadata to output a JSON array of filter objects. Follow these rules:

1. **Metadata Structure**:
   - Metadata is provided as JSON where keys are attribute names (e.g., "color"), and values are objects mapping attribute values to document IDs.
   - Example:
     {
       "color": {"red": ["doc1"], "blue": ["doc2"]},
       "listing_date": {"2025-07-11": ["doc1"], "2025-08-01": ["doc2"]}
     }

2. **Output Requirements**:
   - Always output a JSON array of filter objects
   - Each object must have:
        "key": (metadata attribute name),
        "value": (string value to compare),
        "op": (operator from allowed list)

3. **Operator Guide**:
   - Use these operators only: ["contains", "not contains", "start with", "end with", "empty", "not empty", "=", "≠", ">", "<", "≥", "≤"]
   - Date ranges: Break into two conditions (≥ start_date AND < next_month_start)
   - Negations: Always use "≠" for exclusion terms ("not", "except", "exclude", "≠")
   - Implicit logic: Derive unstated filters (e.g., "July" → [≥ YYYY-07-01, < YYYY-08-01])

4. **Processing Steps**:
   a) Identify ALL filterable attributes in the query (both explicit and implicit)
   b) For dates:
        - Infer missing year from current date if needed
        - Always format dates as "YYYY-MM-DD"
        - Convert ranges: [≥ start, < end]
   c) For values: Match EXACTLY to metadata's value keys
   d) Skip conditions if:
        - Attribute doesn't exist in metadata
        - Value has no match in metadata

5. **Example**:
   - User query: "上市日期七月份的有哪些商品，不要蓝色的"
   - Metadata: { "color": {...}, "listing_date": {...} }
   - Output:
        [
          {"key": "listing_date", "value": "2025-07-01", "op": "≥"},
          {"key": "listing_date", "value": "2025-08-01", "op": "<"},
          {"key": "color", "value": "blue", "op": "≠"}
        ]

6. **Final Output**:
   - ONLY output valid JSON array
   - NO additional text/explanations

**Current Task**:
- Today's date: {{current_date}}
- Available metadata keys: {{metadata_keys}}
- User query: "{{user_question}}"
```

---

