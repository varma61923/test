# Handover: finish the proof-and-governance roadmap and take CommonTrace from zero revenue to category leader

Paste everything below into a coding agent that has `varma61923/commontrace-v2` checked out. This continues the original master prompt ("make CommonTrace the proof-and-governance layer for agents that learn"). §2 lists every item of that prompt with its true status as of 1 Oct 2026 and the work and acceptance criteria still owed.

**Out of scope for now (owner decision):** SDK work. That means:
- TypeScript SDK parity;
- the Go SDK;
- new framework middleware packages: Claude Agent SDK, Vercel AI SDK, CrewAI, Google ADK, Mastra.

The three framework integrations already built (LangGraph, Pydantic AI, OpenAI Agents SDK) stay and must keep passing.

---

## 0. Role and mission

You are the founding engineering and product lead of CommonTrace. The company has **no revenue**. The goal is a multi-billion-dollar company, so prioritize what produces paying customers first, then what makes the product hard to displace.

**Product:**
- Every company's agents learn: they write memory, rules and lessons.
- CommonTrace is the **neutral layer that proves which learned memory changes business outcomes, withdraws what hurts, keeps an audit trail a risk officer accepts, and bills on the proven improvement.**
- It works across any agent platform, any memory store and **any business function: support, sales, HR/recruiting, coding**, and others.
- Memory vendors are integration partners, not rivals. CommonTrace's own store is the reference implementation and the self-host/air-gap option.

**Why it can be large** (each claim falsifiable; test the cheapest first):
1. **Agent fleets will accumulate learned memory across every function.** *Breaks if:* fleets stay stateless.
2. **Platforms and memory vendors grade their own homework**, so buyers need neutral, randomized proof. *Breaks if:* vendor self-reports satisfy buyers.
3. **CommonTrace already has the hard part:** holdouts, anytime-valid intervals, integrity checks that refuse compromised runs, harm withdrawal, and a signed value ledger. *Breaks if:* customers lack enough outcome-labelled volume. The power planner must say so up front.
4. **Revenue scales with agents under management × proven value.** *Breaks if:* buyers insist on per-seat pricing.
5. **Moats:**
   - outcome connectors into systems of record;
   - a methodology that refuses to lie;
   - a privacy-preserving, cross-customer dataset of which learned memory helps, by function;
   - compliance artifacts.

---

## 1. Ground truth

PR #84 (`claude/commontrace-proof-governance-05d3be`) is open against `main`, and **CI is fully green on Linux, Python 3.10–3.12**. Merge it first if the owner hasn't. Re-verify anything below in the code before relying on it.

**Core (`commontrace/`, PyYAML-only install, 30 CLI commands):**
- Loop: capture → distill → approve → query/inject → experiment (holdout) → reliability/consolidate → release.
- MCP server: `commontrace serve`.
- Measuring memory held elsewhere:
  - `measure.CausalMemory` (any retrieval callable);
  - `memory_sources.FileMemorySource` (`commontrace source`, `##` sections);
  - `memory_adapters` (Mem0, Letta archival passages, Zep, Claude memory stores, AgentCore), each written against its SDK's published source.
- `llm.py` (strict-JSON drafting; Anthropic or any OpenAI-compatible http(s) endpoint; no SDK dependency), used by:
  - `distill --draft`;
  - `lesson suggest-revision --draft`;
  - `lesson suggest-rewrite`;
  - `consolidate --draft`;
  - `lesson auto-approve` (policy `auto_approve_drafts`, requires a started holdout, locks it on).
- `integrations/` (LangGraph, Pydantic AI, OpenAI Agents SDK) and `outcome_detect.py`.
- `commontrace signals` (failure clusters → LangSmith/Braintrust datasets).
- `commontrace kb install` (four engineering packs selected from `commons/seed/substrate-v1.jsonl`).
- `commons/eval/causal_harness.py` (30 seeds: real effect 100%, null not claimed 90%, tampered log refused 100%).
- `commontrace/fixtures/fields/` holds labelled retrieval fixtures for clinical, coding, finance, HR, legal, marketing, robotics and sales.

**Hub (`hub/`, Starlette + Postgres):**
- **Tenancy and identity:** forced row-level security, scoped keys, RBAC, OIDC SSO, SCIM.
- **Records:** audit log plus `hub.manage export-audit jsonl|cef`, retention and legal hold.
- **Integrations:** webhooks, alerts, Stripe *subscription* billing, REST `/api/v1`, opt-in `POST /v1/traces` (OTLP/JSON).
- **Consoles:** `/app` (including a shareable Proof page) and `/admin`.
- **Deployment and ops:** docker-compose on Postgres 18, k8s manifests, and a measured restore drill (`hub/DEPLOYMENT.md` §9b).
- **The Hub has no `lessons` table by design** (`hub/README.md`, "Why there's no lessons table"). `sync --push` uploads active lessons as traces.

**Evidence of value:** one pilot (Loops) self-reported −53% time-to-resolve and −29% churn. It is correlational, a single customer, and never to be presented as proof.

---

## 2. Every item of the original prompt: status and what is still owed

Legend: **Done** · **Partial** (says exactly what is missing) · **Not started** · **Excluded** (SDK scope).

### Phase 0: land and baseline

| Item | Status | Still owed |
|---|---|---|
| Start from latest `main`, CI green | Done (#84 green) | Merge #84. |
| `AUDIT_2026Q4.md` full audit + fix every High/Medium with a regression test | **Not started** | Whole audit, see Phase A below. Known starting points:<br>• no injection-time sanitizer or provenance banner on lesson text handed to agents (only admission-time `memory_guard`);<br>• never fuzzed: org/resource IDs across every Hub route and MCP tool, key-rotation races, webhook replay;<br>• no container scan.<br>Fixed this session: `%-d` strftime and secret-path repr (Hub), LLM URL scheme (B310), Postgres 18 mount, missing `adapters.py` in the image. |
| `metrics/baseline.json` | **Not started** | Time to first trace, first active lesson, first causal verdict (simulated fleet); query p50/p95; Hub p95; install success across the 6 targets. |

### P0-1: Causal layer over any memory

| Item | Status | Still owed |
|---|---|---|
| `MemorySource` interface, stable content hash | Done (`MeasuredMemory` + adapters; text hash = revision) | — |
| Claude Managed Agents memory stores | Done (snapshot → render; delete with `expected_content_sha256`) | Live-account smoke test (owner provides a key). |
| AWS AgentCore (episodic + reflections) | Partial | Verify reflection-strategy records come back through `retrieve_memories` namespaces; live smoke test. |
| Mem0 OSS + Platform | Done (shape verified from source) | Live smoke test. |
| Zep/Graphiti facts | Done (edges; nodes/episodes measured but not deletable) | Live smoke test. |
| **Letta memory blocks** | **Partial** | Only archival passages are covered. **Core memory blocks** sit in context like files. Measure them by rendering per occasion, or by detach/reattach per session if the API makes that safe for concurrent sessions. Verify against `letta-client` source. |
| CLAUDE.md/AGENTS.md/.cursor/rules | Done (`FileMemorySource`) | — |
| **Devin Knowledge exports** | **Not started** | Get a real export format (owner provides a sample), then a parser into sections. |
| Withholding at injection | Done (filtering for search-based stores; render for whole-file/store) | — |
| `CausalMemory` as the public API | Done | — |
| Harm withdrawal → source write-back or blocklist | **Partial** | Manual `withdraw()` only. **Automatic withdrawal is missing:** adapters and `CausalMemory` never consult the HURTS verdict / `--on-harm withdraw` policy that local lessons use. Wire it, and test that a seeded harmful memory is withdrawn within the planned sample size. |
| **Accept:** per adapter, seeded ±effect → correct verdict, CI coverage ≥ 90% over **200 seeds** | **Not met** | Tests use one fixed seed. Add a 200-seed coverage run per adapter (a slow, opt-in harness scenario, not unit tests). |
| **Accept:** zero change in control-arm behaviour | Not tested explicitly | Add a test. |
| **Accept:** one quick start per adapter | **Not started** | Keep each to a short code block (owner wants minimal docs). |

### P0-2: Assisted extraction

| Item | Status | Still owed |
|---|---|---|
| BYO model | Partial | Anthropic + OpenAI-compatible work with zero dependencies. **Bedrock and Vertex are not implemented.** Add them behind an optional `[llm]` extra using the official SDKs (never hand-rolled SigV4/JWT), degrading with a labelled note when missing. |
| Default to the latest Claude model | Done (`claude-sonnet-5`, overridable) | Re-check model IDs at the time of work. |
| `distill --draft` | Partial | Clusters **all** traces. The spec says **failed occasions** using the existing taxonomy. Add a failure filter (reuse `failure_signals`). |
| `suggest-revision --draft`, `suggest-rewrite` | Done | — |
| Consolidation "dream" job | Partial | `consolidate --draft` exists; **no scheduler**. Add an opt-in scheduled run (Hub scheduler pattern in `hub/scheduler.py`, or a documented cron/CI recipe for the local tier). |
| Provenance (model, prompt hash, sources) | Done (`llm_draft` frontmatter) | — |
| Every draft enters the approval gate | Done | — |
| Auto-approve with mandatory holdout | Done | — |
| **Accept:** ≥ 80% of drafts pass scaffolding/safety/redundancy unedited on a curated fixture | **Not measured** | Build the fixture and a draft-quality harness. Needs a real model key (owner). Report the rate. |
| **Accept:** blinded human rating ≥ hand-written, 50-lesson sample | **Not started** | Build the blinding tool; humans rate. |
| **Accept:** tokens **and cost** per draft | Partial | Tokens recorded; add cost from a price table the owner confirms. |
| Core install still PyYAML-only | Done | Keep it. |

### P0-3: Live capture everywhere

| Item | Status | Still owed |
|---|---|---|
| OTLP/HTTP ingest | Partial | `POST /v1/traces`, OTLP/**JSON** only (protobuf → 415). Decide on protobuf via `opentelemetry-proto` behind an extra. **Spans are not linked to holdout occasions.** Map a span attribute (e.g. `commontrace.occasion_id` / OpenInference session id) so outcomes join. Verify OpenInference attribute coverage against its spec. |
| Middleware: LangGraph, Pydantic AI, OpenAI Agents SDK | Done | Keep green. |
| Middleware: Claude Agent SDK, Vercel AI SDK, CrewAI, Google ADK, Mastra | **Excluded** (SDK scope) | — |
| Each integration: auto record_occasion, inject with holdout, capture outcomes | Partial | Recall with holdout works. **Outcome capture is a manual helper**: wire `outcome_detect` signals automatically where the framework exposes them. |
| Automatic outcome detection | Partial | Detectors exist. **Missing:** per-fleet configuration, and connectors to systems of record (Phase B). |
| **Accept:** runnable example + CI smoke test per integration | Partial | Tests exist (offline models); add runnable examples. |
| **Accept:** first causal assignment < 10 min | **Not measured** | Measure it on a clean machine. |
| **Accept:** outcome detection precision ≥ 0.9 on a labelled sample | **Not started** | Needs a labelled sample, ideally a design partner's. |

### P0-4: Zero-to-proof onboarding and hosted tier

| Item | Status | Still owed |
|---|---|---|
| Managed deployment: Helm, Terraform (AWS/GCP), multi-region Postgres, backups, PITR, status page, SLO alerts | **Not started** | All of it. Install `helm` and `terraform` to validate (`helm lint`/`template`, `terraform validate`) in CI. Real deployment needs the owner's cloud account. |
| Signup → key → install → first lesson | Partial | Pieces exist (`/signup`, keys, `install`); not timed end to end, not hosted. |
| "Proof in a week" wizard | **Not started** | Plan (MDE) → start → live progress → verdict → shareable proof page (reuse the existing `/app` Proof page and value ledger). |
| Distribution: Claude plugin/Marketplace, MCP registries, installer skill | **Not started** | Plugin manifest, registry entries, a skill whose quick start is "ask your agent to install CommonTrace". Publishing needs owner approval. |
| **Accept:** cold user → injected lesson < 5 min | Not measured | — |
| **Accept:** wizard → valid verdict on a simulated fleet | Not started | — |
| **Accept:** load test at 10× current Hub throughput passes the SLO | Not started | `hub/bench_concurrency.py` / `bench_scaling.py` exist as a starting point. |

### P1-5: Lesson workbench in the console

| Item | Status | Still owed |
|---|---|---|
| Review queue (diff, cited evidence, redundancy neighbours, approve/edit/reject, bulk) + per-lesson page (revision history with point-in-time text, causal status, harm alerts, withdrawal log) | **Not started** | **Design decision first:** the Hub has no lessons table.<br>Options:<br>(a) a local console served by the CLI over the local store (no new Hub schema);<br>(b) a Hub lessons model with sync.<br>Recommend (a) first; it reuses `lesson_io` history and approval directly. Everything role-gated, audited, CSP and cross-origin protected. |
| **Accept:** Playwright at 390px/1280px, both themes; axe-core zero serious issues | Not started | — |

### P1-6: Failure signals → lessons → fixes

| Item | Status | Still owed |
|---|---|---|
| Named signals (trend, size, affected agents) | Done | — |
| Draft a lesson per signal | Partial | Wire `signals` → `distill --draft` scoped to one signal. |
| Dispatch a coding agent through MCP, verify the fix with a holdout | **Not started** | — |
| Export to LangSmith/Braintrust | Done | — |
| **Accept:** adjusted Rand index ≥ 0.8 vs labelled failure modes | **Not measured** | Build a labelled seeded dataset and report the ARI. The clustering may need to improve to reach it. |

### P1-7: Outcome-based pricing, end to end

| Item | Status | Still owed |
|---|---|---|
| `plans.py` aligned with the value ledger (free by volume; paid on proven `occasions_improved` × customer value per occasion, with a floor) | **Not started** | Owner sets every price. |
| Stripe metered billing from the ledger; invoice line linking to the proof page; never bill a compromised effect | **Not started** | Stripe test mode only. |
| **Accept:** year-long synthetic simulation never bills compromised/unproven; contract test per plan transition | Not started | — |

### P1-8: Enterprise readiness

| Item | Status | Still owed |
|---|---|---|
| BYOK/CMK envelope encryption for trace and lesson content | **Not started** | Extend `hub/encryption.py`; key-rotation tests. |
| Data residency EU/US | Partial | `/disclosure` reports a region; nothing enforces it. |
| VPC/Helm install; air-gapped install with local models | **Not started** | Air-gap can use the OpenAI-compatible path to a local model server. |
| SIEM audit export | Done | — |
| Per-org retention and legal hold | Done (pre-existing) | — |
| DPA/subprocessor pipeline | Partial | `SUBPROCESSORS.md` exists; no pipeline. A DPA needs a legal entity (owner). |
| SOC 2 Type II program (evidence automation from `SOC2_READINESS.md`) | **Not started** | The audit firm is the owner's call. |
| **Accept:** cross-tenant fuzz tests, key rotation tested, restore drill timed | Partial | Rotation tests and a timed restore drill exist; **systematic cross-tenant fuzzing does not**. |

### P1-9: Publish the causal-detection harness

| Item | Status | Still owed |
|---|---|---|
| Reproducible harness, seeds, one command | Partial | Real effect, null and tampered log covered. **Attrition and mid-run edits are missing**, and both are named in the spec. Add them, plus per-adapter and per-function scenarios. Publishing is the owner's call. No vendor comparisons. |

### P2-10: Curated lesson packs

| Item | Status | Still owed |
|---|---|---|
| Reviewed, versioned packs; `kb install` | Partial | Four engineering packs, versioned by content hash. **Missing:**<br>• packs for support, sales and HR, source-cited only (a test pins `source` on every record);<br>• the spec's examples (Stripe webhooks, React 19, LLM tool-call quirks), with real sources. |
| Measured effect across opted-in orgs; privacy layer (PSI or differential privacy) first | **Not started** | Needs the owner's explicit decision before any cross-org data flow. |

### P2-11: Performance and scale

| Item | Status | Still owed |
|---|---|---|
| Warm daemon; ONNX int8 embedder/reranker; pgvector HNSW above 50k lessons; Hub pgvector + BM25 hybrid; background index refresh | **Not started** | `commontrace serve` is a warm process already; measure it first. `pgvector/pgvector` images are available for local testing. |
| **Accept:** local p95 < 300ms warm at 10k lessons; Hub p95 < 150ms at 1M traces/org; rankings identical or differences explained | Not measured | — |

### P2-12: Developer experience

| Item | Status | Still owed |
|---|---|---|
| Docs site (quick start per framework and per memory source); cookbook; troubleshooting generated from `doctor` checks | **Not started** | Keep docs lean; generate what you can. |
| TypeScript SDK parity, Go SDK | **Excluded** | — |
| Versioned protocol + compatibility test suite third parties can run ("CommonTrace-compatible") | **Not started** | Build it from `protocol/schemas`: a CLI command or package that runs conformance checks against a store or Hub. |

### Section 6 metrics

Nothing is instrumented yet. Phase A creates the baseline, and every later phase reports against it.

---

## 3. Make it work for every function (new, beyond the original prompt)

The master prompt is function-agnostic in principle; the code is not yet. Close these:

1. **Remove coding defaults.**
   - `init`, `hub/rest.py`, `hub/otlp.py` and `hub/manage.py` seeding default `agent_type` to `"code"`.
   - Replace with an explicit function, or `"general"`.
   - Add `commontrace init --function support|sales|hr|coding|custom`.
2. **Function kits.** Each is the unit that makes a new function work: outcome model, occasion ID, defaults, starter pack, example agent, demo data.

   | Function | Outcome model | Occasion ID |
   |---|---|---|
   | Support | resolved without reopen within 7 days, CSAT ≥ 4/5, no human takeover | ticket ID |
   | Sales | reply received, meeting booked, or stage advanced within N days | deal or contact ID |
   | HR / recruiting | candidate responded, interview scheduled, offer accepted; HR helpdesk question resolved without escalation | candidate or requisition ID |
   | Coding | CI passes, PR merged, not reverted within 14 days | PR number |

   Each kit also carries:
   - default dosage and holdout settings;
   - a power forecast ("at your volume, a verdict in ~N days");
   - a starter pack (source-cited);
   - an example agent;
   - a demo dataset, so the Proof Report can be shown before the customer has data.
3. **Outcome connectors: the revenue-critical engineering.** Without them there is no proof outside coding.
   - **Systems, in order:** Zendesk, Intercom, HubSpot, Salesforce, Greenhouse, Ashby, GitHub.
   - **What each one does:** Hub-side webhook receivers plus pollers that turn system-of-record events into `record_occasion_outcome`.
   - **What each one needs:**
     - built from the vendor's published API docs or SDK source;
     - signed-webhook verification and replay protection;
     - per-tenant encrypted credentials;
     - SSRF-safe calls;
     - idempotency and a dry-run mode;
     - tests on the vendor's documented payloads.
4. **Per-function harness scenarios** in `causal_harness.py`, using the fixtures in `commontrace/fixtures/fields/`.

---

## 4. Revenue plan

- **Wedge: a paid "Agent Learning Proof."**
  1. Connect the customer's agent platform, their system of record and their existing memory.
  2. Run the power plan.
  3. Run a 14-day randomized holdout on their own memory.
  4. Deliver a signed Proof Report: per-memory verdicts, lift with intervals, harmful memories withdrawn, value at their own value per occasion, the recomputable hash-chained ledger, and integrity findings.
  5. Convert to a continuous subscription billed on proven value.
- **Beachhead:** customer support first (highest volume, crisp outcomes), coding second. Sales and HR kits ship in the product from day one.
- **Pricing hypotheses (owner decides; test with design partners):**
  - free self-hosted or volume-capped hosted tier;
  - a fixed-fee Proof Pilot per function;
  - a platform-fee floor plus a share of proven value.

  Never bill a compromised, underpowered or expired effect.
- **Distribution:**
  - MCP registries;
  - Claude and other agent marketplaces;
  - a GitHub Action that gates memory regressions with `experiment --strict`;
  - memory vendors as "proven by CommonTrace" partners;
  - later, an annual DP-aggregated "Agent Learning Index".
- **Later moats** (each needs owner decisions):
  - a lesson-pack marketplace where authors earn only when packs prove effect;
  - EU AI Act post-market-monitoring exports;
  - memory incident response that pages Slack/PagerDuty with the withdrawal already applied.

---

## 5. Execution order (finish each phase's acceptance before the next)

- **Phase A, baseline (week 0–2):**
  - merge #84;
  - remove coding defaults;
  - `metrics/baseline.json`;
  - `AUDIT_2026Q4.md`, fixing every High and Medium, with these first: injection-time sanitizer and provenance banner; cross-tenant fuzz of every Hub route and MCP tool; webhook replay; key-rotation races; container scan.
- **Phase B, revenue unlock (weeks 1–6):**
  - function kits and outcome connectors (§3);
  - OTLP span → occasion mapping;
  - automatic harm withdrawal for adapters;
  - Letta core blocks;
  - `distill --draft` on failed occasions;
  - per-function and per-adapter harness scenarios, including the 200-seed coverage runs, attrition and mid-run edits.
- **Phase C, hosted and proof flow (weeks 4–10):**
  - Helm, Terraform, backups/PITR, status page, SLOs;
  - the proof wizard and a signed, shareable report;
  - timed cold-start flows;
  - the 10× load test;
  - distribution assets (plugin, registry, skill), published only with owner approval.
- **Phase D, quality evidence (weeks 6–12):**
  - draft-quality harness (≥ 80% pass gates) and blinded rating tool;
  - cost per draft;
  - Bedrock/Vertex behind `[llm]`;
  - the scheduled dream job;
  - outcome-detection precision ≥ 0.9;
  - signal ARI ≥ 0.8;
  - the workbench (local console first), with Playwright and axe.
- **Phase E, billing (weeks 8–14):** P1-7 in full, prices from the owner.
- **Phase F, enterprise and scale (months 3–9):**
  - BYOK;
  - residency enforcement;
  - VPC and air-gapped installs;
  - DPA pipeline;
  - SOC 2 evidence program;
  - P2-11 performance;
  - the protocol conformance suite;
  - docs site, cookbook and doctor-generated troubleshooting;
  - the privacy layer, then cross-org pack effects (owner decision).

---

## 6. Operating rules (non-negotiable)

1. **Evidence before claims.** No feature, README line or number without a test or measurement. A compromised experiment produces no figure. A claim written before its evidence is wrong even if it later comes true.
2. **Minimal docs.** The owner removed long-form docs:
   - README additions are short command lists;
   - no new prose in STRATEGY.md;
   - reasoning goes in concise docstrings and commit messages;
   - quick starts are code blocks, not essays.
3. **No competitor names, comparisons or unverifiable market numbers** in the repo, README or PRs.
4. **Verify against reality.** Read a third-party SDK's published source, or run it, before integrating. Every integration test drives the real framework loop offline. If it can't be verified, don't ship it.
5. **Governance is the product.** Drafts never activate outside the approval gate. Withdrawal blocklists by default; deleting at the source is opt-in.
6. **Security:**
   - new routes are authenticated, scope-checked, rate-limited, body-limited, under RLS, with cross-tenant tests;
   - outbound calls are SSRF-checked, and only http(s) URLs are accepted;
   - secrets are never logged;
   - any new `commontrace` module the Hub imports goes in `.dockerignore`'s allowlist (`test_image_contents.py`).
7. **Light core.** The core install stays PyYAML-only. Heavy dependencies go behind extras or lazy imports (`test_cli.py::test_a_command_imports_only_what_it_uses`).
8. **Git:**
   - feature branches, one coherent change per commit;
   - never force-push;
   - `ruff check .` plus the relevant tests before pushing;
   - Linux CI is the authority; never merge red;
   - use the attribution the session specifies.
9. **Ask the owner before:** prices, cloud resources or any spend, outreach or email, publishing, vendor sign-ups, deleting data, and any cross-org data flow.

---

## 7. Environment notes (Windows dev host)

- **Python:** `py` (3.14). `python`/`python3` are Store stubs.
- **GitHub CLI:** `export PATH="$PATH:/c/Program Files/GitHub CLI"` in each shell. The browser pane can't reach github.com (organization policy).
- **Hub tests:**
  ```bash
  docker run -d --name commontrace-hub-test-db -e POSTGRES_USER=commontrace_dev -e POSTGRES_PASSWORD=devpassword -e POSTGRES_DB=commontrace_hub_test -p 127.0.0.1:5432:5432 postgres:16
  export HUB_TEST_DATABASE_URL=postgresql+asyncpg://commontrace_dev:devpassword@localhost:5432/commontrace_hub_test
  ```
  - Install `hub/requirements.txt` without `numpy==2.2.6`: there's no 3.14 wheel and no compiler here, and the Hub has a pure-Python fallback.
  - The full Hub suite takes about 70 minutes here. Run it in the background and keep the full log.
  - One suite per database at a time.
- **Known Windows-only failures** in the root suite, all environmental: 35 tests (symlink `WinError 1314`, `install.sh` under Git Bash, order-dependent flakes that pass alone). Diff new failures against this baseline.
- **Other projects' containers** are on this machine. Don't touch them.
- **CRLF warnings** on commit are harmless.

---

## 8. Metrics (report every phase against `metrics/baseline.json`)

- **North star:** proven `occasions_improved` per active org per month (causal only).
- **Revenue:** paid pilots, pilot → paid conversion, MRR, net revenue retention, gross margin per proven occasion.
- **Activation:** time to first injected lesson, first recorded outcome and first verdict; % of active lessons with a verdict; % of occasions with an automatic outcome.
- **Trust:** % of experiments compromised (with reasons), false-verdict rate on the harness, time to withdraw a harmful memory.
- **Quality:** CI green rate, p95 latencies, open High/Medium audit findings (target 0).

---

## 9. Owner decisions (raise at the start, each with a recommendation)

1. **Beachhead function.** Recommend: support, then coding.
2. **Design partners.** Recommend five: 2 support, 1 sales, 1 HR, 1 coding. You draft the outreach; the owner sends it.
3. **Cloud provider and region.**
4. **Prices.**
5. **Workbench architecture.** Recommend: a local console first.
6. **OTLP protobuf support** behind an extra (yes or no).
7. **Model keys for the draft-quality harness,** and live vendor accounts for adapter smoke tests.
8. **The privacy layer and opt-in cross-org learning.**
9. **Legal entity, DPA, SOC 2 auditor.**
10. **Brand, domain, and whether to publish to PyPI and MCP registries.**

---

## 10. Report format after each phase

1. What shipped (commit or PR, one line each).
2. Acceptance criteria: pass/fail, with the command output that proves it.
3. Metrics against the baseline.
4. Bugs found and fixed; anything deferred, with the reason.
5. Owner decisions needed, each with a recommendation.
6. The next phase's plan.

Start with Phase A. Do not skip ahead.
