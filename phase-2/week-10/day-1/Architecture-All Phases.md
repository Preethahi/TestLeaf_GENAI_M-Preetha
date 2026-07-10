# Moderation Layer — Rule-Based API Gateway
## Architecture Document v1.0 (Phases 1–5, approved)

---

# PHASE 1 — System Overview

## 1. Purpose
A **rule-based moderation gateway** that sanitizes all text and file content BEFORE it reaches any LLM. It detects and masks PII/CII, blocks secrets, toxic content, prompt injections, invalid files, and oversized payloads. Only safe (masked/allowed) data is forwarded downstream.

**Hard constraints:** Backend-only (REST APIs), no UI. Pure rule engine — no LLM/ML in the moderation path. All rules configured in JSON files (configurable & extensible). Individual API per detector + one orchestrator API.

## 2. Tech Stack
| Layer | Choice |
|---|---|
| Runtime | Node.js >= 18 |
| Framework | Express.js |
| File upload | multer (memory storage) |
| Text extraction | pdf-parse (PDF), mammoth (DOCX) |
| MIME check | magic-byte verification (built-in; file-type optional) |
| Tokenizer | gpt-tokenizer (offline; char/4 fallback) |
| Hot-reload | chokidar (fs.watch fallback) |
| Logging | structured JSON audit logs (pino-compatible interface) |
| Security | x-api-key auth + per-key rate limiting |

## 3. High-Level Flow
```
Client (Text / File Upload)
        │
        ▼
API Gateway (Express) ── x-api-key auth + rate limit
        │
        ▼
1. File Validator + Extraction ── BLOCK on fail
2. Token Size Validator ───────── BLOCK on overflow
3. Secret Detector ────────────── BLOCK
4. Prompt Injection Detector ──── BLOCK (scored)
5. Toxic Content Detector ─────── BLOCK/MASK/FLAG (tiered)
6. PII Detector ───────────────── MASK  [EMAIL_1]...
7. CII Detector ───────────────── MASK  [PROJECT_1]...
        │
        ▼
Decision Engine: BLOCK > MASK > FLAG > ALLOW (fail-fast)
        │
   ┌────┴────┐
BLOCKED    SAFE PAYLOAD (safeText + maskMap) ──► LLM (out of scope)
```

## 4. Core Design Principles
1. **Fail-fast blocking** — any BLOCK verdict stops the pipeline; remaining detectors are skipped.
2. **Reversible masking** — placeholder tokens map to originals in a per-request `maskMap` returned to the caller (for de-masking LLM responses). Raw values are never logged.
3. **Everything is a rule** — patterns, wordlists, severities, actions, thresholds, limits all live in `config/rules/*.json`. New PII type = JSON entry, zero code.
4. **Hot-reload** — file watcher on `config/rules/` + explicit `/admin/rules/reload`. Invalid files rejected; previous ruleset stays active (never ruleless).
5. **Stateless** — no DB; audit trail in rotating JSON log files (counts only, never raw values or maskMap).

## 5. Detector Order Rationale
File (cheapest reject) → Tokens (avoid scanning oversized payloads) → Secrets & Injection (hard-block classes, fail fast) → Toxicity (block/mask) → PII → CII (mask-only, run last on surviving content).

---

# PHASE 2 — Detector Specs & JSON Rule Schemas

## 6. Rule Configuration Layout
```
config/
├── app.config.json          # port, API keys, rate limits
└── rules/
    ├── pii.rules.json            ├── prompt-injection.rules.json
    ├── cii.rules.json            ├── file-validation.rules.json
    ├── secrets.rules.json        ├── token-limits.rules.json
    ├── toxicity.rules.json       └── decision.rules.json
```
Common envelope: `{ "version": "1.0", "enabled": true, ... }`. Every regex is compiled and validated at load time; invalid rules are rejected atomically.

## 7. Detectors
- **7.1 PII (MASK):** regex rules per type — EMAIL, AADHAAR (Verhoeff), PAN, PHONE_IN, CREDIT_CARD (Luhn), SSN, IP_ADDRESS, PASSPORT_IN, DOB. Optional `validator` (luhn/verhoeff) cuts false positives. Output: `[EMAIL_1]`-style placeholders + maskMap.
- **7.2 CII (MASK, hybrid):** `dictionaries` (project codenames, client names — compiled to word-boundary alternations) + `patterns` (EMP-ID, internal domains, tickets).
- **7.3 Secrets (BLOCK):** AWS keys, GitHub/OpenAI/Google tokens, JWT, PEM private keys, password assignments, DB connection strings. `requiresContext` = keyword within ±80 chars to reduce false positives. Values never echoed/logged.
- **7.4 Toxicity (tiered):** HIGH→BLOCK, MEDIUM→MASK `[REDACTED_n]`, LOW→FLAG. `detectObfuscation` normalizes leetspeak via `obfuscationMap` (d@mn→damn). Terms maintained by the org in JSON.
- **7.5 Prompt Injection (scored):** weighted `signatures` (ignore-instructions, system-prompt-leak, DAN, role override…) + pluggable `heuristics` (base64Detection, delimiterDensity, imperativeDensity). Σweights ≥ blockThreshold(10) → BLOCK; ≥ flagThreshold(5) → FLAG.
- **7.6 File Validator (BLOCK on fail):** extension whitelist (.txt .md .csv .json .pdf .docx), magic-byte verification (%PDF / PK zip), UTF-8 check for text types, max size 10MB, filename sanitization (path traversal, control chars), empty-file rejection. Valid files → text extracted (pdf-parse / mammoth / utf-8) into the pipeline.
- **7.7 Token Validator:** offline tokenizer; `maxTokens` 8000 (BLOCK or TRUNCATE per `onOverflow`), `warnTokens` 6000 (FLAG).
- **7.8 Decision Engine:** `precedence: [BLOCK, MASK, FLAG, ALLOW]`, `failFast`, `pipelineOrder` (reorder/disable without code), `detectorActionOverrides` (e.g. escalate piiDetector→BLOCK).

(Full JSON rule files ship in `config/rules/` — they are the source of truth.)

---

# PHASE 3 — API Design

Base: `http://localhost:3000/api/v1` · Auth: `x-api-key` · Rate: 100 req/min/key

| # | Method | Endpoint | Purpose |
|---|---|---|---|
| 1 | POST | /detect/pii | PII detection + masking |
| 2 | POST | /detect/cii | CII detection + masking |
| 3 | POST | /detect/secrets | Secret detection (BLOCK) |
| 4 | POST | /detect/toxicity | Tiered toxicity |
| 5 | POST | /detect/prompt-injection | Injection scoring |
| 6 | POST | /validate/file | File validation + extraction |
| 7 | POST | /validate/tokens | Token count check |
| 8 | POST | /moderate | Full pipeline (text) |
| 9 | POST | /moderate/file | Full pipeline (file upload) |
| 10 | POST | /admin/rules/reload | Reload JSON rules |
| 11 | GET | /admin/rules/status | Active ruleset versions |
| 12 | GET | /health | Liveness (no auth) |

**Common envelope:** `{ requestId, timestamp, detector, verdict: ALLOW|MASK|BLOCK|FLAG, processingTimeMs, data }`.
**Errors:** 400 ERR_INVALID_INPUT · 401 ERR_UNAUTHORIZED · 413 ERR_PAYLOAD_TOO_LARGE · 415 ERR_UNSUPPORTED_TYPE · 422 ERR_RULE_VALIDATION · 429 ERR_RATE_LIMITED · 500 ERR_INTERNAL. A BLOCK verdict is HTTP 200 (successful moderation result).
**Caller contract:** send `data.safeText` to the LLM; keep `data.maskMap` client-side to de-mask the LLM response. (Full request/response samples: README.md.)

---

# PHASE 4 — Project Structure & Component Design

```
moderation-gateway/
├── config/{app.config.json, rules/*.rules.json}
├── src/
│   ├── server.js, app.js
│   ├── core/ ruleStore, ruleWatcher, regexCompiler, maskEngine,
│   │         validators/{luhn,verhoeff}, heuristics/{base64,delims,imperative}
│   ├── detectors/ baseDetector + 7 detectors (uniform interface)
│   ├── pipeline/ orchestrator, decisionEngine
│   ├── extraction/ pdf, docx, text dispatch
│   ├── middleware/ requestId, auth, rateLimiter, validateBody, upload, errorHandler
│   ├── routes/ detect, validate, moderate, admin, health
│   └── utils/ logger (audit channel), responseBuilder
├── logs/ audit-YYYY-MM-DD.log
└── tests/run-core-tests.js
```

**Detector interface:** `detect(text, ctx) → { detector, verdict, detections[], maskedText, maskMap, meta }`. All detectors identical to the orchestrator → detector #8 = new module + one pipelineOrder entry.
**RuleStore:** load → validate → compile regexes → atomic swap; invalid file keeps previous ruleset. Detectors read from cache (O(1)), never the filesystem.
**MaskEngine:** right-to-left replacement, longest-match-wins overlap resolution, one shared MaskContext across PII/CII/toxicity → merged maskMap, no numbering collisions.
**Orchestrator:** iterates pipelineOrder, applies overrides, fail-fast on BLOCK, chains maskedText into the next detector.
**Middleware order:** requestId → auth → rateLimiter → body/multer → validate → route → errorHandler.

**Extensibility playbook (zero-code):** new PII/secret/CII/toxicity/injection rule = JSON edit + hot-reload. New heuristic/validator/file-type/detector = one small module + registry entry.

---

# PHASE 5 — Roadmap, Testing, NFRs

**Milestones:** M1 foundation (server, auth, rate limit) → M2 rule engine + hot-reload → M3 text detectors + /detect routes → M4 file & token validators → M5 pipeline + /moderate → M6 hardening/docs.

**Testing:** unit per detector (hit/miss/overlaps/disabled rules), maskEngine numbering, luhn/verhoeff vectors, heuristic boundaries, ruleStore invalid-file retention; integration via supertest (auth 401/429, contract shape, fail-fast order, override escalation, file fixtures incl. renamed exe, traversal filename, oversize, empty); accuracy corpus per detector re-run on every rule change. `npm run test:core` ships 29 offline assertions.

**NFRs:** p95 < 50ms/5KB text (pre-compiled regex, no I/O in hot path); ReDoS guard (pattern cap + load-time compile); stateless horizontal scaling; nothing persisted except audit logs (no raw values, no maskMap); atomic rule swap — never ruleless; graceful SIGTERM drain; /api/v1 versioning.

**Out of scope:** LLM/ML inference, UI, database, response de-masking (caller uses maskMap), OCR/images.

---

# PHASE 6 — Deployment & DevOps
Multi-stage Alpine Docker image (non-root, HEALTHCHECK on /health); rules mounted as a read-only volume so hot-reload updates rules with zero restarts while the image stays immutable. docker-compose for local; dev/staging/prod env strategy with ENV-var overlay (no secrets in images/JSON). CI/CD: lint+unit → **rule-validation gate** (every rules PR runs through ruleStore; invalid rules fail CI) → integration → image build + trivy scan → deploy with **golden-sample smoke tests** (fixture texts asserted to exact verdicts). Rules-as-code: config/rules in Git, PR-reviewed, synced to the volume, hot-reloaded, git-revert rollback. K8s: rules as ConfigMap, readiness = /health, SIGTERM drain, HPA-ready (stateless).

# PHASE 7 — Observability & Monitoring
prom-client metrics on a separate internal port (9091): request counters/histograms by endpoint+verdict, per-detector executions/verdicts/durations, `detector_hits_total{rule_id}` (rule IDs only — never content), pipeline blocked_by/fail-fast skips/mask volumes, rule reload counters + active-version gauges, auth/rate-limit/error counters. Three Grafana dashboards: Gateway Overview, Detector Analytics, Rules Operations. Alerts: GatewayDown, RuleReloadFailing, LatencyBudgetBreach (p95>50ms), SecretDetectorSpike, BlockRateSpike and **BlockRateZero** (the "silently emptied ruleset" canary). Logs shipped via Promtail/Loki, joined to metrics/traces by requestId (x-request-id header). Optional sampled OpenTelemetry spans per detector. New fail-closed readiness probe /health/ready (all rulesets loaded + watcher active).

# PHASE 8 — Security Hardening
TLS at ingress (gateway never internet-exposed; optional mTLS for internal callers). API keys upgraded to SHA-256 hashes with constant-time compare, metadata (name/scopes/expiry), dual-key zero-downtime rotation, and scopes ("moderate"/"detect"/"admin" — admin endpoints require admin scope + ingress IP allowlist). Secrets from ENV/Vault only. Input hardening additions: safe-regex lint in CI, **per-request scan time budget (>500ms → BLOCK, fail-closed)**, docx zip-bomb extraction cap, prototype-pollution key rejection. helmet headers, CORS default-deny, generic error bodies. Full OWASP API Top-10 mapping. Supply chain: npm ci + lockfile, audit + trivy gates, minimal deps (core engine is zero-dependency). Runtime: non-root, read-only filesystem, dropped capabilities; maskMaps request-scoped, never cached. Invariant: any internal detector/pipeline error resolves to BLOCK, never ALLOW.

# PHASE 9 — Performance & Scalability
Targets: p95<50ms, p99<100ms @5KB; ~1.5–3k RPS/instance; linear horizontal scale. Optimizations: merged same-flag patterns into single named-group alternations (one scan per detector), **Aho-Corasick tries for CII/toxicity dictionaries** (O(text) regardless of term count, rebuilt on reload), fail-fast + cheap-first ordering, tokenizer binary-search truncation. Multi-core via node:cluster/PM2 (per-worker watchers converge within debounce; no IPC). File extraction isolated in a worker_threads (piscina) pool — or a dedicated files deployment — so PDF parsing never blocks text latency. Global rate limits via Redis (rate-limiter-flexible), fail-OPEN to memory (quota control) while moderation stays fail-CLOSED (safety control). Capacity math + autocannon bench matrix (S/M/L/blocked/file corpora) in CI with >10% p95 regression alerts. Rule-author guardrails: pattern-complexity lint + per-pattern 2ms micro-bench gate — perf budget survives ruleset growth.

# PHASE 10 — Advanced Rule Management
Rule lifecycle DRAFT→SHADOW→ACTIVE→DEPRECATED→RETIRED via two schema additions: `mode: "enforce"|"shadow"` and a `meta` governance block (owner/ticket/reason). **Shadow mode** runs rules on live traffic, records hits to metrics/audit, but never affects verdicts/masking — every risky rule change becomes a measured, one-line-reversible canary; a verdict-diff report shows would-have-blocked counts before promotion. Ruleset SHA-256 **hash stamping** in /admin/rules/status, metrics, and every audit line → any decision traces to exact rule bytes; rollback via enabled:false (seconds), git revert, or volume snapshot. Embedded per-rule test vectors (`tests.mustMatch/mustNotMatch`) executed in CI. Analytics: per-rule hit dashboards, dead-rule detection (0 hits/90d), privacy-gated ±30-char sampling for SHADOW rules only. New admin APIs: GET rules/:ruleset (+hash), /diff (drift), POST /validate (CI-grade dry-run), GET /shadow-report.

# PHASE 11 — De-masking & LLM Response Moderation
Round trip: /moderate → {safeText, maskMap} → LLM → **/moderate/response** → de-mask + output-moderate → user. demasking.rules.json governs edge cases: invented placeholders stripped (never fabricate PII), mangled placeholders fuzzy-repaired, maxSubstitutions runaway guard. maskMap custody Mode A (caller-held, stateless, default) or Mode B (gateway session: TTL≤300s, single-use, encrypted, excluded from all logs). Output pipeline has its own response-decision.rules.json (secrets→BLOCK, new PII/CII→MASK, toxicity→BLOCK; injection/file/token skipped). Ordering: **de-mask first, then moderate** — closes the gap where blocked content hides inside a placeholder. New APIs: POST /demask, POST /moderate/response. Failures: corrupt maskMap → BLOCK; expired session → 410, caller shows masked text; response-side detectors fail closed.

# PHASE 12 — Multi-tenancy & Policy Profiles
TENANT (consuming app) → PROFILE (named policy) → BINDING (API key metadata). Rules stay one global reviewed library; profiles only parameterize: enable/disable (honoring per-rule `lockable` — secrets rules not disableable by default), action overrides, thresholds, token limits, tier promotion, and **append-only tenantDictionaries** (tenants add CII/toxicity terms, never remove global ones). Profiles use single inheritance from "default" (= current behavior; migration is invisible), deep-merged at load into immutable cached snapshots — zero hot-path I/O. Isolation: Mode B sessions tenant-checked (403 cross-tenant), audit/metrics gain tenant+profile labels, optional per-tenant rate pools, broken profile keeps previous snapshot. Admin: GET /profiles, GET /profiles/:name (post-merge resolved view), POST /profiles/validate, GET /tenants/:t/report. Rollout: bind all keys to default → candidate profile in shadow → verdict-diff → flip binding.

# PHASE 13 — Compliance & Audit
Data-flow map with a code-verifiable core claim: **zero personal-data persistence by default** — raw text/maskMaps exist only in request memory (or TTL'd single-use Mode B sessions); only decision metadata (types+counts) persists. DPDP/GDPR principle mapping (minimization is the product itself). compliance.rules.json is a **ceiling config**: retention days, session TTL max, shadow-sample caps — other configs validate against it and cannot exceed it; nightly purge job deletes expired artifacts and audit-logs its own deletions. Auto-generated auditor evidence pack: decision summaries, rulesetHash↔git↔PR provenance chain, change log from rule meta blocks, control attestations (rotations, purges, alerts), resolved-config snapshot. DSR position: nothing to retrieve/erase server-side, verified each release by a CI persistence scan; masked placeholders mean the LLM often never received identifiers. One-switch incident mode: all tenants → strictest profile, tagged audit lines, reversible.

# PHASE 14 — Extension Roadmap
Future rule-based detectors in waves: language/script gate, financial IDs (IFSC/UPI/IBAN/GSTIN with mod-97 checksum validator), URL/domain policy; then encoding-smuggler (zero-width/RTL/nested-base64), structured-data field gates, DLP classification markers; image metadata rules (EXIF only, no OCR). Plugin architecture: allow-listed plugins/ directory (detector/heuristic/validator/extractor types) implementing the Phase 4 contract, sandboxed in the worker pool with the 500ms budget and no network/fs — plugins can only fail CLOSED and never bypass the decision engine; identical CI + shadow-first rollout. API evolution: /v1 additive-only, OpenAPI spec + golden samples as contract tests; v2 triggers = batch, streaming (NDJSON), verdict enum changes, with ≥6-month side-by-side + Sunset headers. Maintenance model: platform owners + per-domain rule stewards via CODEOWNERS; weekly rule review, monthly deps, quarterly rotations/benchmarks, yearly threat-model refresh; KPIs incl. false-positive rate and same-day time-to-enforce. **Design invariants:** no LLM/ML in the moderation path; fail closed; no raw values at rest; rules are reviewed data; one detector contract forever.
