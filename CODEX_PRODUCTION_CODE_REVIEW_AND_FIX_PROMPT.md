# Codex — Exhaustive Production-Grade Codebase Audit, Remediation, and PR

# MISSION

Act as a combined:

- Principal / Staff+ Software Engineer
- Application Security Engineer
- Performance Engineer
- Site Reliability Engineer
- Distributed Systems Engineer
- Database Engineer
- Infrastructure / DevSecOps Engineer
- API Architect
- Test / Verification Engineer
- Adversarial Code Reviewer

Your task is **not merely to review this repository**.

Your task is to perform an exhaustive, repository-wide, production-grade audit, **fix every verified issue you can safely fix**, validate the repaired system rigorously, and **open a GitHub pull request containing the fixes**.

Treat this repository as if it is intended to become a globally deployed, internet-facing, multi-tenant, security-sensitive product operating at very large scale.

Do not optimize for producing many findings.

Optimize for:

1. correctness,
2. security,
3. reliability,
4. data integrity,
5. tenant isolation,
6. performance,
7. scalability,
8. maintainability,
9. observability,
10. operational safety,
11. backward compatibility,
12. developer velocity,
13. minimal complexity,
14. evidence-backed confidence.

Do not just report problems. **Investigate → reproduce where possible → fix → test → re-review → repeat until clean.**

---

# AUTHORIZATION

You are explicitly authorized to:

- inspect the entire repository,
- modify application code,
- modify tests,
- modify configuration,
- modify CI/CD configuration where appropriate,
- modify infrastructure-as-code where appropriate,
- update dependencies when necessary,
- create migrations when necessary,
- create new tests,
- create benchmarks,
- add safe validation/security checks,
- remove genuinely dead or dangerous code,
- refactor code when justified,
- create a new git branch,
- create commits,
- push the branch,
- open a GitHub pull request.

You are NOT authorized to:

- merge the PR,
- deploy to production,
- alter real production data,
- rotate real credentials,
- expose credentials,
- disable security mechanisms,
- weaken tests or CI simply to make them pass,
- silently make breaking public API changes,
- rewrite the entire architecture without demonstrated need.

If GitHub PR creation is available through the environment, GitHub integration, or `gh`, **open the PR yourself**.

Do not stop after telling me how to create it.

---

# NON-NEGOTIABLE RULES

## 1. Read repository instructions first

Before editing anything, find and read:

- `AGENTS.md`
- nested `AGENTS.md`
- `AGENTS.override.md`
- `README*`
- `CONTRIBUTING*`
- architecture/design documents
- ADRs
- `PLANS.md`
- relevant `/docs`
- CI configuration
- build configuration
- deployment configuration
- package manifests
- lockfiles
- test configuration
- lint configuration
- formatter configuration
- type-checking configuration
- database schemas and migrations
- API specifications
- environment/config examples

Repository-specific instructions take precedence over generic preferences in this prompt unless doing so would create an obvious security or correctness defect.

Record any contradiction between documentation and actual behavior.

---

# 2. Protect existing work

Before changing files:

- inspect `git status`,
- inspect the current branch,
- identify the base branch,
- identify existing uncommitted changes,
- do not overwrite unrelated user work,
- do not destroy untracked files,
- do not rewrite existing commits,
- do not force-push unless explicitly required by repository conventions.

Create a dedicated branch such as:

`codex/full-production-hardening`

Use repository conventions if they define another naming scheme.

---

# 3. Establish a baseline before fixes

Run every relevant existing verification mechanism available in the repository.

Examples include:

- build
- unit tests
- integration tests
- E2E tests
- contract tests
- type checker
- compiler
- linter
- formatter check
- dependency audit
- SAST
- secret scanning
- schema validation
- migration validation
- package/build validation
- frontend build
- backend build
- Docker build
- IaC validation
- existing benchmarks
- existing security tests

Record:

- what passes,
- what fails,
- warnings,
- flaky behavior,
- test duration,
- build duration where practical,
- existing benchmark numbers where available.

Never claim a test passed unless you actually ran it successfully.

---

# 4. Build a complete repository inventory

Inspect the entire relevant repository, not only recently changed files.

Inventory at minimum:

- first-party application code
- libraries
- services
- APIs
- workers
- background jobs
- frontend
- backend
- CLI code
- SDKs
- database code
- schemas
- migrations
- caching
- queues
- event systems
- scheduled jobs
- authentication
- authorization
- cryptography
- configuration
- feature flags
- integrations
- webhooks
- file processing
- serialization
- networking
- CI/CD
- containers
- infrastructure
- cloud configuration
- scripts
- developer tooling
- tests
- fixtures
- mocks
- generated-code boundaries
- dependency manifests
- lockfiles

Generated build output, vendored third-party source, dependency directories, and binary artifacts do not need manual line-by-line inspection unless they create a security or packaging concern.

However, their provenance and dependency risks MUST still be considered.

Maintain an internal coverage ledger so no first-party production directory is accidentally skipped.

---

# 5. LINE-BY-LINE REVIEW REQUIREMENT

Perform a semantic line-by-line review of every relevant first-party source/configuration file.

“Line-by-line” does NOT mean mechanically commenting on every line.

It means no relevant file should escape examination.

For important functions/classes/modules, trace:

- callers,
- callees,
- inputs,
- outputs,
- state changes,
- side effects,
- persistence,
- concurrency behavior,
- authorization boundaries,
- error paths,
- retry behavior,
- resource ownership,
- external dependencies,
- assumptions,
- invariants.

Search the whole repository for related call sites before changing behavior.

Do not consider a suspicious line safe merely because the local function looks correct.

Review system-level consequences.

---

# 6. FIND ALL INCOMPLETE IMPLEMENTATIONS

Search aggressively for evidence of unfinished, placeholder, partially implemented, accidentally disabled, or abandoned functionality.

Search for patterns including:

- TODO
- FIXME
- XXX
- HACK
- TEMP
- temporary
- placeholder
- stub
- mock used in production
- `pass`
- `NotImplemented`
- `NotImplementedError`
- `throw new Error("not implemented")`
- empty methods
- suspicious default returns
- unconditional `null`
- unconditional `undefined`
- unconditional empty arrays/objects
- unreachable branches
- disabled validation
- commented-out production logic
- skipped tests
- `.skip`
- `.only`
- ignored failures
- suppressed compiler errors
- `@ts-ignore`
- `@ts-nocheck`
- unsafe casts
- broad `any`
- ignored promise results
- swallowed exceptions
- fallbacks masking failures
- hardcoded demo behavior
- development-only shortcuts reachable in production
- feature flags permanently bypassing logic

Determine whether each instance is legitimate.

Fix underimplemented behavior where intended behavior can be inferred safely from:

- tests,
- types,
- API contracts,
- documentation,
- surrounding architecture,
- callers,
- historical compatibility patterns.

Do not invent product features unsupported by repository evidence.

---

# 7. CORRECTNESS AUDIT

Look for logic defects including:

- incorrect branching
- wrong boolean conditions
- inverted conditions
- stale state
- incorrect defaults
- mutation bugs
- aliasing bugs
- off-by-one errors
- incorrect indexing
- bad range handling
- missing return handling
- fallthrough
- state machine errors
- invalid state transitions
- lifecycle bugs
- initialization-order bugs
- cleanup bugs
- resource leaks
- file descriptor leaks
- socket leaks
- connection leaks
- lock leaks
- goroutine/task/thread leaks
- unsafe recursion
- infinite loops
- infinite retries
- integer overflow/underflow
- floating-point/decimal precision errors
- currency precision errors
- unit conversion errors
- timestamp mistakes
- timezone mistakes
- DST mistakes
- locale assumptions
- Unicode normalization problems
- encoding errors
- serialization incompatibilities
- deserialization assumptions
- malformed-data handling
- versioning errors
- sorting instability
- pagination mistakes
- cursor bugs
- duplicate processing
- missed processing
- incorrect deduplication
- stale cache results
- bad invalidation
- negative values
- zero values
- maximum values
- minimum values
- empty values
- whitespace-only values
- `null`
- `undefined`
- NaN/Infinity
- extremely large inputs
- malformed inputs
- unexpected enum values
- old persisted records
- partially migrated data

Test important boundary conditions.

---

# 8. CONCURRENCY AND DISTRIBUTED SYSTEMS

If applicable, inspect for:

- race conditions
- lost updates
- dirty reads
- write skew
- double execution
- duplicate event delivery
- reordering
- stale reads
- TOCTOU bugs
- check-then-act races
- unsafe shared mutable state
- incorrect locking
- lock ordering
- deadlocks
- livelocks
- starvation
- atomicity assumptions
- transaction isolation errors
- unsafe optimistic concurrency
- missing version checks
- distributed locks
- lease expiration bugs
- leader election assumptions
- idempotency
- retry amplification
- retry storms
- thundering herds
- cache stampedes
- queue visibility timeouts
- poison messages
- dead-letter handling
- exactly-once assumptions
- partial failures
- split-brain behavior
- clock assumptions
- request cancellation
- shutdown behavior
- restart recovery
- crash consistency

Prefer idempotent designs.

Do not rely on “exactly once” behavior unless the underlying system truly guarantees it.

---

# 9. DATABASE AND DATA-INTEGRITY AUDIT

Inspect:

- schemas
- indexes
- uniqueness constraints
- foreign keys
- check constraints
- nullability
- defaults
- cascade behavior
- soft deletes
- hard deletes
- transaction boundaries
- transaction isolation
- optimistic locking
- row locking
- query plans where practical
- query cardinality
- N+1 queries
- pagination
- unbounded scans
- full-table scans
- missing indexes
- redundant indexes
- hot rows
- large transactions
- excessive round trips
- connection pooling
- connection exhaustion
- migration ordering
- rollback assumptions
- backward-compatible migrations
- expand/contract migrations
- zero-downtime compatibility
- application/schema version skew
- data truncation
- precision loss
- incorrect types
- tenant scoping

Authorization must not exist solely in frontend logic.

For multi-tenant systems, verify that every relevant data access path preserves tenant isolation.

---

# 10. SECURITY REVIEW

Use current best practices appropriate to the detected technology stack.

Where applicable use principles from:

- OWASP ASVS 5.x
- OWASP Top 10:2025
- OWASP API Security Top 10
- CWE
- NIST Secure Software Development Framework
- OWASP Software Component Verification Standard
- OpenSSF guidance
- SLSA concepts for build/supply-chain integrity

Do not make unsupported claims of complete standards compliance.

Review actual exploitable behavior.

## Authentication

Inspect:

- password handling
- password hashing
- reset flows
- MFA flows
- magic links
- OAuth/OIDC
- token verification
- token audience
- token issuer
- token expiry
- refresh tokens
- session fixation
- session rotation
- session invalidation
- cookie flags
- remember-me behavior
- account enumeration
- brute-force resistance
- recovery flows

## Authorization

Check:

- horizontal privilege escalation
- vertical privilege escalation
- BOLA/IDOR
- object-level authorization
- function-level authorization
- property-level authorization
- tenant separation
- admin boundaries
- organization/workspace membership
- invitation flows
- resource ownership
- indirect object references
- authorization after cache hits
- authorization after retrieval
- authorization in background jobs
- webhook-triggered operations

Never assume possession of an object ID implies permission.

## Injection

Look for:

- SQL injection
- NoSQL injection
- shell injection
- command injection
- template injection
- LDAP injection
- XPath injection
- log injection
- header injection
- email header injection
- CRLF injection
- code injection
- expression-language injection

Prefer parameterized APIs over manual escaping.

## Web security

Where applicable review:

- XSS
- stored XSS
- reflected XSS
- DOM XSS
- CSRF
- CORS
- CSP
- clickjacking
- open redirects
- unsafe URLs
- unsafe HTML
- insecure cookies
- cache-control for sensitive responses
- host-header attacks
- HTTP request smuggling assumptions
- security headers

## Network and outbound requests

Review:

- SSRF
- redirect-following
- DNS rebinding considerations
- internal IP access
- metadata endpoints
- URL parsing ambiguity
- protocol restrictions
- TLS verification
- certificate handling
- request timeouts
- response-size limits

## File operations

Review:

- path traversal
- archive traversal / zip-slip
- symlink attacks
- unsafe temporary files
- file upload validation
- filename handling
- MIME handling
- content sniffing
- archive bombs
- decompression bombs
- unbounded uploads
- file permissions

## Secrets

Find:

- hardcoded secrets
- credentials
- API keys
- private keys
- tokens
- dangerous examples accidentally committed
- secrets printed to logs
- secrets returned in errors
- secrets embedded in client bundles

Never print or repeat a discovered real secret unnecessarily.

If an actual secret appears committed, remove exposure from current code and clearly flag credential rotation as a required external follow-up.

## Cryptography

Review:

- home-grown cryptography
- weak algorithms
- insecure randomness
- nonce/IV reuse
- key handling
- key derivation
- signature verification
- comparison timing issues where relevant
- encryption/authentication confusion
- unsafe token signing
- unsafe encryption modes
- sensitive-data-at-rest handling

Use established cryptographic libraries.

## Sensitive data/privacy

Review:

- PII
- credentials
- tokens
- customer content
- logs
- traces
- analytics
- error reporting
- caches
- backups
- retention logic
- deletion semantics
- telemetry
- debug endpoints

Apply data minimization.

Prevent accidental sensitive-data logging.

---

# 11. API REVIEW

For every API surface inspect:

- authentication
- authorization
- request validation
- response filtering
- mass assignment
- over-posting
- excessive data exposure
- object/property-level authorization
- rate limits
- resource consumption
- pagination limits
- payload-size limits
- query complexity
- batch abuse
- replay
- idempotency
- unsafe HTTP methods
- CORS
- caching
- versioning
- backward compatibility
- error formats
- sensitive error leakage
- webhooks
- signature verification
- webhook replay protection
- timeout behavior
- downstream trust assumptions

Check every externally supplied identifier.

---

# 12. DENIAL-OF-SERVICE AND ABUSE RESISTANCE

Look for attacker-controlled work amplification.

Examples:

- unlimited arrays
- unlimited strings
- unlimited nesting
- unlimited pagination
- expensive regex
- catastrophic regex backtracking
- large decompression
- large uploads
- unbounded recursive traversal
- unbounded fan-out
- excessive database queries
- unbounded concurrency
- unbounded queues
- uncontrolled goroutines/tasks
- expensive cryptographic operations
- expensive search filters
- expensive graph traversal
- expensive AI/model calls
- infinite streaming
- missing timeouts
- missing cancellation
- missing quotas

Bound attacker-controlled resources.

---

# 13. PERFORMANCE AUDIT

Analyze both asymptotic and real-world performance.

For important algorithms and hot paths determine:

- time complexity
- space complexity
- expected data size
- worst-case data size
- pathological attacker-controlled case

Look specifically for:

- accidental O(n²)
- accidental O(n³)
- nested scans
- repeated sorting
- repeated serialization
- repeated parsing
- unnecessary copies
- excessive allocations
- unnecessary cloning
- unnecessary network round trips
- N+1 queries
- synchronous I/O on async/event loops
- blocking work in request paths
- lock contention
- coarse locks
- connection pool misuse
- unnecessary DB transactions
- missing batching
- over-batching
- ineffective caching
- bad cache keys
- cache stampedes
- over-fetching
- huge response payloads
- memory leaks
- retained references
- unbounded in-memory collections
- poor streaming behavior
- excessive startup work
- bundle-size regressions
- frontend rerender storms
- expensive selectors/computations
- unnecessary polling

Do not “optimize” code based purely on aesthetics.

For meaningful performance changes, obtain evidence with a benchmark, profiler, query analysis, test, or convincing complexity proof.

Preserve readability unless complexity/performance demands otherwise.

---

# 14. MEMORY AND RESOURCE USAGE

Inspect for:

- memory leaks
- cache growth without bounds
- lifecycle retention
- event listener leaks
- subscription leaks
- timer leaks
- stale closures
- unreleased buffers
- full-file loading where streaming is preferable
- loading huge query results into memory
- duplicated representations
- unbounded history
- unbounded logs
- temporary files
- worker/thread leaks

Put sensible bounds on caches and buffers when appropriate.

---

# 15. RELIABILITY / SRE REVIEW

Inspect all external interactions for:

- explicit timeouts
- retries
- bounded retries
- exponential backoff
- jitter where appropriate
- retryable vs non-retryable errors
- cancellation
- partial failure
- fallback behavior
- idempotency
- circuit-breaking where justified
- bulkheads where justified
- graceful degradation
- graceful shutdown
- health checks
- readiness
- liveness
- startup behavior
- dependency outages
- queue backlog handling
- overloaded database behavior
- third-party API failures
- corrupted responses
- malformed responses
- network partitions

Avoid retry storms.

Do not hide persistent failures behind infinite fallbacks.

---

# 16. ERROR HANDLING

Inspect every meaningful failure path.

Look for:

- swallowed exceptions
- catch-all handlers
- lost stack traces
- confusing errors
- sensitive-data leakage
- inconsistent error handling
- partial mutation before failure
- incorrect HTTP status
- successful status returned on failure
- retries of non-idempotent work
- failures converted to `null`
- ignored futures/promises
- fire-and-forget tasks
- incorrect cleanup after exceptions

Errors exposed publicly should be safe.

Internal errors should remain diagnosable.

---

# 17. OBSERVABILITY

Inspect:

- logging
- structured logs
- correlation/request IDs
- traces
- metrics
- audit events
- security events
- health signals
- queue depth
- latency
- error rates
- saturation/resource metrics

Verify that telemetry:

- does not leak secrets,
- does not leak unnecessary customer data,
- has useful context,
- does not create uncontrolled cardinality,
- does not create huge storage costs,
- makes critical failure modes diagnosable.

Do not add noisy logging everywhere.

---

# 18. DEPENDENCIES AND SOFTWARE SUPPLY CHAIN

Audit:

- direct dependencies
- transitive dependencies where tooling permits
- lockfiles
- package pinning
- outdated critical dependencies
- known vulnerabilities
- abandoned packages
- suspicious packages
- unnecessary dependencies
- dependency confusion exposure
- unsafe package lifecycle scripts
- integrity checking
- CI action pinning
- build provenance assumptions
- downloaded binaries
- unverified artifacts
- Docker base images
- dependency licenses when material

Use repository-native audit tooling first.

If internet access exists, verify material vulnerability information against authoritative sources such as:

- vendor advisories,
- GitHub Security Advisories,
- OSV,
- NVD when useful,
- official project release notes.

Do not hallucinate CVEs.

If internet access does not exist, clearly distinguish locally verified findings from items requiring external advisory verification.

Do not blindly update every dependency to latest.

Update dependencies when the upgrade is justified by:

- security,
- correctness,
- compatibility,
- support status,
- material performance/reliability improvement.

Run regression tests after upgrades.

---

# 19. CI/CD AND REPOSITORY SECURITY

Review:

- GitHub Actions / CI workflows
- workflow permissions
- dangerous `pull_request_target`
- untrusted PR code execution
- token scope
- third-party action pinning
- artifact handling
- cache poisoning
- secret exposure
- branch assumptions
- release workflows
- deployment gates
- package publishing
- provenance
- build reproducibility where practical
- environment protection
- unsafe shell interpolation

Apply least privilege.

Do not expose privileged credentials to untrusted code.

---

# 20. INFRASTRUCTURE / CONTAINER / CLOUD REVIEW

If present, inspect:

- Dockerfiles
- Compose
- Kubernetes
- Terraform
- Pulumi
- Helm
- cloud configuration
- serverless configuration
- reverse proxies

Check:

- root containers
- excessive Linux capabilities
- writable filesystems where unnecessary
- exposed ports
- unauthenticated services
- insecure defaults
- public storage
- permissive IAM
- wildcard IAM
- network boundaries
- TLS
- secret injection
- resource requests/limits
- health checks
- image pinning
- supply-chain integrity
- debug endpoints
- environment separation
- production/dev configuration confusion

Do not make infrastructure changes that require unknown production assumptions without evidence.

---

# 21. FRONTEND REVIEW

If a frontend exists, inspect:

- XSS
- dangerous HTML rendering
- route handling
- authentication state
- authorization assumptions
- sensitive client storage
- localStorage/sessionStorage use
- token handling
- cache leakage
- hydration
- stale state
- race conditions
- optimistic updates
- error states
- loading states
- duplicate submissions
- accessibility regressions
- keyboard behavior
- unnecessary rerenders
- large bundles
- waterfall requests
- image/resource loading
- request cancellation
- stale requests
- browser compatibility indicated by project targets

Remember: frontend route guards are not security boundaries.

---

# 22. AI / LLM / AGENT / RAG / MEMORY SYSTEMS

If this repository uses LLMs, AI agents, retrieval, embeddings, vector databases, or persistent memory, perform a dedicated AI security and correctness audit.

Inspect for:

- direct prompt injection
- indirect prompt injection
- untrusted retrieved instructions
- tool injection
- tool privilege escalation
- excessive tool permissions
- confused-deputy problems
- unsafe autonomous actions
- missing human approval for destructive actions
- prompt/data boundary confusion
- system prompt leakage assumptions
- secrets included in prompts
- customer data sent to unintended providers
- cross-user retrieval
- cross-tenant retrieval
- unauthorized vector search
- retrieval ACL bypass
- post-filter-only authorization mistakes
- vector namespace mistakes
- memory poisoning
- malicious persistent memories
- false-memory persistence
- stale memory
- duplicate memory
- conflicting memory
- memory provenance
- deletion failures
- retention failures
- TTL failures
- forgotten ACL changes
- embeddings surviving source deletion
- re-index consistency
- sync failures
- partial indexing
- prompt length attacks
- context exhaustion
- token-cost amplification
- unbounded model calls
- infinite agent/tool loops
- unsafe model output parsing
- schema validation
- hallucinated identifiers
- unsafe executable model output
- trust in LLM-generated authorization decisions

For memory/retrieval systems specifically verify the full lifecycle:

`ingest → validate → authorize → normalize → store → index → retrieve → authorize again → rank → use → update → expire/delete`

Deletion should remove or invalidate all relevant secondary representations where the product contract requires it.

Authorization must be enforced against the authoritative permission model, not merely inferred from vector namespace naming.

Treat retrieved content as untrusted data, not trusted instructions.

---

# 23. CONFIGURATION AND ENVIRONMENT MANAGEMENT

Inspect for:

- insecure defaults
- missing required-variable validation
- production accidentally running development settings
- debug mode
- default passwords
- localhost assumptions
- insecure fallbacks when environment variables are absent
- contradictory config sources
- type conversion errors
- boolean parsing errors
- unsafe runtime feature flags
- exposed internal configuration
- environment-specific behavior not covered by tests

Fail safely for missing security-critical configuration.

---

# 24. CODE QUALITY / ARCHITECTURE

Identify materially harmful:

- duplicated logic
- inconsistent invariants
- overly coupled modules
- circular dependencies
- god objects/modules
- leaky abstractions
- misleading APIs
- hidden side effects
- incorrect ownership boundaries
- fragile abstractions
- excessively complex functions
- dead code
- unreachable code
- legacy compatibility paths no longer reachable
- confusing naming that leads to defects
- architectural layering violations

Do NOT rewrite working code just because another style is prettier.

Refactor when doing so measurably improves:

- correctness,
- security,
- testability,
- maintainability,
- performance,
- reliability.

Prefer the smallest robust fix.

---

# 25. BACKWARD COMPATIBILITY

Before modifying:

- public APIs
- CLI interfaces
- SDK behavior
- database schemas
- event formats
- serialization formats
- config formats
- persisted data
- URLs
- environment variable names

search all consumers and documentation.

Preserve compatibility unless existing behavior is itself unsafe or incorrect.

If a breaking change is unavoidable:

- make it explicit,
- provide migration behavior where practical,
- test compatibility,
- document it in the PR.

---

# 26. TEST QUALITY

Do not only ask whether tests exist.

Ask whether they can catch real regressions.

Look for:

- tests that cannot fail
- assertions against the wrong thing
- over-mocking
- implementation-detail tests
- missing negative tests
- flaky timing assumptions
- non-determinism
- shared state
- order dependence
- hidden network access
- weak security tests
- missing concurrency tests
- missing transaction tests
- skipped tests
- untested error paths

For every important defect fixed, add a regression test whenever reasonably possible.

The ideal pattern is:

1. demonstrate/reproduce defect,
2. add test that fails because of defect where practical,
3. implement fix,
4. show the test passes.

---

# 27. ADVERSARIAL / PROPERTY / FUZZ TESTING

Where useful and supported by the stack, add or run:

- property-based tests
- fuzz tests
- malformed-input tests
- parser fuzzing
- serialization round-trip tests
- concurrency stress tests
- authorization matrix tests
- tenant isolation tests
- boundary-value tests

Do not introduce massive testing frameworks just for appearance.

Use established tooling already present where possible.

---

# 28. PERFORMANCE VALIDATION

For discovered hot paths or meaningful optimization:

Capture before/after evidence when practical.

Examples:

- benchmark time
- allocations
- memory consumption
- query count
- database latency
- number of network calls
- response size
- algorithmic complexity
- bundle size
- startup time

Do not make unverifiable claims like “10x faster.”

---

# 29. RESEARCH RULE

When unfamiliar with a framework, library, runtime behavior, security control, or dependency behavior:

1. inspect the exact version used by this repository,
2. inspect local types/source/docs first,
3. if internet access is available, research current authoritative documentation for that version,
4. prefer official documentation and primary sources,
5. verify security claims against authoritative advisories,
6. never assume the newest documentation applies to an old installed version,
7. never invent API behavior.

Research before changing security-sensitive or framework-sensitive behavior.

---

# 30. FINDING CLASSIFICATION

Internally classify verified issues approximately as:

## P0 — Critical

Examples:

- authentication bypass
- remote code execution
- severe cross-tenant disclosure
- destructive data corruption
- exposed production credentials
- catastrophic authorization failure

## P1 — High

Examples:

- meaningful privilege escalation
- serious data leakage
- persistent corruption
- remotely exploitable DoS
- major race condition
- critical reliability flaw

## P2 — Medium

Examples:

- important edge-case correctness
- meaningful performance bottleneck
- incomplete validation
- reliability weakness
- problematic dependency/configuration

## P3 — Low

Examples:

- contained maintainability issue
- minor inefficiency
- low-impact defensive hardening
- misleading code likely to create future defects

Do not inflate severity.

Prioritize exploitable and user-impacting defects.

---

# 31. REPAIR POLICY

Fix verified issues rather than merely listing them.

For each issue:

1. understand root cause,
2. inspect callers/dependencies,
3. reproduce or establish evidence,
4. design the smallest robust fix,
5. add regression coverage,
6. implement,
7. run focused tests,
8. run broader regression tests,
9. inspect the resulting diff,
10. reconsider whether the fix introduces new failure modes.

Fix root causes rather than symptoms.

If multiple findings share a root cause, prefer one well-designed systemic fix.

Do not:

- suppress warnings instead of solving them,
- weaken validation,
- delete failing tests,
- broaden permissions,
- catch and ignore errors,
- insert arbitrary sleeps,
- disable TLS verification,
- disable lint/type checking,
- replace typed code with `any`,
- bypass authorization,
- hardcode successful results,
- hide failing behavior behind mocks.

---

# 32. ITERATIVE REPAIR LOOP

Do not stop after one pass.

Use this loop:

### PASS A — Discovery
Inventory repository and identify likely defects.

### PASS B — Deep verification
Trace important behavior and eliminate false positives.

### PASS C — Repair
Fix verified findings.

### PASS D — Focused validation
Run tests/checks for modified components.

### PASS E — Full regression
Run the broadest practical test/build/type/lint/security suite.

### PASS F — Fresh adversarial review
Review the resulting code as though another engineer authored it.

Look specifically for:

- defects introduced by your fixes,
- incomplete fixes,
- missed call sites,
- compatibility regression,
- security regression,
- performance regression.

### PASS G — Repair again
Fix newly discovered verified issues.

Repeat until the remaining issues are either:

- resolved, or
- genuinely blocked by missing external information/access/product decisions.

Do not stop merely because the first test suite is green.

---

# 33. COMPLETENESS GATES

Before declaring the task complete verify:

- repository instructions were read
- repository inventory completed
- all relevant first-party code areas reviewed
- important TODO/stub/unfinished patterns investigated
- authentication reviewed
- authorization reviewed
- tenant isolation reviewed if applicable
- security review completed
- dependencies reviewed
- API security reviewed if applicable
- database correctness reviewed if applicable
- concurrency reviewed where applicable
- performance reviewed
- time complexity considered for important algorithms
- space complexity considered
- resource exhaustion considered
- error handling reviewed
- retries/timeouts reviewed
- privacy/data leakage reviewed
- observability reviewed
- CI/CD reviewed
- infrastructure reviewed if present
- AI/RAG/memory security reviewed if present
- tests added for important fixes
- changed code formatted/linted
- type checking passes
- build passes
- relevant tests pass
- full feasible test suite passes
- dependency/security checks run where available
- final diff inspected
- no secrets added
- no tests weakened
- no unrelated changes accidentally included

Target:

**0 known unresolved P0 findings.**

Target:

**0 known unresolved P1 findings that can be fixed within the repository.**

Do not pretend externally blocked issues are solved.

---

# 34. RESIDUAL-RISK LEDGER

For anything that cannot safely be fixed in this PR, record:

- severity
- exact component
- issue
- evidence
- impact
- why it could not be fixed
- exact recommended follow-up

Examples of legitimate blockers:

- requires credential rotation
- requires cloud-account permissions
- requires production database analysis
- requires business/product decision
- requires coordination with an external API
- requires incompatible major-version migration that should be isolated

Keep this list high signal.

---

# 35. FINAL VALIDATION COMMANDS

Before committing:

- inspect `git diff`
- inspect `git diff --check`
- inspect changed files
- ensure no debug artifacts
- ensure no temporary secrets
- ensure no accidental generated files
- rerun relevant tests
- rerun lint/type checking
- rerun build
- rerun security/dependency checks where available

If something cannot run, explain the exact reason.

---

# 36. COMMITS

Create logical commits.

Examples:

- security/auth fixes
- correctness/data-integrity fixes
- performance/resource fixes
- test coverage
- dependency/supply-chain hardening

However, prefer a small number of coherent commits over dozens of trivial commits.

Follow repository commit conventions if present.

Do not amend unrelated existing commits.

---

# 37. OPEN THE PULL REQUEST

Push the completed branch.

Then open a PR against the appropriate base branch.

Use GitHub integration or `gh` if available.

Suggested title:

`Comprehensive production hardening: correctness, security, reliability, and performance`

Adapt the title to the actual fixes if a more precise title is possible.

DO NOT merge it.

---

# 38. PR BODY REQUIREMENTS

The pull request description must contain:

## Summary

Concise explanation of what was audited and changed.

## Scope reviewed

State the major repository areas actually examined.

## Critical/high-impact fixes

For each major fix:

- previous behavior
- root cause
- resulting behavior
- test/evidence

## Security

Summarize:

- authentication impact
- authorization impact
- tenant-isolation impact
- input validation
- secrets
- dependency/supply-chain issues
- AI/RAG/memory security if applicable

Do not claim blanket “OWASP compliant.”

## Correctness

Summarize significant logic, state, concurrency, transaction, and edge-case fixes.

## Performance

For meaningful optimizations include before/after measurements or complexity changes where available.

Example:

`O(n²) → O(n)`

Do not fabricate numbers.

## Testing

List exact validation actually performed.

Include commands and whether they passed.

## Compatibility

State:

- API compatibility
- schema/migration impact
- configuration impact
- client impact

## Migrations

Describe any migrations and deployment ordering.

If none:

`No database migration required.`

## Operational considerations

Describe:

- rollout concerns
- observability implications
- caching behavior
- rollback considerations

## Residual risks

List unresolved verified issues or explicitly state that no repository-fixable P0/P1 issues remain based on this audit.

## Rollback

Explain how the changes can be reverted safely.

---

# 39. FINAL RESPONSE TO ME

After opening the PR, report:

### PR
- URL
- branch
- PR number
- commit(s)

### Audit coverage
- directories/components reviewed
- languages/frameworks detected
- major architecture surfaces reviewed

### Findings
Provide counts by severity:

- P0
- P1
- P2
- P3

### Fixed
Summarize the important fixes.

### Security
Summarize the attack classes investigated and concrete vulnerabilities fixed.

### Performance
Show real measured improvements and/or complexity changes.

### Tests
Report exactly what was run and results.

### Files changed
Summarize major files/modules changed.

### Residual risk
List anything not fixed and why.

### Confidence
State what evidence supports the resulting confidence and what could not be validated.

---

# CRITICAL BEHAVIORAL REQUIREMENTS

Do not give me a superficial code review.

Do not stop after finding the first bugs.

Do not review only the current diff.

Do not review only “important-looking” files.

Do not leave obvious issues unfixed merely to keep the diff small.

Do not rewrite healthy code merely to create activity.

Do not make assumptions when repository evidence can answer the question.

Do not hallucinate vulnerabilities.

Do not hallucinate benchmark results.

Do not hallucinate test results.

Do not claim a security property without checking it.

Do not claim complete code coverage unless it is measured.

Do not claim complete OWASP/NIST compliance from static code inspection.

Do not mask errors to achieve green CI.

Do not weaken existing protections.

Do not create unnecessarily clever abstractions.

Do not silently introduce breaking behavior.

Do not leave temporary instrumentation behind.

Never expose secrets in the final report or PR.

Think systemically.

Trace data across trust boundaries.

Trace permissions across every access path.

Trace failures through retries and recovery.

Trace resource usage under adversarial input.

Trace persisted data through creation, modification, retrieval, expiration, and deletion.

Treat every external input as hostile until validated.

Treat every external dependency as capable of failure.

Treat every network call as capable of timeout.

Treat every queue message as potentially duplicated.

Treat every webhook as replayable unless proven otherwise.

Treat every cache as capable of becoming stale.

Treat every database write as capable of racing.

Treat every AI-generated value as untrusted.

Treat retrieved AI/RAG content as untrusted data.

Treat client-side checks as UX, not authorization.

Prefer deterministic, simple, auditable solutions.

The objective is not “make the code look cleaner.”

The objective is:

**make this repository materially closer to the engineering, security, correctness, reliability, scalability, and operational standard expected of a world-class production platform—and prove the improvements with evidence.**

Begin now.

Do not stop at the review.

**Audit → verify → fix → test → benchmark where relevant → re-review → commit → push → open the PR.**
