# ai-mentor-portfolio
AI Mentor Bootcamp — <Bobbykiran Madiki>
Day 5B_Task:
Inference timing comparison (3 runs each, after warm-up):
  API:   min 0.25s | avg 0.31s
  Local: min 0.83s | avg 1.00s
====================================================================================
## Day 6 — Capstone Sprint 1: PlacementDataProcessor

### Engineer Answer

1. **PROBLEM** — JDs from Naukri / LinkedIn are messy text — placement cells need structured data to filter ("which JDs want Java + CGPA 7+?"). Manual extraction is unscalable for 50+ JDs.

2. **ARCHITECTURE** — JD URL → BeautifulSoup scraper (extract clean text) → Gemini structured-output call (response_schema=JD Pydantic) → JSON Lines file. Validation at each step; retry on schema fail.

3. **TRADE-OFFS** —
   - Cost: free Gemini ~1 JD/sec on average; ~30K tokens/day quota → ~5K JDs/day.
   - Accuracy: Pydantic catches schema violations but not semantic errors (e.g., model says skill is "Python" when source says "Python 3.12 specifically").
   - Latency: ~2-5s per JD (Gemini call dominant).
   - Complexity: scraping fragile (sites block automation). Cached fallback is mandatory.

4. **SCALE** —
   - 10 JDs/day: trivial. Today's lab.
   - 100 JDs/day: still in free quota. Add overnight batch + sleep between calls.
   - 10K JDs/day: free tier breaks. Move to paid Gemini OR self-host an open model.

5. **INTERVIEW ANSWER** — "I built a structured-output pipeline that turns scraped JDs into clean filterable JSON, using free Gemini and Pydantic. Schema-first design with retry-on-failure made it production-shaped on a free-tier API."
=====================================================================================
## Day 7 — Capstone Sprint 2: PlacementKnowledgeRAG

### Engineer Answer

1. **PROBLEM** — Frontier LLMs do not know your private data (JDs, syllabi). Students need a chatbot that answers from YOUR placement corpus, with citations they can verify.

2. **ARCHITECTURE** — 5-box RAG: embed (MiniLM 384-dim) → index (ChromaDB persistent collection with metadata) → retrieve (top-4 cosine similarity) → augment (citation-enforcing prompt) → generate (Gemini 2.5).

3. **TRADE-OFFS** —
   - Cost: free (MiniLM local + Gemini quota).
   - Accuracy: top-4 retrieval has ~80% precision on placement-relevant queries.
   - Latency: ~1-2s per query (embedding + retrieval) + 2-5s (Gemini).
   - Complexity: chunking strategy (500-token, 50-overlap) needs tuning per corpus.
   - Caveat: refuses out-of-corpus queries (good!) but only when prompt enforces "do not guess".

4. **SCALE** —
   - 50 docs (today): trivial. ChromaDB returns in <100ms.
   - 5K docs: still fine on one machine.
   - 1M docs: need vector DB optimisation (HNSW indexing, server mode), or move to Pinecone/Weaviate.

5. **INTERVIEW ANSWER** — "I built a citation-enforcing RAG over 50+ placement docs (JDs + syllabi) using free MiniLM embeddings, ChromaDB, and Gemini. The system either cites a specific chunk or refuses — no hallucinated answers. Same pattern scales to thousands of docs without retraining."

### 5 cited Q&A pairs

| # | Question | Answer (excerpt) | Sources cited |
|---|----------|------------------|---------------|
| 1 | Which companies want Java + DSA + CGPA 7+? | "Per jd_0 (TCS Digital): Java + DSA required, CGPA 7.0 cutoff..." | jd_0, jd_5, jd_8 |
| 2 | Sem 5 OS topics? | "Per cse_sem5_2: paging, segmentation, virtual memory..." | cse_sem5_2, cse_sem5_5 |
| 3 | Which JDs require Python? | "Per jd_3 (Accenture)..., per jd_5 (Cognizant)..." | jd_3, jd_5, jd_9 |
| 4 | Top 3 skills across JDs? | "Java, Python, SQL appear in 7+ of 10 JDs..." | jd_0, jd_1, jd_2 |
| 5 | What is TCS Codevita? | "I do not know — not in corpus." | (none) |
============================================================================================
## Day 9 Lab 9A — Trace as a story

1. **Human asked:** "What is TCS's 2026 hiring quota?"
2. **Agent thought:** "I don't know recent figures. I should search."
3. **Agent acted:** called `web_search('TCS 2026 hiring quota')`.
4. **Agent observed:** got back search results mentioning 40-50K range.
5. **Agent answered:** synthesised "Based on search results, TCS plans to hire 40-50K freshers..."

This is the ReAct loop. Every agent we build follows this pattern.
-------------------------------------------------------------------
## Day 9 Lab 9A — Hello-LangGraph

- 1-tool ReAct agent with DuckDuckGo web_search
- 4-message trace on a live-fact question (TCS 2026 hiring)
- Failure case: bad URL → agent reported "could not find" / agent hallucinated [pick one]

### Reflection (3 lines)

1. The trace IS the explanation. Print every step.
2. The doc-string IS the prompt. Bad doc-string = bad tool selection.
3. Real agents handle tool failures gracefully — define failure modes in the doc-string.
- ✅ Hello-LangGraph agent runs end-to-end
- ✅ 4-message trace printed on live-fact question
- ✅ 1 failure case captured + behaviour documented
- ✅ "Trace as a story" markdown cell in notebook
- ✅ Pushed to repo
==============================================================================================
## Day 9 — Capstone Sprint 4: Career Agent

### 3 tools wired
1. **jd_fetcher** — wraps Day 6's fetch_jd. Returns clean text or ERROR string.
2. **skills_gap** — pure function set difference. Deterministic.
3. **answer_scorer** — Gemini-backed scoring 1-10 with rationale.

### 3 successful runs
| # | Student | Tools used | Outcome |
|---|---------|-----------|---------|
| 1 | Ravi Kumar (CSE) → TCS | skills_gap, answer_scorer | Skill-gap: Spring Boot, AWS |
| 2 | Sneha Reddy (ECE) → Cognizant | skills_gap | Strong match — focus on interview practice |
| 3 | Arun Pillai (IT) → Amazon | skills_gap, answer_scorer | Strong match — score 8/10 on sample |

### 1 failure-recovery analysis

Bad URL passed to jd_fetcher. Agent received `ERROR:` from tool. Agent correctly responded:
"I could not fetch the JD URL. Please provide a working URL." No hallucinated JD content.
This is the safe behaviour. If the agent had hallucinated, the fix would be to tighten the doc-string of jd_fetcher.

### Engineer Answer

1. **PROBLEM** — A static RAG cannot take actions. Students need an assistant that fetches JDs, computes skill gaps, and evaluates their answers — autonomously.

2. **ARCHITECTURE** — LangGraph ReAct agent with 3 specialised tools. Each tool is a plain Python function with a precise doc-string. Agent reasons about which tool to call.

3. **TRADE-OFFS** —
   - Cost: 5-15 LLM calls per task (each ~1-3K tokens). ~20K tokens per student session.
   - Latency: 5-10s per task (LLM calls dominant).
   - Reliability: tools must return predictable strings. ERROR returns are part of the contract.
   - Complexity: doc-strings are now part of the prompt. Bad doc-string = wrong tool.

4. **SCALE** —
   - 1 student / minute: free quota OK.
   - 50 students / day: hits free quota. Switch to paid.
   - 1K students / day: needs caching + parallel inference.

5. **INTERVIEW ANSWER** — "I built a 3-tool LangGraph agent that takes a student profile and produces tailored placement prep — JD analysis, skill gap, answer scoring. Each tool is a plain function; the agent picks which to call. Failure recovery is built into tool contracts."
```
========================================================================================


