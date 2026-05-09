# SHL Conversational Assessment Recommender — Approach

## Summary
This project implements a conversational recommendation agent for SHL assessments using a retrieval-driven architecture. The objective was to guide recruiters from vague hiring intent to grounded SHL assessment recommendations while maintaining strict schema compliance, low hallucination rates, and strong Recall@10 performance.

## 1. System Design
The backend was implemented using FastAPI with two public endpoints: `GET /health` and `POST /chat`. The API is fully stateless, so conversation state is reconstructed dynamically from the complete message history included in each request. The system follows a retrieval-first architecture rather than relying purely on LLM reasoning. High-level flow: 

*SHL Catalog JSON → preprocessing → embeddings generation → FAISS index → retrieval → conversational policy engine → grounded response generation.*

## 2. Retrieval Setup
The provided SHL catalog JSON was used as the source of truth. Records were normalized and enriched with derived metadata such as assessment type, role tags, seniority indicators, and normalized duration fields. Embeddings were generated locally using SentenceTransformers (`all-MiniLM-L6-v2`) and stored in a FAISS index for fast semantic retrieval.

Retrieval combined semantic similarity with metadata filtering to improve ranking quality and reduce irrelevant recommendations. Only compact structured metadata was passed to the LLM to minimize token usage and reduce hallucination risk.

## 3. Prompt & Conversation Design
The conversational agent supports five behaviors:
- Clarification for vague hiring requests
- Grounded recommendations
- Refinement when constraints change
- Assessment comparison
- Refusal for off-topic or prompt-injection attempts 

Prompt design focused on deterministic and concise responses. Temperature was kept low (0.1) to improve schema stability and reduce conversational drift. The prompts explicitly restricted recommendations to catalog entries only.

## 4. Model & Inference Strategy
Groq-hosted Llama models were used for inference. A lightweight model handled clarification/routing tasks while a stronger model handled final recommendations and comparisons. This reduced token usage and improved latency under evaluator constraints. The architecture intentionally avoided over-engineered multi-agent orchestration and instead emphasized reliability, retrieval quality, and grounded behavior.

## 5. Evaluation & Improvement
Evaluation focused on:
- Schema compliance
- Retrieval relevance
- Recall@10
- Hallucination prevention
- Multi-turn conversational behavior 

The provided sample conversations were replayed repeatedly to validate clarification quality, recommendation updates, comparison handling, and refusal behavior. Early versions over-relied on the LLM and produced weaker retrieval grounding. Retrieval quality improved after:
- metadata enrichment
- stricter filtering
- limiting prompt context size
- validating recommendations against the catalog before returning them

## 6. Challenges & Trade-offs
The primary challenge was balancing clarification quality against the assignment’s 8-turn conversation limit. Asking too few questions reduced recommendation accuracy, while asking too many risked evaluator penalties. Another trade-off involved deployment constraints. Initial deployments on low-memory environments failed due to embedding and index loading overhead. This was mitigated through lighter embedding models, startup optimization, and deployment adjustments.

## 7. AI Assistance
AI-assisted development tools including Gemini and GitHub Copilot were used for scaffolding, refactoring assistance, deployment setup, debugging support, and iterative prompt refinement. Final architecture decisions, retrieval logic, evaluation strategy, testing, and implementation validation were manually reviewed and refined.
