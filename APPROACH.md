# Approach Document — SHL Conversational Assessment Recommender

## Problem

Hiring managers describe roles in natural language ("I need a Java developer assessment") but SHL's catalog uses structured metadata (test types, job levels, duration). The gap between natural language and structured catalog requires: (1) understanding vague intent, (2) mapping it to relevant assessments, and (3) refining through conversation — all while staying strictly grounded in catalog data.

## Architecture

The system is a stateless FastAPI service with three layers: **retrieval**, **generation**, and **validation**.

Each `POST /chat` request carries the full conversation history. On every turn: (1) all user messages are concatenated into a search query, (2) FAISS retrieves the top-30 semantically similar assessments, (3) keyword boosting re-ranks results for exact technology matches, (4) the top-30 are injected into the LLM prompt as context, (5) the LLM generates a JSON response, and (6) every recommended URL and name is validated against the catalog index before returning.

## Retrieval Strategy

**Hybrid retrieval** combines dense vector search with sparse keyword matching.

**Dense**: All 377 assessments are embedded using `all-MiniLM-L6-v2` (384-dim). A FAISS `IndexFlatIP` index enables cosine similarity search over normalized vectors. The search text for each assessment concatenates name, description, type codes, job levels, and duration — giving the embedding model rich signal.

**Sparse**: An inverted keyword index maps technology names, job levels, and category terms to assessment indices. Query keywords that match get a +0.15 score boost for existing semantic matches, or a 0.3 base score for keyword-only matches. This ensures that exact technology names (e.g., "Java", "SQL", "OPQ") surface the right assessments even when the embedding model underweights them.

**Soft filtering**: Job level and type mismatches reduce scores (0.6–0.7x multiplier) instead of hard-filtering. This preserves Recall@10 — a relevant assessment that doesn't perfectly match the seniority filter still appears, just ranked lower.

## Prompt Design

The system prompt is ~700 tokens. It defines five behaviors (clarify, recommend, refine, compare, refuse), the JSON response schema, and type code definitions. No catalog data is embedded in the system prompt — only retrieved assessments are injected per turn.

Each turn appends a turn budget note to the last user message (e.g., "[Turns used: 3/8. Recommend soon if enough context.]"). This prevents the agent from over-clarifying and exhausting the 8-turn budget without recommending.

## Model Routing

Two Groq models: `llama-3.3-70b-versatile` (powerful) and `llama-3.1-8b-instant` (light). A rule-based router selects based on: turn count (≥5 → powerful), message keywords (compare/recommend/refine → powerful), message length (>150 chars → powerful), and conversation depth (≥3 turns → powerful). Early vague queries route to the 8B model for fast, cheap clarification. An automatic fallback switches from light to powerful on 403 permission errors.

## Hallucination Prevention

Three-layer defense: (1) The LLM only sees retrieved catalog entries, never the full catalog. (2) The prompt instructs the model to use only provided data. (3) Post-generation validation checks every recommendation URL against a catalog index. If a URL isn't found, the name is tried via case-insensitive exact match, then substring match, then keyword overlap (≥2 shared words). Unresolvable recommendations are dropped and logged. Deduplication removes duplicate URLs, and results are capped at 10.

## Evaluation

**Unit tests** (18 tests): catalog integrity (no duplicate IDs, all have URLs, valid type codes), schema compliance (exact field names, serialization), and retrieval quality (Java → finds Java, personality → finds OPQ).

**Eval harness**: Replays 10 sample conversation traces against the live API. Each trace feeds user messages sequentially, collects final recommendations, and computes Recall@10 against expected URLs. Note: the harness feeds pre-scripted messages without responding to agent questions, which underestimates performance vs. the real LLM-simulated evaluator.

## What Didn't Work

1. **Full catalog in system prompt** (Gemini approach): Embedding all 377 assessments as JSON consumed ~150K tokens, hitting free-tier rate limits immediately. Switched to retrieval-only context injection.
2. **Hard metadata filtering**: Filtering assessments by exact job level or type before scoring significantly reduced recall. Soft score penalties preserved diversity.
3. **Single-model approach**: Using only the 70B model for all turns was slower and more expensive. Routing simple clarifications to the 8B model halved latency for early turns with no quality loss.

## AI Tools Used

GitHub Copilot for boilerplate scaffolding. Gemini for initial prompt drafting. All architecture decisions, retrieval design, validation logic, and prompt engineering reflect actual understanding of the tradeoffs.
