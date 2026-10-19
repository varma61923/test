# Codex prompt 2: UniMem next wave (30 features)

Paste everything below the line into Codex, run from the root of the repository. It builds on the 22 features already implemented on branch `feat/frontier-causal-governance`.

---

## Role and mission

You are a principal engineer and applied statistician working on UniMem, a governed, causal memory system for AI agents and physical AI. UniMem's core differentiator is that every memory is measured:

- randomized per-memory holdouts estimate each memory's causal effect on task outcomes;
- harmful memories are withdrawn;
- a causal gate learns per context where each memory helps.

The first roadmap (F1–F22) added lineage, a transparency log, e-processes, a conformal harm gate, incident freeze, promotion certificates, safety memory, certified erasure, red-teaming, an outcome engine, a recovery registry, interference-aware designs, valuation, spatial memory, uplift targeting, attribution, a clean room, compiled memory, a vendor evidence layer and a ROS 2 / MCAP / LeRobot bridge.

In 2026 the memory store itself is a commodity that every hyperscaler bundles. Your mission in this second roadmap is to make UniMem **the independent referee of every change that agents and robots make to themselves**, and to make its evidence **travel across trust boundaries** without exposing the data behind it. Almost every feature below does one of two things:

1. **It adds a new unit of randomization.** The units are a skill, a writing agent, an optimizer's edit, a decision to ask a human, a teleoperator's correction, or a world model's verdict. Each one reuses the same holdout, AIPW, e-process, conformal-gate and promotion machinery.
2. **It makes evidence portable and verifiable by outsiders.** The channels are OpenTelemetry spans, Agent Skills evidence cards, verifiable credentials, zero-knowledge proofs, attested key release, and change streams with erasure receipts.

Implement the 30 features below (G1–G30) in the phase order given. Implement each one completely:

- production code;
- the CLI, MCP and HTTP surfaces where specified;
- tests that prove the statistical, security and latency properties;
- documentation.

Work autonomously. Make reasonable engineering decisions, record them in docstrings and in `docs/decisions/` (one short file per non-obvious decision), and keep going. Stop to ask only if a requirement cannot be met without breaking an invariant listed below.

## Step 0: start from the right base

1. `git fetch origin` and check out `feat/frontier-causal-governance`. It already contains `main`. Create your working branch `feat/next-wave` from it. Do not start from `main`: the 22 first-wave features live only on that branch.
2. Before writing any code, run the full gate list below on the untouched branch and record the baseline in `docs/decisions/0000-baseline.md`: test count, coverage percentage, and the Graphify module count (currently `--max-modules 346`). If any gate is red on the baseline, fix that first in its own commit.
3. Read the first-wave modules you will extend:
   - `lineage.py`, `transparency_log.py`, `eprocess.py`, `conformal_gate.py`, `incident.py`, `promotion.py`;
   - `safety.py`, `retention.py`, `memory_ifc.py`, `redteam.py`, `outcomes.py`, `recovery.py`;
   - `interference.py`, `valuation.py`, `spatial.py`, `uplift.py`, `attribution.py`, `cleanroom.py`;
   - `compile_memory.py`, `certified_erasure.py`, `frontier_protocol.py`, `robotics/bridge.py`, `robotics/edge.py`, `robotics/ingest.py`;
   - the `docs/*.md` page for each.

   Also read the pre-existing modules `skills.py`, `experience_skills.py`, `otel_exporter.py`, `telemetry.py`, `federation.py` and `kb_packs.py`. Several features below extend them.

The product name is UniMem. The Python package directory in this repository is `commontrace/`. Keep that import path and the CLI entry point unchanged in this work; renaming is a separate change. All new user-facing text (docs, CLI help, MCP tool descriptions, evidence cards) says "UniMem".

## Repository facts you must respect

### Stack

- **Core package:** Python 3.10+. The core depends only on the standard library and PyYAML.
- **Optional dependencies** live behind extras in `pyproject.toml`: `security` (cryptography), `otel`, `serve`, `attention`, `fastembed`, `graph`, `mcap`, `robotics`, `compile`, `memory-*`, and others.
  - Import them lazily inside functions.
  - A new feature that needs a library must either degrade gracefully or raise a clear error naming the extra.
  - Add new extras where needed: `zk`, `attest`, `kafka`, `forecast`, `policy-cedar`, `kv`, `crdt`.
  - Never add a heavy dependency to the core install.
- **Hub:** `hub/` is the multi-tenant Starlette server on PostgreSQL.
  - Schema changes are Alembic migrations in `hub/alembic/versions/` with exactly one head. The first wave added three migrations; chain yours after the current head.
  - Hub tests need `HUB_TEST_DATABASE_URL` (Postgres 16).
- **Storage:** append-only JSONL and Markdown via `commontrace/_jsonl.py` and `frontmatter.locked`.
  - Facts live in SQLite (`fact_store.py`), with bitemporal `valid_from`/`valid_until` plus record time, queried with `as_of` and `known_at`.

### Surfaces and how to add to them

- **CLI:** `commontrace/cli.py` has a static `_COMMANDS` tuple and a `_COMMAND_GROUPS` map. Prefer subcommands of existing groups over new top-level commands. Where a feature needs a new group, add one group per theme, not one per feature:
  - `skill` (G1–G3)
  - `team` (G6–G9)
  - `prove` (G11–G14)
  - `person` (G15–G18)
  - `fleet` (G22–G28)
- **MCP (local):** add tools in `commontrace/mcp_server.py` (`build_server`) and list them in `commontrace/mcp_tools.py` `LOCAL_TOOLS`. Keep both in sync. Tool docstrings are read by agents, so write them for an agent reader.
- **HTTP gateway:** add routes in `commontrace/gateway.py` via `self._route(...)` with request-field docs. Regenerate the contract with `python scripts/export_openapi.py` and verify it with `--check`.
- **Hub HTTP:** regenerate with `python scripts/export_hub_openapi.py` and verify with `--check`.
- **Python API:** `commontrace/client.py` (`MemoryClient`) and `commontrace/measure.py` (`CausalMemory`).

### CI gates that must stay green

- `make lint` (ruff, line length 120, target py310).
- `make coverage` (`pytest tests e2e_tests` with an 80% gate).
- The core suite on Python 3.10 with no extras installed.
- Hub tests against real Postgres.
- Bandit `-ll`.
- Strict mypy on the files listed in `.github/workflows/ci.yml` (security-scan job). Add every new pure-logic module to that list.
- `scripts/export_openapi.py --check` and `scripts/export_hub_openapi.py --check`.
- `hub/tests/test_image_contents.py`: every `commontrace` module the Hub imports, even lazily, must be allowlisted in `.dockerignore`.
- **Graphify** (`.github/workflows/graphify.yml`): raise `--max-modules` by exactly the number of `.py` files you add under `commontrace/`, with a comment naming them. Adjust `--max-unreachable` only if a new module is genuinely an entry point, and say why.
- The local retrieval latency perf gate. No new work on the hot retrieval path without a measured budget.

### Invariants you must never break

1. **Governance before ranking.** A memory the caller cannot see must never be returned, used as a hop or bridge, influence an answer, appear in a span attribute, a change-stream event, a handoff bundle, a passport, a KV capsule or a credential. Every new path admits candidates through `search_recipes.admit`, the `_eligible_fact` rules and `memory_ifc` labels.
2. **Randomization integrity.** Assignment for every new unit (skill, writer, optimizer delta, ask decision, operator) goes through `experiment.is_held_out` with a unit-typed key, for example `skill:<id>`, `writer:<agent>` or `ask:<context>`.
   - The key is logged with its rate and context.
   - The rate is published as a hash-chained `allocation_schedule.Schedule` **before** the occasions it governs, so `integrity` audits can re-verify it.
   - When two units are randomized in the same episode, the design must be factorial or explicitly nested, with the interaction logged. Never let one unit's assignment silently change another's eligibility without recording it.
3. **No hash changes.** Never change the hash or canonical form of an existing record, receipt, schedule, ledger, revision, transparency-log leaf or certificate. New fields are omitted from hashed bodies when empty. New record types get new leaf kinds.
4. **Evidence of record.** `experiment.analyze`, `integrity.audit`, `eprocess` and the promotion certificate remain the evidence of record.
   - Every new output states whether it is a **guarantee**, an **estimate** or a **heuristic**, and lists its assumptions.
   - Never relabel an associational number as causal.
   - Where a unit is rare and power is low, say so in the output and report the minimum detectable effect instead of a verdict.
5. **Safety memories are never randomized away.** Protected hazard classes from `safety.py` are excluded from every new holdout, ask-versus-proceed trial, eviction, hibernation, compression, capsule eviction, skill-bundle trial and CRDT merge that could drop them.
   - The ask-versus-proceed trial (G22) may only randomize in contexts that a configured risk policy marks low-risk. It defaults to off.
6. **Bounded everything.** Every input from a user, tool, model, robot, peer or HTTP request is validated and size-bounded, with no unbounded loops, fan-outs, graph walks or files.
   - Non-finite floats are refused wherever a number drives a decision.
   - Network-boundary errors are clean 4xx responses, never 500s.
   - Graph traversals (contagion, delegation chains, safety-case graphs) have explicit depth and node caps.
7. **Honesty.** No fabricated statistics and no claims the code does not back.
   - Cite methods (paper and arXiv or DOI) in module docstrings. Cite them; do not vendor them.
   - Code adapted from permissively licensed projects goes into `THIRD_PARTY_NOTICES.md` with its license text.
   - Never write "first", "only" or "state of the art" in docs or CLI output.
   - Regulatory features (G16–G18) say "aligned with" or "supports", never "compliant", and their docs page carries a "not legal advice" note.
8. **Cryptographic hygiene.**
   - Use `hmac.compare_digest` for every comparison of secrets or MACs.
   - No custom cryptographic primitives: compose stdlib `hashlib`/`hmac` and the `cryptography` extra only.
   - Every signed object has a domain-separation tag and a version.
   - Keys are never logged.
   - Every token and credential has an expiry, which is checked.

### Existing modules to build on

| Area | Modules |
|---|---|
| **Experiment core** | `experiment.py`, `aipw.py`, `allocation.py`, `allocation_schedule.py`, `causal_gate.py`, `holdout_io.py`, `integrity.py`, `evidence.py`, `harm.py`, `heterogeneity.py`, `interactions.py`, `causal_policy.py`, `policy.py`, `survival.py`, `precision.py`, plus first-wave `eprocess.py`, `conformal_gate.py`, `promotion.py`, `interference.py`, `uplift.py`, `attribution.py`, `outcomes.py` |
| **Trust and governance** | `memory_authority.py`, `provenance.py`, `lesson_admission.py`, `defense.py`, `injection_guard.py`, `memory_guard.py`, `receipts.py`, `prereg.py`, `release.py`, `privacy_erasure.py`, plus first-wave `lineage.py`, `transparency_log.py`, `memory_ifc.py`, `incident.py`, `certified_erasure.py`, `safety.py`, `redteam.py`, `frontier_protocol.py` |
| **Lifecycle** | `hierarchical.py`, `fact_store.py`, `fact_conflicts.py`, `fact_consolidation.py`, `consolidate.py`, `ttl.py`, `decay.py`, `dosage.py`, `compression.py`, `recall.py`, `search_recipes.py`, `multihop.py`, `graph.py`, `skills.py`, `experience_skills.py`, plus first-wave `retention.py`, `compile_memory.py` |
| **Economics** | `value.py`, `pricing.py`, `market_settlement.py`, `marketplace.py`, `federation.py`, plus first-wave `valuation.py`, `cleanroom.py` |
| **Observability** | `telemetry.py`, `otel_exporter.py` |
| **Physical AI** | `episodes.py`, `environments.py`, `fleet.py`, `failure_signals.py`, `experience_skills.py`, `assurance.py`, plus first-wave `spatial.py`, `recovery.py`, `robotics/bridge.py`, `robotics/edge.py`, `robotics/ingest.py` |
| **Hub** | `hub/crud.py`, `hub/models.py`, `hub/server.py`, `hub/rest.py`, `hub/audit.py`, `hub/audit_export.py`, `hub/encryption.py`, `hub/kms.py`, `hub/retention.py`, `hub/manage.py`, `hub/value_billing.py`, `hub/market.py` |

## Global engineering standard for every feature

- **Module structure:**
  - Pure, stdlib-only logic goes in new modules whose docstring covers the method, its assumptions, its guarantee, and what it is not.
  - I/O (store paths, JSONL, network, CLI printing) stays separate so the math can be unit-tested and strictly type-checked.
  - The module names suggested below are a default. Merge into an existing module instead when the feature is a small extension of it.
- **Statistical tests:**
  - Every statistical claim gets a **seeded simulation test** that measures the property it claims: coverage, false discovery rate, harm rate at or below alpha, bias at or below a tolerance, or power at a stated effect size.
  - Tolerances must be honest, and each test must run in under 2 seconds.
  - Add a **mutation test**: a test that fails if the key ingredient is removed, such as propensity weights, the transfer test, the read-set check or the trust weighting. Verify it by mutating the code locally, and describe that check in the commit message.
- **Security tests:** every security feature gets adversarial tests for forged, replayed, widened, expired and laundered tokens, cross-tenant access, oversized and non-finite inputs, and confused-deputy handoffs.
- **Latency tests:** every latency claim (G21, G25, G28) gets a benchmark test that asserts a generous budget on CI hardware, plus a `benchmarks/` script that reports p50/p95/p99.
- **Surface tests:** every surface gets an end-to-end test using the existing harnesses: `tests/test_mcp_server.py` (`call`, `cli`, `_write_lesson`), `tests/test_gateway.py` and the clock-freezing fixture in `tests/test_allocation.py`.
- **Defaults:**
  - New behavior is opt-in by configuration.
  - Defaults never withhold, evict, hibernate, quarantine or block anything that is not affected today.
  - Schema additions are additive, and a test must show that old stores load unchanged.
- **Docs:**
  - Every feature gets a `docs/<feature>.md` page with what it is, when to use it, the guarantee and its assumptions, and a worked example.
  - Update the README feature section and project-status table without overclaiming.
  - Update `docs/guarantees.md` for every new guarantee, and `docs/physical-ai.md` for G22–G28.
- **Commits:** one or more conventional commits per feature, for example `feat(skills): …`, `feat(team): …`, `feat(prove): …`, `feat(person): …`, `feat(infra): …`, `feat(robotics): …` or `docs: …`. Each commit leaves the tree green.

## The features

### Phase A (self-improvement, teams, standards-native distribution): G29, G1, G2, G9, G8, G10, G20, G21

#### G29 (do first). Cross-tenant pooled evidence for rare units

**Why first.** Skills, writer agents, operators and ask decisions are much rarer than memories, so per-tenant power will often be too low for a verdict. Pooling is load-bearing for G1, G3, G9 and G22.

**Module.** `pooled_evidence.py`, extending `federation.py` and `eprocess.py`.

**Build.**
- **Inputs.** Each tenant contributes, per unit, only privacy-preserving sufficient statistics:
  - AIPW score sums, squared-score sums and counts per arm;
  - optionally clipped and noised with the Gaussian mechanism at a configured (ε, δ), with a privacy ledger per tenant.
- **Pooling.**
  - Random-effects meta-analysis, using DerSimonian–Laird and a Hartung–Knapp interval as the default.
  - An e-value combination across tenants (product of independent e-values; average when dependence is possible).
  - Leave-one-tenant-out influence diagnostics.
- **Unit identity.** Units are matched across tenants by a content hash (skill body, optimizer delta, memory class), never by tenant-private IDs.
- **Sybil resistance.** Reuse the `cleanroom.py` Sybil defenses so one tenant cannot dominate by splitting itself.
- **Every pooled verdict reports:**
  - the between-tenant heterogeneity τ²;
  - the number of contributing tenants;
  - the per-tenant power had the tenant tested alone;
  - the minimum detectable effect.

**Surfaces.**
- CLI `experiment pool …`
- Hub route that accepts signed contributions
- MCP tool `pooled_effect`

**Tests.**
- Coverage of the pooled interval at or above nominal minus tolerance under simulated heterogeneity.
- The DP noise calibration matches the stated ε.
- A Sybil split does not move the pooled estimate beyond a tolerance.
- Mutation test: removing the between-tenant variance makes coverage fail.

#### G1. Causal Skill Foundry

**Module.** `skill_foundry.py`, extending `skills.py` and `experience_skills.py`.

**Build.**
- **Induction.**
  - Induce candidate skills from successful trajectories, either prose procedures or programmatic helpers, following Agent Workflow Memory (ICML 2025) and Agent Skill Induction (arXiv 2504.06821).
  - The induction model is pluggable. With no LLM configured, the core ships a deterministic template-based inducer used in tests.
- **Pre-screen.**
  - Before any live traffic, each candidate faces verifier tasks mined from measured failure clusters, using Self-Challenging (NeurIPS 2025) style tasks: an instruction, a verifier, and passing and failing cases.
  - Candidates that fail are rejected with a reason.
- **Randomized exposure trial.**
  - In eligible episodes the skill is present in or absent from the catalog the agent sees, with unit key `skill:<content-hash>`.
  - Outcomes flow through AIPW and the e-process.
  - Harm is charged to the `conformal_gate` budget.
  - `promotion.py` issues or refuses a promotion certificate.
- **Bundle size as a factor.** Bundles of up to 3 skills versus larger bundles form a randomized factor (SkillsBench, arXiv 2602.12670, reports that focused bundles beat exhaustive ones). Report the bundle-size effect.
- **Retirement.** An e-process-detected decay or a drift test retires a promoted skill.
- **Publishing.**
  - Promoted skills are written as Agent Skills folders: a `SKILL.md` with standard `name`/`description` frontmatter, plus a signed `EVIDENCE.json` next to it.
  - `EVIDENCE.json` holds the effect, interval, segments, harm budget spent, certificate ID and transparency-log inclusion proof.
  - Do not add non-standard keys to the `SKILL.md` frontmatter; reference the evidence file from the body.
- **Pooling.** Pool across tenants through G29 when configured.

**Surfaces.**
- CLI `skill induce|screen|trial|status|publish|retire`
- MCP tools `skill_catalog` (returns only the skills this episode may see) and `skill_evidence`

**Tests.**
- The trial controls harm at alpha in simulation.
- A useless skill is never promoted beyond the error rate.
- Bundle-size estimation is unbiased.
- The published folder round-trips and the evidence signature verifies.
- Mutation test: exposing the skill regardless of assignment makes the bias test fail.

#### G2. Holdout-paired RL data broker and causal-priority replay

**Module.** `rl_export.py`.

**Build.**
- **Preference datasets.**
  - Export DPO-style preference datasets in JSONL (`prompt`, `chosen`, `rejected`, `meta`) built from with-memory and without-memory episodes matched by task cluster and context.
  - Be honest in the metadata: these are randomized-arm pairs matched on observed task features, not same-instance counterfactuals. `meta` carries the population effect, interval, segment and propensities.
- **Per-episode advantages.** Export advantage labels for GRPO-style training: the outcome minus the AIPW baseline, with propensity included.
- **Replay sampler.** Weight episodes by the confidence-bounded uplift of the memories they used, times the uncertainty. Use the lower bound for exploitation and the width for exploration, with documented weights.
- **KL budget.** Each export carries a KL-to-base budget field (RL's Razor, arXiv 2509.04259). If token logprobs are supplied, the export checks the budget; otherwise it records "unchecked".
- **Connectors.** Writers emit JSONL and record schemas compatible with Microsoft Agent Lightning (arXiv 2508.03680) and with an external grader endpoint for reinforcement fine-tuning.
  - Never call vendor APIs in tests.
  - Document the field mapping, and mark it "unverified against vendor schema" until a fixture from the vendor's docs is added.
- **Governance.** Exports pass through `memory_ifc` egress checks and erasure.
  - Erased subjects are excluded.
  - Each export is logged with its manifest digest so a later erasure can find and flag affected exports.

**Surfaces.**
- CLI `experiment export-rl --format dpo|advantage|replay`
- Gateway route for streaming export

**Tests.**
- Pairs never cross an information-flow label boundary.
- Erased content never appears in an export.
- Advantage labels are unbiased in simulation.
- An export manifest is flagged after a later erasure.

#### G9. Writer-level holdouts and an agent reliability ledger

**Module.** `writer_holdout.py`, extending `interference.py` and `attribution.py`.

**Build.**
- **Writer masking.** In a randomized fraction of team episodes, mask everything written by agent A or role R (unit key `writer:<id>`). This gives an interventional estimate of each writer's memory contribution to team outcomes.
- **Nesting.** Writer holdouts are nested over memory holdouts. When a writer is masked, that writer's memories are logged as "ineligible (writer-masked)" for memory-level analysis, never as held out.
- **Reliability score.** Score each writer online from the downstream outcomes of its memories, in the Σ-Mem style (arXiv 2607.27958). Use it as a prior in admission, retrieval ranking and routing. It is labeled an **estimate**.
- **MAST tags.** Team incidents in `incident.py` get MAST failure-mode tags (arXiv 2503.13657): 14 modes in 3 categories, assigned by rules plus an optional LLM tagger.

**Surfaces.**
- CLI `team writers|reliability|mask`
- MCP tool `writer_reliability`

**Tests.**
- Writer-effect coverage in simulation.
- The nested design does not bias memory-level estimates; a mutation that counts writer-masked memories as held out must fail.
- Reliability ranking recovers planted good and bad writers.

#### G8. Contagion radius and independent-source retrieval

**Module.** `contagion.py`, extending `lineage.py`.

**Build.**
- **Read graph.** Record a **read graph**: which agent, episode and output read which memory, with bounded retention. It sits alongside the write lineage.
- **Contagion radius.** `contagion_radius(memory_id)` returns every agent, episode, derived memory, skill, export and capsule that read the memory or descends from it, with depth and node caps. One action feeds the result into `incident.py` freeze.
- **Effective independent sources.** At retrieval, group results by lineage root and compute the effective number of independent sources (n_eff, a diversity index over roots). Annotate the results, and optionally down-weight agreement that traces to one root, following CAMA's "memory correlation bias" (arXiv 2608.19701).
- **Re-retrieval.** When n_eff falls below a threshold on a decision-relevant query, trigger an independent-source re-retrieval through `search_recipes.admit`.

**Surfaces.**
- CLI `lineage contagion`
- Gateway `/v1/recall` gains optional `independence` annotations, which are additive in the response
- MCP tool `contagion_radius`

**Tests.**
- The radius is complete on planted graphs.
- Caps hold on adversarial graphs.
- A false majority from one root is detected.
- Admission is never bypassed by the re-retrieval.

#### G10. OpenTelemetry memory spans with causal extensions

**Module.** `otel_memory.py`, extending `otel_exporter.py` and `telemetry.py`.

**Build.**
- **Spec-conformant spans.** Emit spans for memory operations following the OpenTelemetry GenAI semantic conventions: create, search, update, upsert and delete memory, using `gen_ai.memory.store.id`, `gen_ai.memory.record.id` and `gen_ai.memory.record.count`. Pin the convention version in a constant and in the docs.
- **Extension attributes** under a vendor namespace `unimem.*`:
  - writer agent;
  - delegation-chain digest (G6);
  - lineage root and depth;
  - label ceiling;
  - isolation level, read version and staleness (G7);
  - policy decision;
  - holdout arm and propensity;
  - n_eff (G8).
- **Privacy.** Query text and record contents are never emitted unless explicitly enabled, and are always filtered by labels.

**Surfaces.**
- Configuration only, plus `docs/otel.md` with a Grafana/Datadog query example that slices task outcomes by holdout arm.

**Tests.**
- The span schema matches the pinned conventions.
- Spans for invisible memories are never emitted.
- With the `otel` extra absent, the feature degrades cleanly.

#### G20. Memory change streams, erasure propagation and byte-verified receipts

**Modules.** `change_stream.py` and `erasure_verify.py`, extending `certified_erasure.py`.

**Build.**
- **Typed stream.** Publish the append-only log as a typed stream of events: assert, retract, contradict, supersede, erase, quarantine.
- **Transports.**
  - SSE on the gateway.
  - MCP notifications.
  - Kafka/Redpanda behind the `kafka` extra.
  - Cursors are resumable.
- **Standing queries.** Agents register standing queries (an entity, predicate or namespace filter, evaluated through admission) and receive diffs with provenance.
- **Erasure acknowledgement.**
  - Erase events require a downstream acknowledgement, recorded in an erasure-ack ledger.
  - Unacknowledged subscribers are surfaced and, past a deadline, cut off.
- **Byte-verified receipts.** After erasure, a verification scan of the raw local stores searches the bytes for fingerprints of the erased content: exact strings, n-gram hashes, and embedding byte patterns where vectors are stored.
  - Stores covered: SQLite pages, including freelist pages, after `secure_delete` and `VACUUM`; JSONL; vector index files.
  - Soft-deleted HNSW vectors are known to be recoverable (arXiv 2606.18497), so the scan is the point.
  - The resulting "purged and verified" receipt is logged in the transparency log.
  - Stores that cannot be scanned are listed as "unverified", never silently passed.

**Surfaces.**
- CLI `stream tail|subscribe|acks`
- CLI `erasure verify`
- Gateway SSE route
- MCP tool `subscribe_beliefs`

**Tests.**
- Standing queries never leak invisible facts.
- Resume after a disconnect is exactly-once at the cursor level.
- The scan finds planted residue in a soft-deleted SQLite row and passes after `VACUUM`.
- A receipt is never issued while any subscriber is unacknowledged.

#### G21. Hibernation, residency and ACL-native tiering

**Module.** `tiering.py`.

**Build.**
- **Hibernation.** Idle namespaces (agent, user or robot) migrate from the hot tier to a cold tier.
  - Cold tiers: a local directory, or S3/GCS-compatible object storage behind an extra.
  - Each namespace has a region tag.
- **Residency policy.** The policy refuses reads, writes and migration outside the namespace's allowed regions, with a 4xx at the boundary.
- **Pre-warm.** Hooks on session start, robot boot and task assignment pre-warm the namespace, so cold latency never lands on a user turn when the hook fired in time.
- **ACL-native filtering.** Push tenant and label predicates into the vector backends' native filters (Lance `where`, pgvector `WHERE`) instead of post-filtering, in the spirit of ACORN (arXiv 2403.04871).
  - Keep `search_recipes.admit` as the final check.
  - Prove with a test that results are identical to post-filtering.
- **Blue/green re-embedding.** Changing the embedding model is a blue/green rebuild with a recall-delta report on a held-out query set. Cut-over is atomic and rollback is one command.
- **Safety.** Safety memories are never hibernated.

**Surfaces.**
- CLI `tier status|hibernate|warm|residency|reembed`

**Tests.**
- Residency violations are refused.
- Hibernate and warm round-trip losslessly.
- The latency benchmark after pre-warm meets the hot-tier budget.
- Native and post-filter results match.

### Phase B (teams, optimizers, personal memory into production): G6, G7, G3, G4, G5, G14, G15, G16, G18, G22

#### G6. Delegated memory capabilities, handoff bundles and revocation taint

**Module.** `capabilities.py`.

**Build.**
- **Capability tokens.**
  - Local tokens are macaroon-style HMAC chains (stdlib), which allow offline attenuation by sub-agents.
  - Cross-organization tokens are Ed25519-signed, behind the `security` extra.
  - Both follow the attenuating agent-token pattern of IETF draft-niyikiza-oauth-attenuating-agent-tokens: parent hash binding and fail-closed caveats.
  - Caveats name namespaces, a label ceiling, operations (read, write, forget, export), memory classes, purpose (G16), expiry and the audience agent key.
- **Delegation stamping.**
  - Every write is stamped with the delegation-chain digest, with the full chain stored in lineage.
  - Every read is checked against the leaf capability.
- **Handoff bundles.**
  - On an agent handoff, emit a minimal signed bundle: filtered by the recipient's clearance, redacted by `memory_ifc`, bound to the recipient's key, short-lived, and carrying sticky labels so the receiver's writes inherit them.
  - Ship adapters for the OpenAI Agents SDK handoff `input_filter` and for an A2A extension. Document the shapes, and do not import those SDKs in core.
- **Revocation taint.**
  - Revoking a root or intermediate capability quarantines every memory written under it, using the contagion read graph from G8 for downstream impact.
  - A memory is released only if another still-valid chain independently authorized it.

**Surfaces.**
- CLI `team grant|attenuate|revoke|handoff`
- MCP tool `handoff_bundle`
- Gateway and Hub accept a capability header

**Tests.**
- Widening attacks fail, including adding namespaces, raising the ceiling, extending expiry and swapping the audience.
- Replayed and expired tokens fail.
- Confused-deputy handoffs fail.
- Revocation quarantines exactly the derived set.
- A test shows the chain digest in the G10 span.

#### G7. Team-memory isolation levels (stale-generation guard)

**Module.** `team_isolation.py`, extending `fact_store.py`.

**Build.**
- **Read-sets.** A team write declares its read-set: the (fact ID, transaction time) versions it was generated from.
- **Isolation levels.** Each space has a level:
  - `none`
  - `snapshot`
  - `serializable`: the read-set is validated at commit
  - `sequenced`: commit order is pinned per work item
- **Conflicts.** A conflicting commit returns `STALE_GENERATION` with the superseding facts and offers `regenerate` or `merge`. Merge is semantic: it goes through `fact_conflicts` and produces a reviewable contradiction, never last-write-wins.
- **Leases.** Leases cover exclusive work items, with expiry and fencing tokens.
- **Check order.** Contradiction detection runs **before** dedup on every team write. Turn the MemClaw ordering bug (arXiv 2606.24535) into a regression test.
- **Read-path scoping.** Sub-tenant scope is enforced on every read path, including fetch-by-ID. Add a test for the MemClaw GET-by-id bypass.

**Surfaces.**
- CLI `team isolation|lease`
- Gateway write routes accept an optional `read_set` field

**Tests.**
- Reproduce the lost-update anomaly under `none` and show it is prevented under `serializable`.
- A stale fencing token is refused.
- A contradictory write is detected even when it is a near-duplicate.
- Fetch-by-ID across sub-tenants is refused.

#### G3. Optimizer proposal adapter with per-delta attribution

**Module.** `optimizer_intake.py`.

**Build.**
- **Intake.** Accept candidate edits in a documented generic JSON schema: base artifact, delta, optimizer name, claimed validation score and parent.
  - Ship adapters for GEPA Pareto candidates (arXiv 2507.19457), ACE playbook bullets (arXiv 2510.04618), Training-Free GRPO experiences (arXiv 2510.08191), DSPy compiled programs and code-edit diffs.
  - Mark each adapter "format unverified" until a fixture from that project's documentation is added.
- **Trial.** Each candidate gets a randomized arm (`delta:<hash>`), anytime-valid monitoring and harm-budget spend through the existing promotion gate.
- **ACE bullets.** Treat each bullet as a memory-level unit, so per-memory holdouts attribute effect per bullet. Prune bullets that are causally harmful even when the optimizer's reflector approved them.
- **Feedback.** Return losing candidates to the optimizer as negative evidence, in a reflection-ready JSON format.
- **Pareto frontiers.** Build per-segment Pareto frontiers from the randomized data, never from the optimizer's own validation scores.

**Surfaces.**
- CLI `skill optimize intake|status|frontier`
- MCP tool `submit_candidate`

**Tests.**
- A candidate with a high claimed score but zero true effect is not promoted.
- Per-bullet attribution recovers planted harmful bullets.
- The frontier is computed from randomized data only; a mutation that uses claimed scores must fail.

#### G4. Misevolution and forgetting sentinel

**Module.** `sentinel.py`.

**Build.**
- **Transfer-test quarantine.**
  - An experience, lesson or skill whose randomized effect is positive in its origin segment and negative in other segments is quarantined before dreaming, reflection, skill induction, compilation or bundle export can generalize it.
  - The test uses heterogeneity e-values with BH or e-BH across segments.
  - This is the signature of experience-poisoning attacks such as OEP (2026) and of misevolution (arXiv 2509.26354).
- **Update interference ledger.**
  - Every update (compiled cartridge, adapter, merge, skill bundle or playbook delta) is deployed in a randomized arm.
  - The effects of a canary set of previously proven memories are estimated separately in update and control arms.
  - The ledger records each prior effect's change, KL to base when logprobs are supplied, and adapter provenance (digest and signer).
  - Updates that regress proven effects beyond tolerance are blocked by the promotion gate.
- **Drift monitors.** Monitor the four misevolution pathways (model, memory, tool, workflow) with e-process alarms.

**Surfaces.**
- CLI `sentinel status|quarantine|ledger`
- Incident integration

**Tests.**
- A planted OEP-style poison is quarantined while a benign heterogeneous memory is not, with the false-quarantine rate below the stated level.
- A planted forgetting update is blocked.
- Mutation test: removing the cross-segment test lets the poison through.

#### G5. Anti-sycophancy personalization audit

**Module.** `sycophancy.py`.

**Build.**
- **Secondary metrics.** Add named secondary metrics to outcomes, additively: `accuracy` and `agreement`. A memory or memory class (for example, the user's stated opinions) is held out on matched queries, and three effects are estimated separately: task outcome, factual accuracy and agreement rate.
- **Flattering memories.** Memories that raise agreement without raising outcome or accuracy are flagged as "flattering" and down-weighted by the causal gate.
- **Factual queries.** A "memory is not evidence" rule applies: personal-opinion memories are excluded from factual-claim contexts unless their measured accuracy effect is non-negative.
- **Cross-domain leakage.** Context-scope rules block memories from leaking across domains.
- **Acceptance harnesses.** Ship harness adapters for PersistBench (arXiv 2602.01146) and MemSyco-Bench. Datasets are downloaded by the user and never bundled; check their licenses.

**Surfaces.**
- CLI `experiment sycophancy`
- Report section in the evidence pack

**Tests.**
- A planted flattering memory is flagged.
- A planted genuinely helpful personal memory is not.
- All three effects are estimated without bias in simulation.

#### G14. Verified memory policy with change proofs

**Module.** `policy_proof.py`.

**Build.**
- **Policy language.** Express memory read, write, recall and export permissions in a small declarative policy language over finite domains: principal attributes, actions, memory classes, labels and purposes.
- **No-widening check.**
  - On every policy change, compute the set difference of authorized (principal-class, action, memory-class) tuples between the old and new policy, by exhaustive evaluation over the finite domain. This is sound and complete for finite domains, and the docs must state that bound.
  - Emit a "no-widening" certificate or a widening diff as a new transparency-log leaf kind.
- **Cedar adapter.** Behind the `policy-cedar` extra, an adapter exports and imports Cedar policies and uses Cedar's analysis when available (Cedar is formally modeled in Lean, arXiv 2403.04651).
- **ReBAC.** An optional OpenFGA/SpiceDB relationship check supplies authorized IDs as a retrieval pre-filter.
- **Change gate.** Policy changes that widen access require a review receipt (`lesson_admission`-style) before activation.

**Surfaces.**
- CLI `prove policy diff|certify|activate`
- Hub route

**Tests.**
- A planted widening is always detected.
- An equivalent rewrite yields "no widening".
- A domain-size bound is enforced.
- An unreviewed widening never activates.

#### G15. Signed memory passport, import firewall and selective-disclosure credentials

**Modules.** `passport.py` and `credentials.py`.

**Build.**
- **Passport.** A user's or tenant's memory exports as a signed, content-addressed bundle: a Merkle manifest of memories with provenance, confidence, timestamps, consent scope (G16), labels and per-segment capability grants (G6).
  - Import of a UniMem passport round-trips losslessly.
  - Align the field names with Portable Agent Memory (arXiv 2605.11032) where possible, and document the mapping.
- **Import firewall.** Inbound summaries from other assistants (pasted text or files) are parsed into candidate memories marked "unverified, self-reported via <source>". They are scanned with `injection_guard` and `defense`, and staged for review. They are never auto-admitted.
- **Selective-disclosure credentials.** A single fact can be issued as a W3C VC 2.0 credential secured with SD-JWT (selective disclosure) under the `security` extra, with expiry and a status-list revocation entry.
- **Model identity.** The producing model is identified by digest, compatible with OpenSSF model signing.
- **Erasure.** Erased subjects can never appear in a passport or credential.

**Surfaces.**
- CLI `person export|import|stage|issue|revoke|verify`
- MCP tool `verify_credential`

**Tests.**
- Round-trip export and import.
- A tampered bundle fails.
- Injection in an import is caught and staged.
- Undisclosed claims in an SD-JWT presentation cannot be recovered.
- A revoked credential fails verification.

#### G16. Purpose-bound memory with consent receipts

**Module.** `purpose.py`.

**Build.**
- **Purpose and consent.** Each memory may carry a purpose set and a consent record. Fields are aligned with ISO/IEC TS 27560 consent records and the W3C Data Privacy Vocabulary (DPV) purpose taxonomy.
- **Retrieval filter.** Retrieval is filtered by the requesting agent's declared purpose: a shopping agent never sees health memories. The purpose is carried in G6 capabilities.
- **Expiry.** When a purpose expires, the memory is forgotten through `retention`, `certified_erasure` and the G20 stream.
- **User view.** "What do you remember about me, and for what?" lists the memories, including inferred ones, with their purposes and consent.

**Surfaces.**
- CLI `person purposes|consent|explain`
- Gateway recall gains an optional `purpose` field
- MCP tool `explain_memory_about_me`

**Tests.**
- A purpose mismatch never retrieves.
- Expiry triggers erasure.
- Inferred memories appear in the explanation view.

#### G18. Regulated-subject write-time policy compiler

**Module.** `subject_policy.py`.

**Build.**
- **Rule packs** keyed by subject class (child under 13, adult, care resident) and deployment context (workplace, education, home, clinical), enforced at **write time**:
  - `coppa` (2025 amended rule): biometric identifiers handled as personal information, written retention schedules, and deletion when the retention purpose ends.
  - `eu_ai_act_art5_1f`: in workplace and education contexts, quarantine emotion-labeled memories inferred from biometric channels (voice, face, physiology), except for medical or safety purposes. The prohibition has applied since 2 February 2025.
- **Modality tags.** Ingest must tag each memory's source modality. A write with an unknown modality into a regulated context is held for review rather than written.
- **Compliance evidence.** Every allow, deny or quarantine decision is logged.

**Surfaces.**
- CLI `person policy compile|test|explain`

**Tests.**
- An inferred emotion from voice in a workplace context is quarantined.
- The same inference in a clinical context with a medical purpose is allowed.
- A child's biometric gets a retention schedule.
- An unknown modality is held for review.

#### G22. Human-help ledger: correction quality and measured value of asking

**Module.** `human_help.py`, extending `recovery.py` and `outcomes.py`.

**Build.**
- **Intervention ledger.** Every teleop takeover or human correction, from a robot or an agent, becomes a governed record: the pre-intervention window, the corrective segment, the operator ID and skill tier, the outcome, and an advantage label from the outcome engine.
- **Correction quality.**
  - Score each operator's corrections by their downstream outcome.
  - Quarantine low-value correctors' data before it enters training exports (G2), with review and appeal.
  - This addresses RECAP-style training (arXiv 2511.14759), which trusts interventions wholesale.
- **Measured value of asking.**
  - In contexts the risk policy marks low-risk only, randomize ask versus proceed (`ask:<context>:<question-type>`) to measure the realized value of information per context and question type.
  - Combine it with belief-based value-of-information priors (EVPI clarification, arXiv 2511.08798) as an empirical-Bayes prior.
- **Ask-budget planner.** A fleet-level planner returns "ask", "don't ask" or "which mode" under an operator-minutes budget.
- **KPIs.** Report interventions per hour, operator-minutes saved and an intervention-cost report.

**Surfaces.**
- CLI `fleet interventions|operators|ask-policy`
- MCP tool `should_ask`
- ROS 2 service via `robotics/bridge.py`

**Tests.**
- Ask randomization never happens in contexts not marked low-risk, or in safety contexts.
- Value-of-asking estimates are unbiased in simulation.
- A planted bad operator is quarantined while a good one is not.
- The planner respects its budget.

### Phase C (proofs that leave the building, households, governed inference, robot fleets): G11, G12, G17, G19, G23, G24, G26, G27

#### G11. Zero-knowledge effect certificates

**Modules.** `zk_effect.py`, plus `zk/` for the optional prover program.

**Build.**
- **Commitments.** Commit each trial's per-episode (assignment, propensity, outcome) rows as a Merkle root in the transparency log.
  - Use fixed-point integer encoding with documented scale and overflow bounds.
  - Assignment is seeded by a verifier-supplied nonce, committed before outcomes exist, to stop cherry-picking.
- **Statements.** Define a canonical statement format: "the Hajek/IPW effect estimate over the committed rows is E, and the lower bound at level α is L", or "harm rate ≤ B".
- **Prover backend interface** with three backends:
  1. A **transparent audited recomputation** backend (core, stdlib): a verifier given the rows checks the Merkle root and recomputes. This is **not zero-knowledge**, and its output says so.
  2. A **zkVM** backend behind the `zk` extra: a guest program in `zk/` (Rust, for RISC Zero or SP1) recomputes the estimator over the committed rows and outputs only the statement. Build it in CI only when the toolchain is present.
  3. A hook for chunked circuits for large trials. ZK-SR117 (arXiv 2608.02664) shows chunking is needed past about 30K rows.
- **Optional DP.** Statements can be DP-noised, with the noise seed bound into the proof.
- **Scope.** Prove the estimator, never the agent runs.

**Surfaces.**
- CLI `prove effect|verify`

**Tests.**
- The transparent backend catches altered rows and altered statements.
- Fixed-point error stays within the documented bound.
- The zkVM test is skipped with a clear reason when the toolchain is absent.
- The output never says "zero-knowledge" for the transparent backend.

#### G12. Attested memory enclave with log-gated key release

**Module.** `attested_keys.py`, extending `hub/kms.py` and `transparency_log.py`.

**Build.**
- **Key release.** A tenant data key unwraps only for a worker whose attestation evidence matches a build measurement registered in the transparency log. This applies the Private Cloud Compute pattern to memory.
- **Verifier interface.**
  - Pluggable backends for AMD SEV-SNP, Intel TDX and NVIDIA GPU attestation, behind the `attest` extra.
  - A core **simulated** verifier with test keys, clearly labeled, so the policy logic is fully tested.
- **Client policy.** Clients refuse nodes whose measurement is not in the log.
- **Sealed trials.** Sealed trials in `cleanroom.py` emit an attestation quote bound to the result digest.
- **Cross-organization joins.** For cross-organization joins, both parties release keys only to the same attested join binary.

**Surfaces.**
- CLI `prove measurements register|list`
- Hub key-release route

**Tests.**
- An unregistered measurement is denied.
- A replayed quote is denied (nonce).
- A revoked measurement is denied.
- A simulated end-to-end sealed trial produces a bound quote.

#### G17. Household and bystander memory scopes

**Module.** `household.py`, extending `spatial.py` and `memory_ifc.py`.

**Build.**
- **Person-keyed scopes.** Scopes are `resident`, `household`, `guest`, `bystander` and `caregiver`.
- **Multi-party consent.** Consent can come from several parties, including a proxy consent for a resident who lacks capacity.
- **Bystanders.** Unrecognized persons are **ephemeral by default**: their memories are never consolidated, auto-expire, and their face and voice embeddings are never stored.
- **No-memory rooms.** These extend `spatial.py` regions: perception there is never persisted.
- **Household export.** Export works through G15.
- **Identity.** Identity signals come from the robot stack via `robotics/bridge.py`. When identity is unknown, treat the person as a bystander.

**Surfaces.**
- CLI `person household …`
- MCP tool `household_scope`

**Tests.**
- Bystander data never survives consolidation.
- A no-memory room persists nothing.
- Proxy consent works and is logged.
- An unknown identity defaults to bystander.

#### G19. Governed KV capsules

**Module.** `kv_capsules.py`.

**Build.**
- **Capsules.** For memories past a promotion threshold, compile per-memory or per-block KV for each model fingerprint.
  - Key: `H(tenant salt, memory hash, model hash)`. Tenants never share capsules, which closes the cross-tenant KV prompt-leak path (NDSS 2025).
  - Compression is pluggable (CacheGen-style).
  - Capsules register in a tiered pool through an adapter interface for LMCache, Dynamo KVBM or Mooncake, behind the `kv` extra. The core ships an in-memory/disk reference backend for tests.
- **Composition.** Capsules compose with CacheBlend-style partial recompute; this is the adapter's responsibility, and it is documented.
- **Eviction.** Retracting, erasing or quarantining a memory evicts its capsule on every tier and emits a G20 event. The erasure receipt lists capsule tiers.
- **Scope.** Not applicable to closed APIs; say so.

**Surfaces.**
- CLI `tier capsules build|status|evict`

**Tests.**
- Cross-tenant capsule reuse is impossible.
- Erasure evicts every tier.
- A capsule for a stale model fingerprint is never served.

#### G23. World-model trust ledger with sim-gated holdouts

**Module.** `world_model_trust.py`, extending `outcomes.py` (the sim-to-real calibration).

**Build.**
- **Registry.** Register world models by digest.
- **Verdicts.** Before a memory, recovery or checkpoint reaches real robots, replay it in a registered world model from stored real failure states and record the verdict.
- **Trust ledger.** Track per-task agreement between world-model verdicts and later real outcomes, with an anytime confidence sequence.
- **Action-following check.** Perturb the commanded action and require the predicted rollout to change. World models often ignore off-expert actions (arXiv 2608.24885).
- **Rules.**
  - A low-trust model may **veto but never promote**.
  - Real trials are spent first on the top and bottom candidates.
  - Trust decays with model or embodiment change.

**Surfaces.**
- CLI `fleet worldmodel register|trust|gate`

**Tests.**
- A model that ignores actions fails the check.
- Low trust can veto but never promote.
- The trust bound has coverage in simulation.

#### G24. Force and proprioceptive signature memory

**Module.** `robotics/signatures.py`.

**Build.**
- **Profiles.** For each (skill, embodiment, site), store nominal multivariate profiles of joint current, force/torque, tactile and temperature, with task phase and payload as covariates. Ingest them from MCAP or ROS 2 via the existing bridge.
- **Bands.**
  - The core uses robust per-phase quantile bands (median/MAD).
  - Behind the `forecast` extra, Chronos-2 style zero-shot quantile forecasts with covariates.
  - Document that applying time-series foundation models to robot joint data is not yet established in published work.
- **Anomalies.** Exceedances become anomaly memories linked to the episode and the recovery registry.
- **Drift.** Slow bitemporal drift becomes a maintenance memory, queryable with questions like "which wrists' currents drifted since firmware X?".

**Surfaces.**
- CLI `fleet signatures build|check|drift`
- MCP tool `signature_check`

**Tests.**
- A planted anomaly is detected at the stated false-alarm rate.
- A planted drift is found at the right firmware boundary.
- Edge evaluation stays within a latency budget on downsampled windows.

#### G26. Collective site memory across vendors and dead zones

**Modules.** `robotics/vda5050.py`, `robotics/site_memory.py` and `mesh_crdt.py`.

**Build.**
- **Ingest.**
  - VDA 5050 3.0 state, zone and path-sharing messages. Validate with a schema subset written from the public spec; do not copy spec text.
  - Open-RMF adapters.
  - Both feed bitemporal spatial memory as a vendor-neutral layer.
- **Scene-graph merge.** Merge per-robot scene graphs layer by layer, in the style of Hydra-Multi (arXiv 2304.13487), so robots with different sensors contribute.
- **Site rules.** Learn site rules (for example, "zone Z congests 14:00–15:00") as estimates with intervals, and push them back as zone suggestions.
- **Dead zones.**
  - Robots sync peer to peer through a signed op-based CRDT memory document: an OR-set plus registers, with ops signed per robot key.
  - Automerge is used behind the `crdt` extra when present.
  - Semantic conflicts become reviewable contradictions, not last-writer-wins.
  - A revoked robot's ops are rejected and tainted.
- **Safety.** Safety memories cannot be removed by a merge.

**Surfaces.**
- CLI `fleet site ingest|merge|rules|mesh`

**Tests.**
- CRDT convergence under arbitrary op orders (property test).
- A forged op from an unknown or revoked key is rejected.
- A semantic conflict produces a contradiction.
- Safety memory survives a hostile merge.

#### G27. Living safety-case evidence graph

**Module.** `safety_case.py`, extending `safety.py`.

**Build.**
- **Graph.** A goal-structured graph of claims, arguments and evidence (GSN-like, aligned with UL 4600's claim-argument-evidence style). Evidence nodes link to episodes, trials, certificates and scenarios.
- **Mining.** Keep mined field events only when they are failures or near-failures **and** reproducible.
- **Scenario export.** Lift each kept event to a logical scenario exported in OpenSCENARIO DSL 2.1 text form, as a template generator. Validate the syntax only, and document the subset.
- **Staleness.** Every model, software or world-model change marks the affected evidence stale, through dependency edges, and queues re-validation using G23 sim-gated replay where trusted.
- **Coverage view.** Show hazards with weak, stale or missing evidence.

**Surfaces.**
- CLI `fleet safetycase build|stale|coverage|export`
- A docs page with an end-to-end example

**Tests.**
- A model change stales exactly the dependent evidence.
- Coverage reports a planted gap.
- An exported scenario parses with the documented grammar subset.

### Phase D (latency- and safety-critical, plus go-to-market packaging): G13, G25, G28, G30

#### G13. Oblivious recall

**Module.** `oblivious.py`.

**Build.** A querier retrieves top-k memories from a Hub without the Hub learning the query or the results.

- **Reference mode (core, stdlib): two-server PIR.**
  - The index is clustered, and cluster centroids are public to authorized queriers.
  - The client picks clusters locally and fetches cluster blocks through two-server XOR PIR from non-colluding servers.
  - It re-ranks locally.
  - Document the leakage: the number of clusters fetched, and the cluster count.
- **Interfaces** for single-server ORAM-over-HNSW (Compass, ePrint 2024/1255) and verifiable PIR (ePrint 2026/923) as future backends.
- **Labels.** The served database is per-label-ceiling, so PIR never serves content above the querier's clearance.
- **Latency.** About +1 s per query is acceptable; never use this in robot control loops.

**Surfaces.**
- CLI `prove recall-oblivious`
- Paired Hub routes

**Tests.**
- Either server's view is independent of the query (a statistical test over its transcripts).
- Correctness matches the plaintext top-k on the clustered index.
- Content above the clearance is never served.

#### G25. Governed in-loop VLA working memory

**Module.** `robotics/working_memory.py`.

**Build.**
- **Service.** An on-robot keyframe and working-memory service behind the policy, with an interface compatible with MemER (arXiv 2510.20328) and MemoryVLA (arXiv 2508.19236).
- **Hard retrieval budget.** Default 5 ms inside a roughly 40 ms control step; this is a design target, configurable. Over budget, it returns the best result so far and flags it.
- **Eviction.** Learned eviction using utility from `retention.py`. Protected safety keyframes are never evicted.
- **Consolidation.** At episode end, predictive keyframes are promoted to long-term episodic memory with an audit trail.
- **No network calls in the loop.**

**Surfaces.**
- Python API
- ROS 2 node in `robotics/bridge.py`

**Tests.**
- p99 retrieval stays within budget on CI, with a generous multiplier.
- Safety keyframes survive eviction pressure.
- Consolidation is audited.

#### G28. Memory-to-action temporal-logic shield

**Module.** `robotics/shield.py`.

**Build.**
- **Spec language.** A small bounded-time Signal Temporal Logic (STL) language: always, eventually and until with time bounds, predicates over named signals.
  - Robustness semantics computed in stdlib.
  - Specs can be attached to the memory itself, for example "this grasp memory is valid only if force < X within T s".
- **Runtime monitor.** Every robot action derived from a recalled memory passes through the monitor. Violations **block** the action or **repair** it with a safe fallback from the recovery registry. The verdict is written into the memory's lineage.
- **Scope.** RoboGuard (arXiv 2503.07885) is the motivating evidence. This is a runtime monitor, not a certified safety function. The docs must say it complements and never replaces ISO 10218 functional safety.

**Surfaces.**
- Python API
- ROS 2 service
- CLI `fleet shield check|spec`

**Tests.**
- Robustness matches hand-computed values.
- A planted violating action is blocked.
- A safe action passes.
- The monitor latency budget holds.
- A malformed spec is rejected.

#### G30. Vertical packs and reference demos

**Module.** Extend `kb_packs.py`, or add `packs.py` if cleaner.

**Build.** Eight installable configuration packs. Each pack bundles defaults, rule packs, report templates, a docs page and a runnable synthetic demo that exercises the features end to end on seeded data with no network.

| Pack | Features |
|---|---|
| `coding-agent` | G1, G2, G3, G7, G8, G10 |
| `support-outcome` | G3, G5, G16, G20, G22 |
| `bank-ai-governance` | G4, G6, G9, G11, G12, G14 |
| `healthcare` | G11, G12, G13, G16, G18 |
| `robot-fleet` | G22, G23, G24, G25, G26, G28 |
| `autonomy-safety-case` | G23, G27, G28 |
| `home-eldercare-robot` | G5, G15, G17, G18 |
| `sovereign-airgapped` | G12, G14, G19, G21, with all network-dependent features disabled |

Each demo prints a one-page buyer report: what was measured, the effects with intervals, harm budget spent, and what is a guarantee versus an estimate. The report contains no invented numbers; everything comes from the seeded run.

**Surfaces.**
- CLI `pack list|install|demo`

**Tests.** Each demo runs in CI under 10 seconds and its report is deterministic for its seed.

## Order of work and definition of done

1. **Step 0:** base, baseline and reading.
2. **Phase A:** G29 → G1 → G2 → G9 → G8 → G10 → G20 → G21.
3. **Phase B:** G6 → G7 → G3 → G4 → G5 → G14 → G15 → G16 → G18 → G22.
4. **Phase C:** G11 → G12 → G17 → G19 → G23 → G24 → G26 → G27.
5. **Phase D:** G13 → G25 → G28 → G30.

Build G4 (sentinel) before any further expansion of `compile_memory.py`.

**After each feature:**
- Run `make lint`, the new and affected tests, the core suite on Python 3.10 with no extras, strict mypy on new modules, bandit, both OpenAPI `--check` scripts and `hub/tests/test_image_contents.py`. Run the Hub tests whenever `hub/` changed.
- Update `.dockerignore`, the Graphify module budget and the strict-mypy list.
- Commit.

**After each phase:**
- Run the full `make coverage` and confirm it stays at or above 80%.
- Append a phase summary to `docs/decisions/` covering what shipped, what was deferred and why, and any format marked "unverified".

**The roadmap is done when:**
- All 30 features are implemented with their surfaces, tests and docs, and every CI gate is green.
- The README feature list and project-status table are accurate, and the README describes UniMem as the referee of how agents and robots change themselves.
- `docs/guarantees.md` covers every new guarantee and its exact assumptions.
- `docs/physical-ai.md` walks a robot integrator through G22–G28 end to end.
- A new `docs/standards.md` lists each external standard touched and the version pinned:
  - Agent Skills
  - OpenTelemetry GenAI conventions
  - W3C VC 2.0 and SD-JWT
  - ISO/IEC TS 27560 and DPV
  - Cedar
  - VDA 5050 3.0
  - Open-RMF
  - OpenSCENARIO DSL 2.1
  - UL 4600
  - ISO 10218

  For each one, it marks which parts of the mapping are verified against fixtures and which are not.

## Do not

- Do not weaken, skip or delete existing tests to get green, and do not lower the coverage gate.
- Do not change the hash or canonical form of any existing record type, receipt, schedule, ledger, revision, log leaf or certificate.
- Do not enable by default any feature that withholds, evicts, quarantines, hibernates or blocks.
- Never randomize safety memories, and never run ask-versus-proceed outside configured low-risk contexts.
- Do not label a backend "zero-knowledge", "attested" or "compliant" unless that property actually holds in the code path that ran.
- Do not add network calls to tests, and do not add heavy dependencies to the core install.
- Do not bundle third-party datasets or benchmark data, and do not copy spec text.
- Do not claim state-of-the-art results, "first" or "only", or put unverified numbers in docs.
- Do not copy code from other projects without license compliance and a `THIRD_PARTY_NOTICES.md` entry.
- Do not rename the `commontrace` package or CLI in this work.
