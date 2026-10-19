# Codex prompt: implement the CommonTrace frontier roadmap (22 features)

Paste everything below the line into Codex, run from the root of the CommonTrace repository.

---

## Role and mission

You are a principal engineer and applied statistician working in the CommonTrace repository. CommonTrace is a governed, causal memory system for AI agents and physical AI. Its core differentiator is that every memory is measured. Randomized per-memory holdouts estimate each memory's causal effect on task outcomes, harmful memories are withdrawn, and a causal gate learns per context where each memory helps.

Your mission is to turn CommonTrace from a system that **measures** memory into one that **guarantees, governs and prices** memory, for both software agents and robots. Implement the 22 features specified below, in the phase order given. Implement each one completely: production code, CLI/MCP/HTTP surfaces where specified, tests that prove the statistical and security properties, and documentation.

Work autonomously. Make reasonable engineering decisions, record them in docstrings, and keep going. Do not stop to ask questions unless a requirement is impossible to meet without breaking an invariant listed below.

## Repository facts you must respect

### Stack

- **Local package:** Python 3.10+, package `commontrace/`. The core depends only on the standard library and PyYAML.
- **Optional dependencies:** live behind extras in `pyproject.toml` (`attention`, `fastembed`, `graph`, `vector-lance`, `vector-postgres`, `security`, `serve`, `llm`, `otel`, `dev`, …). They are imported lazily inside functions.
  - A new feature that needs numpy, cryptography, or another library must degrade gracefully, or raise a clear error naming the extra, when that library is absent.
  - Prefer stdlib math, as `commontrace/aipw.py` and `commontrace/causal_gate.py` do.
- **Hub:** `hub/` is the multi-tenant Starlette server on PostgreSQL. Schema changes are Alembic migrations in `hub/alembic/versions/`, and there must be exactly one head.
  - Hub tests need `HUB_TEST_DATABASE_URL`, a Postgres 16 database. In CI they run in the `test-hub` job.
- **Storage:** the canonical local storage is append-only JSONL and Markdown, written with `commontrace/_jsonl.py` and `frontmatter.locked`.
  - Facts live in SQLite (`fact_store.py`). Bitemporal semantics come from `valid_from`/`valid_until` plus record time, queried with `as_of` and `known_at`.

### Surfaces and how to add to them

- **CLI:** `commontrace/cli.py` has a static `_COMMANDS` tuple. Commands live in `commontrace/commands/*_cmd.py`, or in grouped modules (`governance.py`, `platform.py`, `knowledge.py`) mapped through `_COMMAND_GROUPS`. Prefer subcommands of existing groups (`allocate`, `experiment`, `fact`, `graph`, …) over new top-level commands.
- **MCP (local):** tools are defined in `commontrace/mcp_server.py` (`build_server`) and listed in `commontrace/mcp_tools.py` `LOCAL_TOOLS`. Keep both in sync. Tool docstrings are user-facing: write them for an agent reader.
- **HTTP gateway:** routes are declared in `commontrace/gateway.py` via `self._route(...)` with request-field docs. Regenerate the contract with `python scripts/export_openapi.py` and verify it with `--check`.
- **Hub HTTP and MCP:** regenerate with `python scripts/export_hub_openapi.py`.
- **Python API:** `commontrace/client.py` (`MemoryClient`) and `commontrace/measure.py` (`CausalMemory`).

### CI gates that must stay green

- `make lint` (ruff, line length 120, target py310).
- `make coverage` (`pytest tests e2e_tests` with an 80% coverage gate).
- The core suite on Python 3.10 with no extras.
- Hub tests against real Postgres.
- Bandit `-ll`.
- Strict mypy on the files listed in `.github/workflows/ci.yml` (security-scan job). Add every new pure-logic module you create to that strict list, and make it pass.
- `scripts/export_openapi.py --check` and `scripts/export_hub_openapi.py --check`.
- `hub/tests/test_image_contents.py`: every `commontrace` module the Hub imports, even lazily inside a function, must be allowlisted in `.dockerignore` as `!commontrace/<module>.py`.
- **Graphify architecture audit:** `.github/workflows/graphify.yml` caps `--max-modules`, the count of `.py` files under `commontrace/`. Raise it by exactly the number of modules you add, with a comment naming them.
- The perf gate (local retrieval latency). Do not add work to the hot retrieval path without a budget.

### Invariants you must never break

1. **Governance before ranking.** A fact or memory the caller cannot see must never be returned, used as an intermediate hop or bridge, or influence an answer. Every new retrieval path admits candidates through `search_recipes.admit` or the existing `_eligible_fact` rules.
2. **Randomization integrity.** Holdout assignment is `experiment.is_held_out` (a deterministic hash of salt, lesson and occasion), driven by `holdout_io.plan_assignments`/`assign_and_log`.
   - Every assignment row logs its own `rate` and, since the causal gate, an optional `context`.
   - Any new adaptive rate must be published as a hash-chained allocation schedule (`allocation_schedule.Schedule`) before the occasions it governs, so `integrity.check_assignment_drift` can re-verify it.
   - Never change how existing schedules, receipts, ledgers or revisions hash. Add fields so that they are omitted from hashed bodies when empty, as `Schedule.context_rates` does.
3. **Evidence of record.** `experiment.analyze` and `integrity.audit` remain the evidence of record for billing and release.
   - New estimators can add guarantees, but every output must state its assumptions and say whether it is a guarantee, an estimate or a heuristic.
   - Never relabel an associational number as causal.
4. **Safety memories are never randomized away.** Once F18 exists, a protected hazard class must be excluded from every holdout, allocation, compression, eviction and consolidation path.
5. **Bounded everything.** Every input from a user, tool, model or HTTP request is validated and size-bounded, with no unbounded loops, fan-outs or files.
   - Non-finite floats (NaN/∞) are refused wherever a number drives a decision.
   - Errors at network boundaries are clean 4xx responses, never 500s.
6. **Honesty in outputs.** No fabricated statistics. No claims in docs or CLI output that the code does not back.
   - Cite methods (paper and arXiv id) in module docstrings. Cite them, but do not vendor them.
   - If you adapt code from a permissively licensed project, add it to `THIRD_PARTY_NOTICES.md` with the license text.

### Existing modules to build on

Read these before writing anything; most features below are extensions, not greenfield.

| Area | Modules |
|---|---|
| **Experiment core** | `experiment.py` (analyze, BH, anytime CS); `aipw.py` (AIPW scores, asymptotic confidence sequences); `allocation.py` and `allocation_schedule.py` (eras, schedules, `mode=verdict\|thompson`); `causal_gate.py` (per-context Hajek IPW, empirical-Bayes shrinkage, Thompson rates, drift, off-policy SNIPS, source reputation); `holdout_io.py` (assignment log, outcomes, `context_label`); `integrity.py` (audits); `evidence.py` and `harm.py` (withdrawal); `heterogeneity.py`; `interactions.py` (2x2 factorial); `causal_policy.py` and `policy.py` (exploration and OPE); `survival.py`; `precision.py` (outcome-detector accuracy) |
| **Value and economics** | `value.py` (hash-chained ledger, HMAC signing); `pricing.py` (bills on the CI lower bound); `market_settlement.py`; `marketplace.py`; `overlap.py` (MinHash fleet overlap); `federation.py` (randomized response, pooled effects) |
| **Governance and trust** | `memory_authority.py` (write-time origin binding; derived facts inherit minimum trust); `provenance.py`; `lesson_admission.py` (review receipts); `commons_integrity.py` (HMAC); `privacy_erasure.py` (local cascade, excludes backups); `memory_guard.py`; `defense.py`; `injection_guard.py`; `receipts.py`; `prereg.py`; `release.py`; `proof.py` |
| **Memory lifecycle** | `hierarchical.py` (atomic facts); `fact_store.py`; `fact_conflicts.py`; `fact_consolidation.py`; `consolidate.py`; `commands/dream_cmd.py`; `ttl.py`; `decay.py`; `dosage.py` (context budget selection); `compression.py` (compression ladder); `recall.py`; `search_recipes.py`; `multihop.py`; `graph.py`; `entity_store.py` |
| **Physical AI** | `episodes.py` (multimodal robot/device episodes); `environments.py` (sim vs. real separation); `fleet.py` (pooled fleet experiments, protected prefixes); `failure_signals.py`; `failure_import.py`; `experience_skills.py`; `assurance.py` (replay sensitivity) |
| **Hub** | `hub/crud.py`; `hub/models.py`; `hub/server.py`; `hub/rest.py`; `hub/audit.py`; `hub/audit_export.py`; `hub/encryption.py`; `hub/kms.py`; `hub/retention.py`; `hub/manage.py` (legal holds); `hub/value_billing.py`; `hub/market.py` |

## Global engineering standard for every feature

- **New modules:**
  - Put pure, stdlib-only logic in new modules with a docstring that explains the method, its assumptions, its guarantee, and what it is not.
  - Keep I/O (store paths, JSONL, CLI printing) separate from math, so the math can be unit-tested and type-checked strictly.
- **Statistical tests:**
  - Every statistical guarantee gets a **seeded simulation test** that measures the property it claims: coverage, false discovery rate, harm rate at or below alpha, no-regression rate, or bias at or below a tolerance.
  - The tolerances must be honest, and the test must run in under 2 seconds.
  - Also add a test that fails if the key ingredient is removed (for example, propensity weights or a leave-one-out prior). Check it by mutating the code locally, and describe the check in the commit message.
- **Security tests:** every security feature gets adversarial tests (forged inputs, replay, laundering, cross-tenant access, oversized and non-finite inputs).
- **Surface tests:** every surface (CLI, MCP tool, gateway route, Hub route) gets an end-to-end test using the existing harnesses:
  - CLI: `tests/test_mcp_server.py` helpers `call`, `cli`, `_write_lesson`.
  - Gateway: `tests/test_gateway.py` `call`/`recall`.
  - Holdout and allocation: the clock-freezing fixture in `tests/test_allocation.py`.
- **Defaults:**
  - New behavior is opt-in by configuration unless it is a pure validity fix (F3's e-BH). Defaults must never withhold a memory that is not withheld today.
  - Schema additions to records are additive and backward compatible. Old stores must load unchanged, which a test must show.
- **Docs:** update the README (feature section and project-status table) for every shipped feature, without overclaiming. Add a `docs/<feature>.md` page for features with operator-facing workflows (F1, F2, F12, F13, F15, F16–F18, F21, F22).
- **Commits:** one conventional commit or more per feature (`feat(causal): …`, `feat(trust): …`, `feat(robotics): …`, `docs: …`). Each commit leaves the tree green.

## The features

Each feature lists its goal, where it lives, what to build, the surfaces, and the acceptance criteria. "Reuse" names the existing code to build on.

### Phase 0 (foundations): F11, F12, F3, F1, F15

#### F11. Non-malleable lineage graph (taint, quarantine, invalidation cascade)

- **Goal:** every derived object carries a provenance record that summarization or tool echo cannot launder. Trust flows downward as the minimum of the inputs, and harm and supersession cascade.
- **Build** (new `lineage.py`, extending `memory_authority.py` and `provenance.py`):
  - **Derivation attestation**, in-toto style: `{inputs: [content hashes and ids], operation, model_id, prompt_hash, created_at, actor}`, canonical JSON and SHA-256.
    - It is attached to every derived object: lessons, distilled skills, dream outputs, consolidated facts, summaries, triplets, cache blocks, compiled artifacts and marketplace listings.
  - **Integrity level** = the minimum over inputs. It can be raised only by:
    - a review receipt (`lesson_admission`), or
    - an **origin-diverse corroboration quorum**: at least k independent origins with distinct root origins, weighted by causal source reputation from `causal_gate`. Corroboration that shares a root origin counts once. This is the defense against FARMA-style self-amplification.
  - **Graph API:** `ancestors(id)`, `descendants(id)`, `taint(id, reason)`, and `invalidate(fact_id)`, which flags every descendant when a fact is superseded (hook `hierarchical.supersede_fact` and `fact_conflicts`).
  - **Harm taint:** when `evidence.withdrawn` lists a memory, descendants derived from runs that used it are down-ranked and listed for review.
- **Surfaces:**
  - CLI: `commontrace lineage show|taint|invalidate`.
  - MCP: `lineage_explain(id)`.
  - Retrieval results gain `integrity` and `lineage_digest` fields.
- **Acceptance:**
  - A summary of a low-trust memory stays low-trust (laundering test).
  - Three same-origin "corroborations" do not raise trust, and three independent origins do.
  - Superseding a fact flags its derived lesson and triplet.
  - Old records without attestations load as `integrity="unknown"` and are never treated as reviewed.
  - Strict mypy passes.

#### F12. Memory transparency log (tamper-evident history)

- **Goal:** make it impossible to silently drop a revocation, back-date an approval or show different histories to different tenants.
- **Build** (new `transparency_log.py`, Hub table and migration):
  - An append-only Merkle log (RFC 6962/9162-style leaf and node hashing) of signed statements for every write, approval, revocation, harm withdrawal, schedule publication, erasure and policy promotion.
  - It follows the IETF SCITT architecture (RFC 9943): signed statement in, receipt with inclusion proof out.
  - It publishes **signed tree heads** (Ed25519 via the `security` extra, with an HMAC fallback that is clearly labelled), **consistency proofs** between tree heads, and **inclusion proofs**.
  - Witnessing is optional: export tree heads for external witnesses to co-sign.
  - Locally, the log is a JSONL file of leaves plus a compact node cache. On the Hub, it is per-org Postgres tables.
- **Surfaces:**
  - CLI: `commontrace log head|prove|verify|consistency`.
  - Hub routes: `GET /v1/log/head`, `GET /v1/log/proof?leaf=`, `GET /v1/log/consistency?from=&to=`.
  - MCP: `log_verify`.
- **Acceptance:**
  - Inclusion and consistency proofs verify, and tampering with any leaf, reordering, or truncation is detected.
  - Two tenants cannot be shown forked heads without a consistency-proof failure.
  - Hashes are stable across Python versions (golden vectors in `protocol/conformance/`).

#### F3. E-process evidence ledger, e-BH, and peeking-safe regression adjustment

- **Goal:** make the evidence valid under arbitrary dependence and continuous monitoring, and portable across deployments.
- **Build** (new `eprocess.py`):
  - **Betting e-process** on the AIPW pseudo-outcomes from `aipw.aipw_scores`. Use a predictable plug-in bet (Waudby-Smith and Ramdas, arXiv 2010.09686) for the null "effect ≤ δ" and for "effect ≥ −δ".
  - **Multiplication across allocation eras.** Each era's e-value multiplies into the running product, which stays valid under optional continuation (Ramdas and Wang, arXiv 2410.23614).
  - **e-BH** (Wang and Ramdas, arXiv 2009.02824) for discoveries across memories. Replace BH in decision paths where shared tasks create dependence, and keep BH only for display with a note.
  - **Anytime-valid regression adjustment (CUPED-style)** on pre-task covariates (Lindon et al., arXiv 2210.08589). Supported covariates are pre-treatment context labels and a prior success rate.
  - **Ledger entries:** store each memory's e-value per era in the existing hash-chained ledger (`value.py`), so third parties can combine evidence by multiplication or averaging.
- **Surfaces:**
  - `commontrace experiment --evidence e-values`.
  - An `e_value` field in `experiment_status` and `causal_gate_report`.
- **Acceptance:**
  - A seeded simulation shows type-I error ≤ α under continuous peeking with dependent memories that share tasks; show that BH can exceed α in the same simulation.
  - Power is reported against BH.
  - Combining two deployments' e-values gives a valid combined test.

#### F1. Conformal harm-budget injection gate

- **Goal:** the operator declares "at most α of tasks harmed by injected memories", and the system guarantees it in finite samples.
- **Build** (new `conformal_gate.py`):
  - **Split conformal risk control** (Angelopoulos et al., arXiv 2208.02814) picks the injection threshold λ on a calibration slice built from holdout data. The loss is a harm indicator: an injected memory on an occasion that failed when its withheld counterpart succeeded, estimated by IPW from logged propensities.
  - **Learn-then-Test** (arXiv 2110.01052) handles several harm metrics at once.
  - **Stratification** by context label (role, tool, robot skill), so a memory that is safe on average cannot be unsafe in one stratum (conditional conformal, arXiv 2305.12616).
  - **Online conformal update** with decaying step sizes for drift (arXiv 2402.01139).
  - **Trajectory mode:** step-level losses and an anytime supermartingale alarm that halts an episode mid-run when the harm budget is breached (ToolChain-CRC style).
  - **Robot mode, KnowNo style:** if the conformal set of plausible actions given retrieved memories contains more than one action, return `ask_human=True`.
- **Surfaces:**
  - Config `retrieval --harm-budget 0.02 [--per-context]`.
  - The gate is applied in `dosage`/`recall` admission.
  - `commontrace allocate gate` reports the calibrated λ and the realized harm rate.
  - MCP `retrieve` returns `harm_budget` metadata.
- **Acceptance:**
  - Seeded simulations show realized harm ≤ α + O(1/n) marginally and per stratum.
  - The trajectory alarm fires within a bounded delay on an injected harmful stream and rarely on benign streams.
  - Safety-class memories (F18) are always admitted, regardless of the gate.

#### F15. Incident freeze, as-of replay, and regulator evidence packs

- **Goal:** one action pins memory state at an incident while the agent keeps operating, and produces signed evidence for regulators and courts.
- **Build** (new `incident.py`, extending `hub/audit_export.py` and `hub/manage.py` legal holds):
  - **`freeze(incident_id, valid_time)`** pins the bitemporal state. It blocks erasure and compaction of pinned records through a legal hold, and writes the freeze to the F12 log.
  - **`timeline(incident_id)`** shows which memories were retrieved and injected, which were withheld, who approved them, what was believed at the time versus what was corrected later (`known_at`), and the lineage (F11).
  - **Evidence packs**, each a signed bundle with a manifest:
    - **EU AI Act:** Art. 12 lifetime logs; Arts. 19/26(6) retention attestations; Art. 72 post-market monitoring summary from the causal effects; Art. 73 incident pack; Annex IV technical documentation sections generated from config, schedules and audits.
    - **ISO/IEC 42001** control-evidence mapping.
    - **NIST AI RMF and NIST AI 600-1** mapping.
    - **GDPR Art. 22 / CCPA ADMT** per-decision lineage record.
    - **EU Product Liability Directive Arts. 9–10** scoped disclosure export.
  - **Governance-platform feed:** JSON export of the evidence with control IDs, so AI-governance platforms can ingest it.
- **Surfaces:**
  - CLI: `commontrace incident freeze|timeline|pack --regime eu-ai-act|iso42001|nist|gdpr22|pld`.
  - Hub console page and routes.
  - MCP: `incident_timeline`.
- **Acceptance:**
  - Replaying "as of" a frozen incident returns exactly what the agent retrieved then, even after later corrections.
  - Packs verify offline: signatures, F12 inclusion proofs and file digests.
  - Erasure of a frozen record is refused with a clear reason, unless the operator explicitly releases the hold.
  - Mark clause mappings "verify against the Official Journal" where the wording is not yet final.

### Phase 1 (release safety, robot safety, lifecycle): F2, F18, F8, F9, F13, F14

#### F2. Provable promotion gate and Memory Change Control Plan

- **Goal:** no memory-policy change ships unless it provably beats the incumbent. This covers rankers, priors, consolidation rules, skill libraries, model or embedding upgrades, and the Thompson gate itself.
- **Build** (new `promotion.py`, reusing `causal_gate.policy_value`, `policy.py`, `prereg.py` and `release.py`):
  - **Doubly robust, anytime-valid off-policy confidence sequence** for V(candidate) − V(incumbent) on logged data with exact logged propensities (Waudby-Smith et al., arXiv 2210.10768; Cook, Mishler and Ramdas, arXiv 2311.18274). Promote only if the lower bound exceeds a declared margin.
  - **SPIBB fallback** (arXiv 1712.06924): in contexts where logged support is below a threshold, the candidate is forced to copy the incumbent.
  - **Memory Change Control Plan (MCCP):** a preregistered envelope of allowed changes, acceptance criteria, monitoring and rollback, modelled on FDA's predetermined change control plans, AI Act Art. 43(4) and SR 11-7.
    - Each promotion emits a signed **release certificate**: estimate, confidence sequence, margin, support coverage and design hash, logged in F12.
  - **Model-upgrade gate:** shadow-evaluate a new LLM or embedding model against the logged traffic before switching.
- **Surfaces:**
  - CLI: `commontrace promote evaluate|certify|rollback`.
  - Allocation `plan` refuses to publish a Thompson schedule that fails the gate when `--require-certificate` is set.
- **Acceptance:**
  - A simulation shows that a worse candidate is promoted at a rate ≤ α.
  - A better candidate with good overlap is promoted.
  - With thin support, behavior equals the incumbent.
  - Certificates verify offline.

#### F18. Safety-evidence ledger and holdout-exempt hazard memory

- **Goal:** make it acceptable to run holdouts on robots at all.
- **Build** (new `safety.py`, extending `fleet.py` protected prefixes and `episodes.py`):
  - **Protected `hazard` memory class** (zones, objects, conditions, proximity to humans):
    - never withheld, compressed, evicted or consolidated without review;
    - always retrieved when relevant;
    - requires a review receipt;
    - exportable as an offline edge cache.
  - **Enforcement in code:** every assignment, allocation, gate, dosage, compression, eviction and dream path refuses to treat a hazard memory as eligible for withholding. Add tests for each path.
  - **Near-miss and safety-performance-indicator time series** computed from episodes (UL 4600 leading indicators).
  - **Safety decision recorder** for machinery with self-evolving behaviour, per EU Machinery Regulation 2023/1230 Annex III 1.2.1 as summarized in secondary sources (verify the wording against the Official Journal):
    - retain safety-related decision data for 1 year;
    - keep a 5-year tracing log of interventions and safety-software versions;
    - ensure a correction path is always available.
    - The records are logged in F12.
- **Surfaces:**
  - CLI: `commontrace safety mark|spi|recorder export`.
  - MCP: `safety_context(site, robot)`.
- **Acceptance:**
  - Property tests over random configurations prove that hazard memories are never withheld by any path.
  - The recorder honours its retention periods and survives restarts.
  - SPI series compute correctly on synthetic episodes.

#### F8. Causal dreaming and learned utility retention

- **Goal:** consolidation and forgetting happen only where evidence says they help.
- **Build** (extend `commands/dream_cmd.py`, `consolidate.py` and `fact_consolidation.py`; new `retention.py`):
  - **Dream budget** goes to memories with high predicted future retrieval × positive effect.
  - **Itemized, provenance-carrying changes** in the style of ACE (arXiv 2510.04618), never wholesale rewrites.
  - **Quarantine** of negative-effect memories.
  - **Every rewrite is holdout-tested against its original** before it replaces it. Use the existing randomization, with the rewrite and the original as two arms.
  - **Learned retention model:** a small utility model (logistic or a 2-layer MLP in stdlib or numpy-optional) trained on randomized labels from the holdout, predicting marginal value. It drives eviction and tiering instead of plain TTL, with pooled or empirical-Bayes priors for low-traffic memories.
- **Acceptance:**
  - A simulation shows causal dreaming beats LLM-judgment-only consolidation on a synthetic task with planted harmful memories.
  - No hazard memory is ever touched.
  - Rewrites that lose their holdout are rolled back automatically.

#### F9. Value-per-token context packing and a cache-stable proven-memory prefix

- **Goal:** spend the context budget on causal value, and turn proven memories into cheap cached prefixes.
- **Build** (extend `dosage.py` and `compression.py`):
  - **Knapsack selection.** Value is the posterior uplift (from the causal gate for the context) × relevance. Weight is tokens, after optional compression. Near-duplicate distractors are penalized.
  - **Cache prefix.** Approved, high-value, low-churn memories go into a deterministic, versioned prefix block before the provider cache breakpoint; volatile memories go after it.
  - **Context bill:** report the estimated dollar and latency savings per block from provider cache pricing in config. Do not hard-code prices.
  - **KV reuse hook:** optional precomputed-KV integration points for self-hosted inference (LMCache/CacheBlend-style). Interface only, behind an extra.
- **Acceptance:**
  - Under a fixed budget, knapsack selection never yields lower expected value than relevance ranking in simulation.
  - Prefix bytes are identical across calls when memories are unchanged (golden test).
  - Hazard memories are always included.

#### F13. Certified erasure

- **Goal:** erasure that reaches derivatives, indexes and backups, with a certificate.
- **Build** (extend `privacy_erasure.py`, `hub/encryption.py` and `hub/kms.py`):
  - **Crypto-shredding.** Encrypt each subject's content under a per-subject data key, wrapped by the existing envelope keys. Destroying the key makes backup copies unreadable.
  - **Lineage walk (F11)** to embeddings, graph nodes, summaries, distilled lessons and listings: re-distill or withdraw each, then force vector-index compaction (HNSW rebuild) within a configured SLA.
  - **Membership-inference probe** against the live retriever after deletion (arXiv 2405.20446 style), with a pass/fail threshold.
  - **Signed deletion certificate** logged in F12. It keeps a **hash tombstone**, so audit chains still verify while the content is gone.
  - **Respect F15 legal holds**, with an explicit, logged conflict resolution.
- **Acceptance:**
  - After erasure, no retrieval, embedding-neighbor or graph query returns the subject's content or derivatives.
  - Backup snapshots taken before erasure cannot be decrypted.
  - The certificate verifies.
  - Records that were never erased are unaffected.

#### F14. Pre-action defenses and continuous memory red-teaming

- **Goal:** stop poisoning before it acts, rather than only after holdout evidence accrues.
- **Build** (new `memory_ifc.py` and `redteam.py`):
  - **Information-flow labels on retrieval results:** integrity and confidentiality labels plus the F11 lineage digest.
    - Low-integrity memories are returned as quoted data with a `procedure=False` flag, never as instructions to imitate. This blocks MemoryGraft-style imitation (arXiv 2512.16962) and follows the CaMeL and FIDES patterns (arXiv 2503.18813, 2505.23643).
  - **Origin-diverse certified aggregation** for queries flagged high-consequence: isolate memory groups by root origin, answer each group separately, then aggregate with a bound on the number of poisoned origins (RobustRAG, arXiv 2405.15556). Cost-gate it by a consequence flag.
  - **Robot sensor corroboration:** memories about physical-world state must be corroborated by fresh sensor evidence, a recent episode or a spatial fact (F17), before they drive motion.
  - **Per-tenant canary memories** with egress matching after base64/hex/Unicode canonicalization. A hit raises an alert and an F12 event.
  - **Scheduled red-teaming on a shadow store:** inject MINJA-, AgentPoison-, MemoryGraft-, sleeper- and FARMA-class memories (synthetic, built in-repo, no external payloads). Report write, retrieve and act rates and the **time-to-withdraw** of the causal harm loop.
- **Acceptance:**
  - Laundered and same-origin amplification attacks are blocked in tests.
  - Canaries trip on encoded exfiltration.
  - The red-team report is deterministic for a seed.
  - IFC labels appear in MCP and gateway responses, and existing clients still parse responses.

### Phase 2 (outcomes, robotics, validity at fleet scale, monetization): F5, F16, F4, F19, F17, F6

#### F5. Outcome engine: surrogates, debiased labels, sim-to-real calibration

- **Build** (new `outcomes.py`, extending `aipw.py`, `precision.py` and `environments.py`):
  - **Surrogate index.** Map early signals (step success, subgoal completion, grasp success) to the delayed outcome (Athey et al., arXiv 1603.09326), learned from CommonTrace's own corpus of past experiments (arXiv 2309.07893). Monitor surrogate-versus-final agreement and fall back when it degrades.
  - **Prediction-powered inference** for noisy labelers, such as LLM judges and robot success detectors (PPI and PPI++, arXiv 2301.09633 and 2311.01453): debias AIPW effect estimates with a small, actively chosen gold-labelled set (arXiv 2403.03208).
  - **Sim-to-real discrepancy ledger** as bitemporal facts per condition: where simulators or world models disagreed with reality. The gate learns a trust weight for each sim-derived signal, and sim and real data are never pooled silently.
- **Acceptance:**
  - Simulations show PPI-corrected effects are unbiased when the judge is biased and that naive effects are biased.
  - Surrogate decisions agree with final-outcome decisions above a reported rate on synthetic data.
  - Sim/real separation is enforced.

#### F16. Failure-mode and recovery registry

- **Build** (extend `failure_signals.py`, `failure_import.py` and `episodes.py`; new `recovery.py`):
  - **Failure signatures:** a calibrated failure score, a structured explanation (fields compatible with AHA/RoboFAC/FailSafe style outputs) and scene context.
  - **Candidate recoveries** are stored as memories, and **each recovery's lift is measured by randomized holdout**, scoped by site, robot and embodiment.
  - **MCP tool `recall_recovery(signature, context)`** returns recoveries ranked by effect, with intervals and `n`.
  - **Interventions per hour** is a first-class metric.
- **Acceptance:** an end-to-end simulated fleet test shows that recoveries with real lift rise to the top, harmful ones are withdrawn, and hazard memories are never withheld.

#### F4. Interference-aware fleet experiments

- **Build** (extend `fleet.py` and `integrity.py`; new `interference.py`):
  - **Write-path diversion.** The holdout arm cannot read memories derived from treated-arm runs, using F11 lineage.
  - **Cluster-level randomization** (agent cluster, robot cell or site) for shared memories, with Horvitz–Thompson estimation and design-based confidence sequences (arXiv 2210.08639).
  - **Clustered switchback designs** for fleet-wide policy changes, with carryover burn-in (arXiv 2312.15574, 2009.00148).
  - **New integrity check `symbiosis_bias`:** compare task-level and cluster-level estimates, and flag disagreement (arXiv 2309.07107).
- **Acceptance:** a simulation with write-back contamination shows the task-level estimate is biased, the clustered design is unbiased, and the audit flags the bias.

#### F19. Causal valuation certificate, warrantied memory, data-value export

- **Build** (extend `interactions.py`, `value.py`, `pricing.py` and `hub/value_billing.py`; new `valuation.py`):
  - **Banzhaf-style per-memory values** from independent Bernoulli inclusion: regress outcomes on inclusion indicators with pairwise terms for redundancy and complementarity.
    - Issue them as signed certificates: estimate, interval, n, design hash and F12 anchor.
    - In the docstring, state that the equivalence to Data Banzhaf's maximum-sample-reuse estimator (Wang and Jia, AISTATS 2023) is a design analogy, unless you prove it in a test or derivation.
  - **Warrantied memory tier.** Configurable lift thresholds; when verified lift falls below them, credits are computed automatically in the billing ledger.
  - **Insurer-grade export** with fields mapped to AI-insurance underwriting inputs (reliability series, incidents, harm budgets, change certificates).
  - **Robot data-value export:** episodes with intervention segments, success labels and memory effects, in LeRobot-compatible and RECAP-style layouts, ranked by measured value.
- **Acceptance:** simulated values recover planted per-memory effects and interactions within tolerance, and certificates and exports verify offline.

#### F17. Bitemporal spatial-object memory behind a planner-level MCP

- **Build** (new `spatial.py`, using `graph.py` and `fact_store.py`):
  - **Object-level scene-graph deltas**, in a schema compatible with Hydra, HOV-SG, ConceptGraphs and DynaMem outputs: object id, class, embedding ref, pose and frame, relations `located_in`/`on`/`near`/`seen_at`, confidence, robot id and timestamp. Store them as bitemporal facts. Dense geometry stays on the robot.
  - **Queries:** `where_is(object)`, `where_was(object, when=event)`, `believed_at(robot, time)` (record time) and `changes_since(t)`.
  - **Hub sharing** of site maps across robots and vendors under tenant permissions, through F14 labels.
  - **Planner MCP tools:** `spatial_where`, `spatial_history` and `spatial_believed`, each returning text plus keyframe and media references (`episodes` content addresses).
- **Acceptance:**
  - "Where was X when Y happened" and "what did robot 7 believe at time T" are correct across moves and corrections.
  - Multi-tenant isolation holds.
  - Updates are bounded in kilobytes per minute.

#### F6. Validated uplift targeting and embodiment-scoped transfer

- **Build** (extend `heterogeneity.py` and `causal_gate.py`; new `uplift.py`):
  - A **DR-learner** (arXiv 2004.14497) on the AIPW pseudo-outcomes, with features: context label, tool, model version, and for robots embodiment, gripper type, site and object class.
  - **RATE/AUTOC validation** (arXiv 2111.07966) on held-out data. No targeting rule goes live unless RATE is significantly positive; otherwise fall back to the empirical-Bayes gate.
  - **Fleet transfer:** the Hub promotes a memory fleet-wide only into embodiment and site classes where its measured effect is positive.
  - **Exploration slice:** always keep a slice of untargeted randomized traffic.
- **Acceptance:** a simulation with embodiment-specific effects (helps a parallel-jaw gripper, hurts a suction gripper) shows correct targeting, and RATE rejects a noise-only targeting rule.

### Phase 3 (diagnosis, market, long bets, platform reach): F7, F20, F10, F21, F22

#### F7. Per-episode counterfactual attribution

- **Build** (extend `assurance.py`; new `attribution.py`):
  - **Ablation surrogate:** for sampled episodes, run randomized masks of the injected memories (ContextCite-style, about 32 ablations, arXiv 2409.00729) and fit a sparse linear surrogate to attribute the outcome to memories and steps.
  - **Restore-counterfactual audit** of eviction decisions.
  - **Counterfactual memory regret** as a poisoning-severity score.
  - **Feedback loop:** results feed the gate and the F12 log.
  - Sampling and cost caps are mandatory, and digital-twin replay can substitute for robot re-runs.
- **Acceptance:** attribution recovers a planted culprit memory in synthetic episodes, and cost stays within the configured budget.

#### F20. Sealed-trial memory clean room with Sybil-resistant causal royalties

- **Build** (extend `marketplace.py`, `market_settlement.py`, `overlap.py` and `federation.py`; new `cleanroom.py`):
  - **Sealed trial.** A buyer runs a signed listing under holdout in its own environment, and the lesson content stays sealed (seller-side sandbox interface, with an enclave-attestation hook as an interface).
  - **Escrow** settles only on ledger-attested lift (F3 e-values plus the F19 certificate), and the price comes from MinHash fleet overlap.
  - **Royalties** are proportional to interventional value measured in buyers' environments. Near-duplicate lessons collapse into one MinHash cluster that shares one reward, and stake can be slashed on significant harm.
  - **Cross-tenant effect sizes** are released under differential privacy with per-querier budget accounting (arXiv 2511.07637). Reuse `federation.randomized_response`, and add a Gaussian/Laplace mechanism with accountant.
  - **Payments:** Stripe Connect integration behind configuration, with no live calls in tests.
- **Acceptance:**
  - Sybil duplication does not increase total payout (simulation).
  - The DP accountant refuses queries over budget.
  - Escrow never releases without a verified certificate.

#### F10. Governed compilation into parametric memory (long bet)

- **Build** (new `compile_memory.py`, behind a new extra, for open-weight models only):
  - **Compile targets:** an approved, stable, causally positive memory bundle compiles into a LoRA adapter, a Cartridge-style trained KV prefix (arXiv 2506.06266), or knowledge edits for atomic facts (AlphaEdit-style).
  - **Governance:** each artifact has a manifest of source memory ids and digests (F11). The artifact is holdout-tested against retrieval of the same memories. Retracting any source triggers rollback or recompilation.
  - **Default build:** ship the governance, manifest and holdout plumbing with a pluggable trainer interface and a tiny CPU-only reference trainer for tests. Real training is a plugin.
- **Acceptance:** retracting a source marks the artifact stale and removes it from serving, and the compiled artifact cannot be promoted without an F2 certificate.

#### F21 (beyond the report). Evidence layer over third-party memory stores

- **Goal:** position CommonTrace as the measurement and governance layer **over** hyperscaler and vendor memory, not only as a replacement.
- **Build** (extend `memory_adapters.py` and `measure.CausalMemory`):
  - **Read adapters:** Mem0, Zep/Graphiti, Letta, AWS Bedrock AgentCore Memory, Google Vertex AI Memory Bank, Microsoft Foundry memory and the Anthropic memory tool, each behind an extra, with HTTP mocked in tests.
  - **Holdouts on foreign memories:** the adapters wrap the vendor's retrieval so each returned item gets holdouts, causal-gate rates, F1 harm budgets, F11 lineage and F12 logging. Stable item ids come from vendor ids, or content hashes when the vendor has none.
  - **Comparative report:** "which store's memories helped".
- **Acceptance:**
  - Adapter conformance tests pass against recorded fixtures.
  - The causal loop runs end to end over a mocked vendor store.
  - No vendor credentials appear in logs.

#### F22 (beyond the report). Physical-AI bridge: ROS 2, MCAP and LeRobot

- **Goal:** make robot integration a one-hour job.
- **Build** (new `robotics/` subpackage; count each module against the Graphify budget):
  - **ROS 2 node, behind an extra.** It subscribes to configurable topics (task start/end, success detector, failure signals, scene-graph deltas), calls the gateway (`/v1/recall` with `context` = embodiment/site/skill, and `/v1/outcome`), and publishes retrieved memories to the planner.
  - **MCAP and LeRobot ingest:** convert logs into `episodes` (content-addressed media, sensor summaries) without decoding raw media in the Hub.
  - **Edge cache:** an offline-first cache of hazard memories (F18) and proven memories, with signed updates verified against F12 tree heads, and a store-and-forward outcome queue for intermittent connectivity.
- **Acceptance:**
  - A simulated robot loop (no ROS installation required in CI; mock the rclpy interface) completes recall → act → outcome → holdout analysis.
  - The edge cache rejects unsigned or forked updates.
  - MCAP and LeRobot fixtures ingest correctly.

## Order of work and definition of done

1. **Phase 0:** F11 → F12 → F3 → F1 → F15. Ship the F4 `symbiosis_bias` audit early as a check only, because current fleet estimates may already be affected.
2. **Phase 1:** F2 → F18 → F8 → F9 → F13 → F14.
3. **Phase 2:** F5 → F16 → F4 (full designs) → F19 → F17 → F6.
4. **Phase 3:** F7 → F20 → F10 → F21 → F22.

After each feature:

- Run `make lint`, the new and affected tests, the core suite on Python 3.10, strict mypy on new modules, bandit, both OpenAPI `--check` scripts and `hub/tests/test_image_contents.py`. Run the Hub tests whenever `hub/` changed.
- Update `.dockerignore`, the Graphify module budget and the strict-mypy list as needed.
- Commit.

After each phase, run the full `make coverage` and confirm it stays at or above 80%.

The roadmap is done when:

- All 22 features are implemented with their surfaces, tests and docs, and every CI gate is green.
- The README's feature list and project-status table are accurate.
- A new `docs/guarantees.md` explains, in plain language, each guarantee CommonTrace makes (harm budget, promotion certificate, e-value evidence, interference-aware estimates, certified erasure) and the exact assumptions under which it holds.
- A new `docs/physical-ai.md` walks a robot integrator through F16–F18, F22 and F5 end to end.

## Do not

- Do not weaken, skip or delete existing tests to get green, and do not lower the coverage gate.
- Do not change the hash or canonical form of any existing record type, receipt, schedule, ledger or revision.
- Do not make any feature that withholds memories enabled by default, and never randomize safety memories.
- Do not add network calls to tests, and do not add heavy dependencies to the core install.
- Do not claim state-of-the-art benchmark results or put unverified numbers in docs.
- Do not copy code from other projects without license compliance and a `THIRD_PARTY_NOTICES.md` entry.
