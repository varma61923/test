# CommonTrace: State-of-the-Art Memory Platform Implementation

## Context

CommonTrace is a governed memory protocol for AI agents with causal proof of what helps or hurts. The current branch `feat/memory-platform-evolution` has implemented:

- Adaptive-v1 lexical scorer with stemming and query-relative relevance cutoff
- Scoped lessons (project/team routing)
- Temporal validity (valid_from, valid_until)
- COGX interop export/import
- Provenance tracking with per-edge/node evidence
- Canonicalization with alias normalization
- Daemon mode for background consolidation
- Watch mode for cascade reconcile
- Git-backed memory
- Performance: 3x query latency via persisted corpus index, lazy CLI
- Hierarchical episode→fact expansion
- Entity index with entity-boosted ranking
- Profile store with static/dynamic split
- Multi-modal ingestion pipeline
- Autonomous agent loop
- Knowledge graph with max_hops/max_edges caps
- 4-tier E2E test harness

## Audit Findings Summary

We audited 6 state-of-the-art memory systems (Cognee, EverOS, Mem0, Letta-code, Supermemory, Zep). Key findings:

### Critical Gaps (High Priority)

1. **Ingestion Pipeline Modularization** (Zep, EverOS, Cognee)
   - Current: Direct writes without transformation/validation
   - Need: Loader → Transforms → Submitter pattern with preview()
   - Transforms: TextChunker, LLMContextualizer, AliasCanonicalizer, LimitGuard
   - Impact: Prevents data corruption, 40-60% ingestion speedup with lazy hashing

2. **Full Bi-Temporal Graph Architecture** (Zep, Cognee)
   - Current: valid_from/valid_until only (valid time dimension)
   - Need: Add expired_at (record time), valid_at distinct from created_at
   - Need: Contradiction resolution ("latest valid_at wins")
   - Impact: Accurate fact evolution for long-running agent fleets

3. **Git-Backed Memory (MemFS)** (Letta-code)
   - Current: Flat files in memory/
   - Need: Git-backed filesystem with version control
   - Need: Pre-commit validation with configurable limits
   - Need: Automatic conflict repair
   - Impact: Production-grade memory management with audit trail

4. **Entity Extraction & Linking** (Mem0, Zep)
   - Current: Basic entity types in graph
   - Need: Sophisticated spaCy-based extraction (772 lines battle-tested)
   - Need: Global deduplication, batch processing, entity store
   - Impact: Richer graph construction, 30-50% noise reduction

5. **Hybrid Retrieval with Reranking** (EverOS, Supermemory, Zep, Cognee)
   - Current: Semantic + graph boost, simple fusion
   - Need: BM25 + vector fusion via RRF
   - Need: Cross-encoder reranking (MMR for diversity)
   - Need: Multi-channel (chunks, entities, facts, global context)
   - Need: Truth subspace for temporal consistency
   - Impact: 25-40% retrieval quality improvement

6. **Comprehensive Observability** (Cognee, EverOS)
   - Current: Basic logging
   - Need: OpenTelemetry-native tracing with semantic conventions
   - Need: Structured logging with context propagation
   - Need: Metrics collection (operations, queries, errors)
   - Impact: 10x faster debugging with <5% overhead

7. **Declarative Ontology** (Zep, Cognee)
   - Current: Hard-coded entity/edge types
   - Need: RDFLib-based ontology resolvers
   - Need: Entity canonicalization against ontology
   - Need: Flexible graph modeling without code changes
   - Impact: 30-50% graph noise reduction

8. **Memory Graph Visualization** (Supermemory)
   - Current: No visualization
   - Need: Interactive force-directed graph (d3-force)
   - Need: Memory relationships (updates, extends, derives)
   - Need: Version chains for memory evolution
   - Impact: Better understanding of memory relationships

### Medium Priority

9. **Context Budgeting** (Letta-code)
   - Conservative character-based token estimation (4 chars/token)
   - Per-subagent budgets with intelligent truncation
   - Prevents context overflow in subagent operations

10. **Async Batching & Queueing** (Mem0, Zep, EverOS)
    - Phased pipeline with fallback mechanisms
    - Batch API with transparent sequential fallback
    - Impact: 30-50% ingestion time reduction

11. **Multi-Tenant Dataset Isolation** (Cognee)
    - Per-user+dataset isolated graph/vector databases
    - Handler registry for different backends
    - Impact: Security and reduced cross-dataset interference

12. **Temporal Decay / Automatic Forgetting** (Supermemory)
    - Expiration dates for lessons/traces
    - Forget reason tracking
    - Contradiction resolution
    - Impact: Automatic cleanup of stale content

### Low Priority

13. **Multi-Modal Ingestion** (EverOS, Supermemory)
    - Images, PDFs, audio, office documents
    - AST-aware code chunking
    - Impact: Broader content support

14. **LLM Provider Ecosystem** (Mem0)
    - 24 LLM providers with abstract base class
    - Impact: Flexibility if multiple backends needed

## Implementation Priority

### Phase 1: Foundation (Critical)

1. **Ingestion Pipeline Modularization**
   - Create `commontrace/ingest/pipeline.py` with Loader/Transform/Submitter pattern
   - Implement transforms: TextChunker, LLMContextualizer, AliasCanonicalizer, LimitGuard
   - Add preview() method for pre-flight validation
   - Add lazy hashing for large files (<256MB hashed immediately, larger on collision)

2. **Full Bi-Temporal Graph**
   - Add fields to GraphEdge: valid_at, invalid_at, expired_at
   - Implement contradiction resolution
   - Add temporal retriever with interval queries

3. **Git-Backed Memory (MemFS)**
   - Convert memory/ to git-backed filesystem
   - Implement pre-commit validation hook
   - Add automatic conflict repair
   - Implement memory handoff pattern

4. **Entity Extraction & Linking**
   - Port Mem0's 772-line entity extraction pipeline
   - Implement global deduplication
   - Add entity store with batch processing
   - Integrate entity boosting into retrieval

### Phase 2: Enhanced Retrieval

5. **Hybrid Retrieval with Reranking**
   - Implement BM25 + vector fusion via RRF
   - Add cross-encoder reranking (MMR)
   - Implement multi-channel retrieval
   - Add truth subspace for temporal consistency

6. **Declarative Ontology**
   - Create ontology.py with EntityType/EdgeType declarations
   - Implement RDFLib-based ontology resolvers
   - Migrate hard-coded types to default ontology
   - Add entity canonicalization

7. **Comprehensive Observability**
   - Add OpenTelemetry integration with semantic conventions
   - Implement structured logging with context propagation
   - Add metrics collection
   - Implement secret redaction

### Phase 3: Advanced Features

8. **Memory Graph Visualization**
   - Adopt/fork Supermemory's memory-graph package
   - Implement interactive force-directed graph
   - Add memory relationships (updates, extends, derives)
   - Add version chains

9. **Context Budgeting**
   - Implement character-based token estimation
   - Add per-subagent budgets
   - Implement intelligent truncation

10. **Async Batching & Queueing**
    - Implement phased pipeline with fallback
    - Add batch API with transparent sequential fallback
    - Implement job queue for async operations

## Architectural Principles

1. **Governance First**: Every feature must support causal proof of value
2. **Protocol-Driven**: All changes must respect protocol/PROTOCOL.md
3. **Test-Driven**: 4-tier E2E test harness must pass
4. **Performance-First**: Every change must be benchmarked
5. **License-Compliant**: Apache 2.0 ideas are fine, no code copying
6. **Incremental**: Build on existing feat/memory-platform-evolution branch

## Your Task

You are to continue building CommonTrace into a state-of-the-art memory platform. 

1. Review the current state of `feat/memory-platform-evolution` branch
2. Review the detailed audit findings in `.agentteam/teams/repo-scan/artifacts/`
3. Prioritize Phase 1 features (Foundation)
4. Implement the highest-impact features first
5. Ensure all tests pass
6. Benchmark performance improvements
7. Document changes in CHANGELOG.md

Start with the ingestion pipeline modularization - it's the foundation for everything else and prevents data corruption. Then proceed with bi-temporal graph and git-backed memory.

Build something that would make CommonTrace a multibillion-dollar startup product.
