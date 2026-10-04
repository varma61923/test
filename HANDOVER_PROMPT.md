# CommonTrace: Road to Best-in-World Memory

## How to use this

Run one prompt per fresh coding session, in the order given, and merge it before starting the next. Each prompt is self-contained: paste the **Shared preamble** first, then the one prompt.

1. Start a new session on a new branch from the latest `main`.
2. Paste the Shared preamble, then exactly one enhancement prompt.
3. Require the agent to run the baseline benchmarks *before* changing code, so every claim is a measured before/after.
4. Merge only if the regression gate in the preamble passes. If the change helps one benchmark and hurts another beyond noise, the agent must make it opt-in instead of default.
5. Record the new scores in the scoreboard table in the README, then move to the next prompt.

Prompts are ranked by expected gain per unit of effort. Tier 1 attacks the largest measured gaps; Tier 5 is product and platform work that does not move benchmark scores but that buyers check for. A prompt marked **needs keys** requires a model API key and spends money; the agent must print an estimated cost and stop for your approval before any paid call.

## Where CommonTrace stands

CommonTrace puts the right evidence in context for 54–81% of questions at 1,500 tokens, but it has never measured **answer accuracy**, the number every competitor publishes. The leaders now claim 94–96% on LongMemEval and 64–73% on BEAM at 10M tokens. Until CommonTrace reports the same metric under the same judge, "best in the world" cannot be claimed or disproved.

**CommonTrace today** (evidence recall: share of a question's cited evidence that lands in the recalled context; `minilm` embeddings + blended cross-encoder unless noted; merged in PR #92):

| Benchmark | Questions | 1,500 tokens | 4,000 tokens | Recall time |
| --- | --: | --: | --: | --: |
| LoCoMo | 1,540 | 80.6% | 88.6% | \~0.6 s |
| LongMemEval (keyword-only) | 120 | 77.4% | 84.4% | 13 ms |
| BEAM 100K | 400 | 67.4% | 73.3% | 1.7–5.7 s |
| DolphinBench (task requests) | 600 | 54.1% | 71.3% | 4.3 s |

**Published answer accuracy of the leaders** (not directly comparable to the table above: these are judged answers, not evidence recall):

| System | LongMemEval | LoCoMo | BEAM 1M | BEAM 10M | Source |
| --- | --: | --: | --: | --: | --- |
| Agent Zero Memory | 95.6% | 93.6% |  |  | [arXiv 2608.29606](https://arxiv.org/abs/2608.29606) |
| Mastra Observational Memory | 94.9% |  |  |  | [mastra.ai](https://mastra.ai/research/observational-memory) |
| Scroll (context as an environment) | 94.8% |  |  | 73.1% | [arXiv 2608.21690](https://arxiv.org/abs/2608.21690) |
| Mem0 (April 2026 algorithm) | 94.4% | 92.5% | 64.1% | 48.6% | [mem0.ai](https://mem0.ai/blog/ai-memory-benchmarks-in-2026) |
| Hindsight | 91.4% |  | 73.9% | 64.1% | [vectorize.io](https://hindsight.vectorize.io/blog/2026/04/02/beam-sota) |
| Honcho |  |  | 63.1% | 40.6% | [vectorize.io](https://hindsight.vectorize.io/blog/2026/04/02/beam-sota) |

**The gaps that matter most, in order:**

1. **No answer-accuracy number.** Every comparison above is apples to oranges until Prompt 1 lands.
2. **Whole-history abilities are weak.** BEAM summarization is 18% / 26%, event ordering 42% / 52%, multi-session 60% / 67% (chart below). These need structure built at write time, which keyword and vector search cannot supply.
3. **Temporal questions.** LongMemEval temporal reasoning is 57.5% / 72.5% keyword-only; Mastra reports 95.5% with dated observations.
4. **Preferences.** LongMemEval single-session preference is 65% / 65%; Mastra reports 100% with GPT-5-mini.
5. **Abstention has no retrieval signal.** Confidence and cross-encoder scores separate unanswerable questions at chance (AUC about 0.47).
6. **Scale is untested.** Nothing has run at BEAM 1M or 10M, where the leaders differentiate and where Honcho drops from 63% to 41%.
7. **The hosted Hub has no conversation memory.** It lives only in the local CLI, MCP server and gateway; the TypeScript SDK and the framework integrations do not expose it.

&#91;embedded content: PR #92 benchmark runs · BEAM 100K, 400 questions · minilm embeddings + blended cross-encoder · evidence recall, not answer accuracy\]

The three blue abilities need the whole history (summaries, order, several sessions), which similarity search cannot gather; information extraction misses are mostly paraphrases. Prompts 3, 5, 8 and 9 target these four.

## Shared preamble (paste before every prompt)

Copy this block verbatim, then paste one enhancement prompt under it.

```markdown
You are working on CommonTrace (repo: varma61923/commontrace-v2). Read AGENTS.md and README.md first and follow them: Python 3.10+, stdlib-first, argparse, os.path.join for paths, no hardcoded platform paths, optional heavy dependencies imported lazily behind extras.

Conversation memory lives in commontrace/conversation/ (store.py, search.py, profile.py, timeparse.py, embed.py, answer.py, summary.py, extract.py). Retrieval arms: commontrace/semantic_arm.py, commontrace/rerank_arm.py, commontrace/_lexical.py. Benchmarks: benchmarks/conversation_bench.py (--dataset locomo|longmemeval|beam|dolphin, --embedder, --rerank, --rerank-blend, --budget) and benchmarks/dolphinbench/commontrace_harness.py. Competitor source, where a prompt cites it, is open source: check its LICENSE, keep the copyright notice on any adapted code, and prefer re-implementing the idea over copying.

WORK RULES
1. Do exactly ONE enhancement: the one in the prompt below. Do not start others, even if you notice them; list them at the end instead.
2. Baseline first. Get the data once (keep it out of git): LoCoMo locomo10.json from github.com/snap-research/locomo; LongMemEval longmemeval_s from huggingface.co/datasets/xiaowu0162/longmemeval-cleaned; BEAM 100K/1M/10M parquet from huggingface.co/datasets/Mohammadta/BEAM and BEAM-10M; DolphinBench from its repo. Every command below also takes --data <path>. Before changing code, run and save:
   python benchmarks/conversation_bench.py --dataset locomo --embedder none --rerank none --budget 1500,4000
   python benchmarks/conversation_bench.py --dataset longmemeval --limit 120 --embedder none --rerank none --budget 1500,4000
   python benchmarks/conversation_bench.py --dataset beam --embedder none --rerank none --budget 1500,4000
   python benchmarks/conversation_bench.py --dataset dolphin --embedder none --rerank none --budget 1500,4000,7000
   plus the same four with --embedder minilm --rerank auto when the prompt touches semantic or rerank code.
   Record per-category numbers, recall latency, and ingest time.
3. Research before building. Read the cited papers (arxiv.org/abs/<id>) and competitor files. Write a short design note in the PR description: the idea, what you adopt, what you reject and why.
4. Keep the keyword-only path fast and dependency-free. New model-based behaviour must be optional, off when its dependency or key is missing, and must never fail a recall.
5. Raw turns stay the source of truth. Anything derived (facts, summaries, timelines, entities) must point back to the turn ids it came from, and recall must be able to show that evidence.
6. Paid calls: if a step needs an LLM API key, print the number of calls, tokens and estimated cost, and stop for approval. Cache every model response on disk keyed by a hash of its input so reruns cost nothing.
7. Tests: add unit tests for every new behaviour (tests/test_conversation_*.py style), run ruff check . and python -m pytest tests/ -q, and the hub image test if hub/ changed.

REGRESSION GATE (all must hold to make the change a default)
- Target metric improves by at least the amount the prompt names, at both 1,500 and 4,000 tokens.
- No benchmark's overall score drops by more than 0.5 points, and no category by more than 2 points, at any budget.
- Keyword-only recall latency grows by no more than 20%; ingest time by no more than 50% unless the prompt allows more.
- Report a paired bootstrap 95% confidence interval for the target metric change once Prompt 2 has landed.
If the gate fails, ship the feature behind an Options flag (default off) with the measured numbers in the README, and say so.

DELIVERABLES
- One commit series on a fresh branch from main, a PR with: design note, before/after tables per benchmark and category, latency and cost, and a list of follow-ups you noticed but did not do.
- README scoreboard and CHANGELOG updated with measured numbers only. No model identifiers in commits or code comments.
```

## Tier 1 — Measurement and retrieval quality

Prompts 1 and 2 come first because every later prompt is judged by them. Prompts 3–9 attack the largest retrieval gaps, each with a numeric target.

### Prompt 1 — Answer accuracy, judged the way the leaderboards judge it

Moves: comparability with every competitor. **Needs keys.**

```markdown
ENHANCEMENT: Measure end-to-end answer accuracy exactly as each benchmark's official protocol does.

WHY: CommonTrace reports evidence recall; competitors report judged answer accuracy (LongMemEval 94-96%, LoCoMo 92-94%, BEAM-10M 64-73%). benchmarks/conversation_bench.py --answer already answers and grades, but with one generic judge (JUDGE_PROMPT) for every dataset, so its numbers cannot be compared with anyone's.

RESEARCH:
- LongMemEval (arXiv 2410.10813) and its repo src/evaluation/evaluate_qa.py (MIT): one judge prompt per question type and a separate one for abstention (_abs) questions.
- LoCoMo (arXiv 2402.17753) and the Mem0 paper (arXiv 2504.19413): the LLM-as-judge "J" protocol, which categories are scored (1-4) and which are excluded (adversarial, 5).
- BEAM (arXiv 2510.27246) and its dataset card: per-ability rubric/nugget scoring, Kendall tau-b for event ordering, abstention scoring.
- Mem0's 2026 benchmark post: report tokens per query and p50 latency next to accuracy.

BUILD:
1. benchmarks/judges/ with one module per benchmark that reproduces its official judge prompt and scoring rule verbatim; a header comment names the source file and commit. Keep today's judge as --judge generic.
2. Separate --answer-model and --judge-model; default the judge to the model the official protocol names; record both in the output JSON.
3. Per category: accuracy, mean context tokens sent to the answer model, answer latency p50/p95, total cost. For BEAM, the official per-ability score.
4. Disk cache for answers and judgments keyed by (model, prompt hash); resume after interruption; --max-cost USD refuses to start when the estimate is higher.
5. Two reference modes in the same run: full-context (whole history when it fits the model) and no-memory, so every table shows memory's lift.
6. Targets for the first run: LongMemEval-S all 500, LoCoMo categories 1-4 (1,540), BEAM 100K (400). Print the cost estimate and stop for approval.

ACCEPTANCE:
- On a 100-answer sample, our judge agrees with the official script's verdict on at least 98% (run both, diff them, report disagreements).
- README scoreboard gains an "answer accuracy" table with models, date and cost.
- No retrieval code changed in this PR.
```

### Prompt 2 — Statistics and scale in the benchmark harness

Moves: trust in every later result; exposes BEAM 1M/10M. No keys.

```markdown
ENHANCEMENT: Make benchmark deltas statistically honest and add the scales where leaders differentiate.

WHY: Our tuning moved scores by 0.5-2 points, within the sampling error of 120-400 questions. Leaders separate at BEAM 1M and 10M (Hindsight 73.9% -> 64.1%, Honcho 63.1% -> 40.6%), where we have never run. Retrieval-only systems also publish Recall@k (agentmemory: 95.2% R@5 on LongMemEval), which we cannot quote.

BUILD:
1. Paired bootstrap (10,000 resamples, fixed seed) for the difference between two result JSONs: benchmarks/compare.py A.json B.json prints delta and 95% CI per benchmark and category, and marks changes whose CI excludes zero.
2. Standard retrieval metrics beside evidence recall: Recall@5/10, NDCG@10 at turn and session level (match LongMemEval's src/evaluation/print_retrieval_metrics.py definitions).
3. BEAM 500K, 1M and 10M support with streaming ingestion (no whole-file load), ingest throughput (turns/s), store size on disk, peak RSS, recall p50/p95.
4. StateMemBench (arXiv 2608.19652) loader if its data is public: current-state accuracy proxy = evidence of the latest state in context and superseded state absent.
5. A one-command scoreboard: benchmarks/scoreboard.py runs the fast keyword-only suite (< 15 min on 4 cores) and writes a markdown table; CI runs it on a 10% stratified sample and fails on a regression whose CI excludes zero.

ACCEPTANCE: compare.py reproduces a known delta from PR #92; BEAM 1M ingests and answers end to end with numbers in the README; CI job green and under 10 minutes.
```

### Prompt 3 — Fact-augmented keys (index what each turn means, not only what it says)

Moves: information extraction, preferences, multi-session. Target: +6 points evidence recall on BEAM information extraction and LongMemEval single-session-preference at 1,500 tokens. No keys for the rule path; optional model path.

```markdown
ENHANCEMENT: Index every turn under extra keys (facts, keyphrases, normalised entities, dates) that point back to the same turn.

WHY: LongMemEval's analysis found that expanding each value's index key with extracted user facts raised recall and accuracy, and that round-level (turn-level) values beat session-level ones. Our misses on BEAM information extraction are paraphrases ("colour technologist" asked as "profession"; "Canva Pro $12.99/month" asked as "service I use for my resume").

RESEARCH: LongMemEval (2410.10813) section on key expansion and its repo src/index_expansion/batch_expansion_turn_userfact.py and batch_expansion_turn_keyphrases.py (MIT). Doc2query-style expansion. Mem0's April 2026 change: entities extracted, embedded and cross-linked as a retrieval boost.

BUILD:
1. A key table in commontrace/conversation/store.py: (turn_id, kind, text) with kinds fact|keyphrase|entity|date. Written at add() time.
2. Rule path (default, no model): keyphrases by RAKE/YAKE-style scoring in stdlib, entities from commontrace/entities.py, hypernym/category hints from a small curated map (job titles -> profession/occupation/job, money+per month -> subscription/cost/price, pet species -> pet), normalised dates.
3. Model path (opt-in, uses commontrace/conversation/extract.py's model config): one batched call per 40 turns producing self-contained user facts, cached.
4. Search: keys are a separate lexical (and, when an embedder is on, semantic) arm fused by RRF with a weight; a key hit retrieves its source turn, never the key text itself.
5. explain shows which key matched.

ACCEPTANCE: target above with the gate in the preamble; ingest time rule path <= +30%; tests for each key kind and for the key -> turn mapping.
```

### Prompt 4 — Time-aware retrieval: event dates, time filters, temporal arm

Moves: temporal reasoning, event ordering. Target: LongMemEval temporal-reasoning evidence recall 57.5% -> 75% at 1,500 tokens keyword-only; BEAM temporal and event ordering +5.

```markdown
ENHANCEMENT: Resolve when things happened (not only when they were said), and let questions filter and rank by those dates.

WHY: LongMemEval found time-aware query expansion (narrowing the search range to the time the question names) raised temporal recall substantially. Mastra's Observational Memory stores up to three anchors per observation (observation date, referenced date, relative offset) and reports 95.5% on temporal questions. Hindsight runs a dedicated temporal arm in parallel with semantic, BM25 and graph.

RESEARCH: LongMemEval (2410.10813) and src/index_expansion/batch_expansion_session_temp_event.py, temp_query_search_pruning.py (MIT). TReMu (arXiv 2502.01630): time-aware memorisation and neuro-symbolic date reasoning. Hindsight (MIT): hindsight_api/engine/query_analyzer.py (TemporalConstraint), search/temporal_extraction.py, search/reranking.py compute_recency_decay and temporal proximity in apply_combined_scoring.

BUILD:
1. Write time: for each turn, extract referenced dates/intervals from its text with commontrace/conversation/timeparse.py relative to the turn's own timestamp ("last Tuesday" said on 2024-03-14 -> 2024-03-12); store as (turn_id, start, end, granularity, phrase).
2. Query time: parse the question's time constraint (absolute, relative to now, "before/after X", "first/last time", "how many days between"). Hard-filter when explicit, soft-boost otherwise.
3. A temporal arm: turns whose event interval overlaps the constraint, ranked by proximity; fused by RRF with the other arms.
4. Annotate recalled lines with resolved event dates, e.g. [said 2024-03-14; about 2024-03-12], so the reader can do arithmetic.
5. Ordering questions return their evidence sorted by event date with ordinals (1., 2., 3.).

ACCEPTANCE: targets above; a unit table of 40 tricky phrases ("the weekend before last", "two Fridays ago", "in Q3", "over Christmas") with expected intervals; no change to recall on questions without time words.
```

### Prompt 5 — Entity graph with spreading activation (multi-hop)

Moves: multi-session reasoning, LoCoMo multi-hop. Target: BEAM multi-session +6 and LoCoMo multi-hop 57.9% -> 65% at 1,500 tokens.

```markdown
ENHANCEMENT: Add a graph arm: entities and turns as nodes, mentions/co-occurrence/time-adjacency as edges, ranked by personalised PageRank seeded from the question.

WHY: Multi-hop and multi-session questions need evidence that shares no words with the question but shares an entity with evidence that does. HippoRAG 2 reports gains on associative (multi-hop) retrieval with PPR over a phrase/passage graph; Hindsight builds temporal, semantic, entity and causal links at retain time and searches the graph in parallel with vector, BM25 and temporal arms; Mem0's April 2026 algorithm added entity linking as a retrieval boost.

RESEARCH: HippoRAG 2 (arXiv 2502.14802) and repo src/hipporag/HippoRAG.py run_ppr, graph_search_with_fact_entities (MIT). Hindsight (arXiv 2512.12818) and repo hindsight_api/engine/retain/link_creation.py, search/graph_retrieval.py, search/fusion.py cap_per_source (MIT). Zep/Graphiti (arXiv 2501.13956; graphiti_core search recipes, Apache-2.0).

BUILD:
1. Reuse commontrace/entities.py for mentions; store edges in SQLite (turn-entity, entity-entity co-mention with counts, turn-turn same-session adjacency with time gap).
2. Query: seed entities = entities in the question plus entities in the top-5 lexical/semantic hits; run PPR (damping 0.5-0.85, sweep it) with numpy if present, pure-Python power iteration otherwise; cap 30 iterations.
3. Arm output = turns ranked by PPR mass; cap per source before fusion (as Hindsight does) so the graph cannot crowd out direct matches.
4. Gate the arm: only when the question is multi-hop/broad (search.is_broad, two or more entities, or "both/and/compare/relationship"), so single-hop latency does not grow.

ACCEPTANCE: targets above; graph build adds <= 40% ingest time; recall p95 +<= 30 ms keyword-only on BEAM 100K; tests with a 3-hop toy conversation.
```

### Prompt 6 — Embedder and reranker bake-off on our own benchmarks

Moves: every semantic number. Target: +3 points overall evidence recall on LoCoMo, BEAM and DolphinBench at 1,500 tokens at equal or lower recall latency.

```markdown
ENHANCEMENT: Pick the default embedder and cross-encoder by measured quality-per-millisecond on our four benchmarks, not by general leaderboards.

WHY: We ship minilm/arctic-m and an accurate cross-encoder chosen in 2025. On DolphinBench the replacing cross-encoder cost evidence until we blended it; newer small models (0.3-0.6B) are trained on instruction-style queries closer to agent requests.

RESEARCH: Qwen3 Embedding and Reranker (arXiv 2506.05176). EmbeddingGemma (arXiv 2509.20354). Late interaction / multi-vector as an option (ColBERT-style). Matryoshka truncation for smaller vectors. Hindsight's cross_encoder.py and ji
