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
