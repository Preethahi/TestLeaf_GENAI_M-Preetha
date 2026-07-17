# Architecture.md — AI-Powered UI Visual Comparison Tool (with JIRA/TestRail Integration)

**Status:** Approved for implementation
**Spec version:** 2.0
**Audience:** Any coding agent (Claude, GitHub Copilot, Cursor, or IDE-integrated LLM) building this system from scratch, with no further clarification required.

---

## 1. System Overview

A web tool that lets a QA tester get an AI-assisted visual QA report for a UI screenshot, either via manual image upload or via automatic retrieval from JIRA and TestRail — without any browser automation. Two top-level workflows:

- **Current Sprint (New Feature/Page)** — a new feature/page with no baseline. Single image in, manual upload only. Vision-LLM-only checklist analysis (layout, alignment, text, spacing, colour, missing/extra elements).
- **Regression (Existing Feature/Page)** — an existing feature/page with a baseline. Two images in (baseline + current), sourced either by **manual upload** or by **automatic fetch from JIRA (baseline) + TestRail (current)**. Tester picks one of three sub-modes: **Pixel-by-pixel**, **Text-extraction diff**, or **Hybrid** (pixel + text).

Every comparison run — regardless of image source — produces a downloadable, self-contained **HTML report** with a findings table, a severity pie chart, and a numeric + categorical severity score. The comparison, scoring, and reporting pipeline is identical whether images arrive by upload or by integration fetch (mandatory req 4.6).

The system is **stateless**: nothing is persisted beyond the lifetime of a single run. Uploaded or fetched images live in a temp folder that is cleaned up after the run; the HTML report is generated to a short-lived temp location, downloaded once, and cleaned up on the same cadence.

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React.js + TypeScript |
| Backend | Node.js + Express |
| Pixel diff | pixelmatch + sharp (deterministic library — never an LLM). Substituted for Resemble.js: Resemble.js's `node-canvas` dependency requires native compilation and has no prebuilt binary for current Node runtimes without a local MSVC/build toolchain — pixelmatch (diff algorithm) + sharp (PNG/JPEG/WebP decode via prebuilt libvips binaries) is a pure drop-in for the same deterministic, non-LLM role. |
| Vision analysis (layout/text/alignment) | Groq-hosted vision model, configurable via `.env` |
| Reporting / severity scoring | A **different** Groq-hosted model, configurable via `.env` |
| Report charts | Chart.js via CDN, embedded directly in the generated HTML |
| JIRA integration | JIRA REST API v2/v3 (Basic auth: email + API token), configurable via `.env` |
| TestRail integration | TestRail REST API v2 (Basic auth: username + API key), configurable via `.env` |

**Build order (mandatory):** API first (all endpoints, contracts, and business logic working end-to-end via curl/Postman), then the React UI is built against the finished API. Do not build UI components before their backing endpoint exists and is tested.

---

## 3. Project Structure

```
visual-comparison-tool/
├── server/                          # Node.js + Express API
│   ├── src/
│   │   ├── routes/
│   │   │   ├── v1/
│   │   │   │   ├── comparisons.routes.ts
│   │   │   │   ├── config.routes.ts
│   │   │   │   ├── reports.routes.ts
│   │   │   │   └── health.routes.ts
│   │   │   └── index.ts
│   │   ├── controllers/
│   │   │   ├── newFeature.controller.ts
│   │   │   ├── regression.controller.ts
│   │   │   ├── config.controller.ts
│   │   │   └── reports.controller.ts
│   │   ├── services/
│   │   │   ├── pixelDiff.service.ts         # Resemble.js wrapper
│   │   │   ├── visionAnalysis.service.ts    # Groq vision LLM calls
│   │   │   ├── textExtraction.service.ts    # Groq-based text extraction + diff
│   │   │   ├── scoring.service.ts           # Severity scoring engine
│   │   │   ├── reporting.service.ts         # Groq reporting LLM calls
│   │   │   ├── reportRenderer.service.ts    # HTML report generation (Chart.js)
│   │   │   └── screenshotFetch.service.ts   # Orchestrates JIRA + TestRail fetch
│   │   ├── integrations/
│   │   │   ├── jiraClient.ts                # JIRA REST calls (auth, attachment fetch)
│   │   │   ├── testRailClient.ts            # TestRail REST calls (auth, result/attachment fetch)
│   │   │   └── integrationClient.ts         # shared retry/backoff wrapper (mirrors groqClient.ts)
│   │   ├── middleware/
│   │   │   ├── upload.middleware.ts         # multer config, type/size validation
│   │   │   ├── errorHandler.middleware.ts
│   │   │   └── requestLogger.middleware.ts
│   │   ├── lib/
│   │   │   ├── groqClient.ts                # retry/backoff wrapper around Groq SDK
│   │   │   ├── tempFileManager.ts           # temp folder lifecycle, cleanup
│   │   │   └── env.ts                       # typed env accessor with defaults
│   │   ├── config/
│   │   │   └── featureToggles.ts            # reads .env, exposes enabled modes/integrations
│   │   ├── types/
│   │   │   └── index.ts
│   │   ├── app.ts
│   │   └── server.ts
│   ├── .env.example
│   ├── package.json
│   └── tsconfig.json
├── client/                          # React + TypeScript
│   ├── src/
│   │   ├── api/
│   │   │   └── comparisonsApi.ts
│   │   ├── context/
│   │   │   └── ComparisonWizardContext.tsx  # React Context + useState
│   │   ├── components/
│   │   │   ├── MenuSelector/                # Current Sprint vs Regression
│   │   │   ├── ImageUploader/
│   │   │   ├── SubModeSelector/             # pixel / text / hybrid
│   │   │   ├── SourceSwitch/                # Manual Upload vs Fetch from JIRA/TestRail
│   │   │   ├── IntegrationFetchForm/        # JIRA issue key + TestRail case ID inputs
│   │   │   ├── ResultsView/
│   │   │   └── ReportDownloadButton/
│   │   ├── pages/
│   │   │   ├── HomePage.tsx
│   │   │   ├── NewFeaturePage.tsx
│   │   │   ├── RegressionPage.tsx
│   │   │   └── ResultsPage.tsx
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── .env.example
│   ├── package.json
│   └── tsconfig.json
├── README.md
└── Architecture.md                  # this file
```

---

## 4. Environment Configuration

Nothing is hardcoded. Every menu option, sub-menu option, model choice, threshold, integration toggle, and mapping field is configurable via `.env`. Backend and frontend each get their own `.env` file — secrets, model config, and integration credentials never reach the client bundle.

### 4.1 `server/.env.example`

```ini
# --- Server ---
PORT=4000
NODE_ENV=development

# --- Groq API ---
GROQ_API_KEY=

# Vision model: layout/alignment/text/spacing/colour/missing-element analysis
GROQ_VISION_MODEL=meta-llama/llama-4-scout-17b-16e-instruct

# Reporting model: MUST be different from GROQ_VISION_MODEL
GROQ_REPORT_MODEL=llama-3.3-70b-versatile

# Retry/backoff for Groq calls
GROQ_MAX_RETRIES=3
GROQ_BASE_BACKOFF_MS=500
GROQ_TIMEOUT_MS=30000

# --- Feature toggles (menus / sub-modes) ---
ENABLE_NEW_FEATURE_MODE=true
ENABLE_REGRESSION_MODE=true
ENABLE_REGRESSION_PIXEL_SUBMODE=true
ENABLE_REGRESSION_TEXT_SUBMODE=true
ENABLE_REGRESSION_HYBRID_SUBMODE=true

# --- JIRA integration (baseline screenshot source) ---
ENABLE_JIRA_INTEGRATION=true
JIRA_BASE_URL=https://yourorg.atlassian.net
JIRA_EMAIL=
JIRA_API_TOKEN=
# Filename pattern used to locate the baseline image among an issue's attachments
JIRA_BASELINE_ATTACHMENT_PATTERN=baseline*.png
# Custom field on the JIRA issue that stores the mapped TestRail case ID (see req 4.3)
JIRA_TESTRAIL_CASE_FIELD_ID=customfield_10099
JIRA_MAX_RETRIES=3
JIRA_BASE_BACKOFF_MS=500
JIRA_TIMEOUT_MS=15000

# --- TestRail integration (executed/current screenshot source) ---
ENABLE_TESTRAIL_INTEGRATION=true
TESTRAIL_BASE_URL=https://yourorg.testrail.io
TESTRAIL_USERNAME=
TESTRAIL_API_KEY=
# Optional filename filter applied to the latest result's attachments; empty = take first attachment
TESTRAIL_RESULT_ATTACHMENT_PATTERN=
TESTRAIL_MAX_RETRIES=3
TESTRAIL_BASE_BACKOFF_MS=500
TESTRAIL_TIMEOUT_MS=15000

# --- Upload constraints ---
ALLOWED_IMAGE_TYPES=image/png,image/jpeg,image/webp
MAX_UPLOAD_SIZE_MB=10

# --- Temp file lifecycle ---
TEMP_UPLOAD_DIR=./tmp/uploads
TEMP_REPORT_DIR=./tmp/reports
TEMP_FILE_TTL_MINUTES=30

# --- Resemble.js pixel-diff config ---
PIXEL_MISMATCH_THRESHOLD_PERCENT=0.1
PIXEL_IGNORE_ANTIALIASING=true
PIXEL_IGNORE_COLORS=false
# Comma-separated "x,y,width,height" regions to exclude, empty = none
PIXEL_IGNORE_REGIONS=

# --- Hybrid scoring weights (must sum to 1.0; validated at startup) ---
PIXEL_WEIGHT=0.6
TEXT_WEIGHT=0.4

# --- Severity bands (numeric 0-100 -> label) ---
SEVERITY_BAND_LOW_MAX=25
SEVERITY_BAND_MEDIUM_MAX=50
SEVERITY_BAND_HIGH_MAX=75
# anything above HIGH_MAX = Critical

# --- CORS ---
CLIENT_ORIGIN=http://localhost:5173
```

### 4.2 `client/.env.example`

```ini
VITE_API_BASE_URL=http://localhost:4000/api/v1
VITE_MAX_UPLOAD_SIZE_MB=10
VITE_ALLOWED_IMAGE_TYPES=image/png,image/jpeg,image/webp
```

> The frontend never reads feature toggles, integration availability, or credentials from its own `.env` directly — it fetches them from `GET /api/v1/config/modes` at load time, so backend `.env` is the single source of truth for what menus/sub-modes/integrations are enabled. The frontend `.env` only holds client-safe, non-secret values (API base URL, mirrored upload constraints for early client-side validation). JIRA/TestRail credentials live exclusively in `server/.env` and are never returned by any API response.

---

## 5. High-Level Architecture

```
┌─────────────────────┐        HTTPS/REST         ┌──────────────────────────────────────────┐
│   React Client        │ ─────────────────────────▶│   Express API (v1)                        │
│  (upload wizard,      │◀───────────────────────── │                                            │
│   source switch,      │     JSON + report link     │  ┌────────────────────┐                  │
│   results view)       │                             │  │ Upload/Validation  │                  │
└─────────────────────┘                             │  └─────────┬──────────┘                  │
                                                     │            │                              │
                                        ┌────────────┼────────────┼──────────────┐               │
                                        │            ▼            │              │               │
                                        │   screenshotFetch.service.ts (source=integration)       │
                                        │            │            │                               │
                                        │   ┌────────┴────────┐   │                               │
                                        │   ▼                 ▼   │                               │
                                        │ jiraClient.ts   testRailClient.ts                        │
                                        │ (baseline via   (current via latest                      │
                                        │  attachment      result attachment)                      │
                                        │  pattern)                                                 │
                                        │            │            │                               │
                                        │            └─────┬──────┘                               │
                                        │                  ▼                                      │
                                        │            Pixel Diff       Vision Analysis              │
                                        │           (Resemble.js,     (Groq vision                 │
                                        │            deterministic)    model)                       │
                                        │                  │              │                        │
                                        │                  └──────┬───────┘                        │
                                        │                         ▼                                │
                                        │                  Scoring Engine                           │
                                        │               (weighted, .env-driven)                     │
                                        │                         │                                │
                                        │                         ▼                                │
                                        │                 Reporting (Groq                           │
                                        │               reporting model — severity                  │
                                        │               narrative + final score)                    │
                                        │                         │                                │
                                        │                         ▼                                │
                                        │               HTML Report Renderer                        │
                                        │              (table + Chart.js pie)                       │
                                        └─────────────────────────┼────────────────────────────────┘
                                                                   ▼
                                                          tmp/reports/{id}.html
                                                          (served once, TTL cleanup)

                                                           Node.js + Express
```

For `source = manual`, `screenshotFetch.service.ts` is bypassed entirely and uploaded files flow straight into the pixel/vision stage — the rest of the pipeline is identical (mandatory req 4.6).

---

## 6. API Specification

Base path: `/api/v1`. All responses are JSON except the report download endpoint. All endpoints follow a consistent error envelope (§6.6).

### 6.1 `GET /api/v1/health`
Returns service status. No auth.

**200 OK**
```json
{ "status": "ok", "uptimeSeconds": 1234 }
```

### 6.2 `GET /api/v1/config/modes`
Returns which menus/sub-modes/integrations are enabled and current upload constraints, driven entirely by backend `.env`. Frontend uses this to render the wizard dynamically.

**200 OK**
```json
{
  "newFeatureEnabled": true,
  "regressionEnabled": true,
  "regressionSubModes": {
    "pixel": true,
    "text": true,
    "hybrid": true
  },
  "integrations": {
    "jiraEnabled": true,
    "testRailEnabled": true,
    "fetchModeAvailable": true
  },
  "upload": {
    "allowedTypes": ["image/png", "image/jpeg", "image/webp"],
    "maxSizeMb": 10
  }
}
```

`integrations.fetchModeAvailable` is `true` only when both `jiraEnabled` and `testRailEnabled` are `true` — the client's Regression source switch (§9.2) only offers "Fetch from JIRA/TestRail" in that case; otherwise only manual upload is shown.

### 6.3 `POST /api/v1/comparisons/new-feature`
**Current Sprint (New Feature/Page)** flow — no baseline, Vision-LLM checklist only, manual upload only.

- Content-Type: `multipart/form-data`
- Field: `image` (single file, required)

**201 Created**
```json
{
  "comparisonId": "nf_9f2a1c",
  "mode": "new-feature",
  "findings": [
    {
      "category": "layout",
      "description": "Submit button is misaligned relative to the form container",
      "severity": "medium"
    },
    {
      "category": "text",
      "description": "Placeholder text truncated in the email field",
      "severity": "low"
    }
  ],
  "score": {
    "numeric": 42,
    "band": "Medium"
  },
  "reportUrl": "/api/v1/reports/nf_9f2a1c/download"
}
```

Checklist categories always evaluated: `layout`, `alignment`, `text`, `spacing`, `colour`, `missing_or_extra_elements`.

### 6.4 `POST /api/v1/comparisons/regression`
**Regression (Existing Feature/Page)** flow — baseline + current image, one of three sub-modes. Supports two mutually exclusive image sources via the `source` field: `manual` (default) or `integration`.

- Content-Type: `multipart/form-data` when `source=manual`; `application/json` when `source=integration`.

**Manual source fields:**
- `source` = `"manual"` (optional, default)
- `baselineImage` (file, required)
- `currentImage` (file, required)
- `subMode` (string, required): `"pixel"` | `"text"` | `"hybrid"`

**Integration source fields (req 4.1):**
- `source` = `"integration"` (required)
- `jiraIssueKey` (string, required) — e.g. `"PROJ-123"`
- `testRailCaseId` (string, required) — e.g. `"C4521"`
- `subMode` (string, required): `"pixel"` | `"text"` | `"hybrid"`

```json
{
  "source": "integration",
  "jiraIssueKey": "PROJ-123",
  "testRailCaseId": "C4521",
  "subMode": "hybrid"
}
```

When `source = "integration"`, the server (req 4.4):
1. Resolves the baseline screenshot from the JIRA issue's attachments via `jiraClient.ts`.
2. Resolves the current/executed screenshot from the latest TestRail result for the given case via `testRailClient.ts`.
3. Automatically invokes the same comparison pipeline used for manual uploads — no further user action required.

The `jiraIssueKey` and `testRailCaseId` supplied by the tester are independent inputs (the JIRA↔TestRail mapping field, `JIRA_TESTRAIL_CASE_FIELD_ID`, is informational/validation-only in this flow — see §7.3) — both must be provided explicitly.

**201 Created** (hybrid example — response shape is identical regardless of `source`, per req 4.6)
```json
{
  "comparisonId": "rg_7bd44e",
  "mode": "regression",
  "subMode": "hybrid",
  "source": "integration",
  "sourceMeta": {
    "jiraIssueKey": "PROJ-123",
    "testRailCaseId": "C4521"
  },
  "pixelDiff": {
    "mismatchPercent": 3.42,
    "diffImageUrl": "/api/v1/reports/rg_7bd44e/diff-image",
    "flaggedRegions": [
      { "x": 120, "y": 340, "width": 200, "height": 40 }
    ]
  },
  "textDiff": {
    "baselineText": "Welcome back, User",
    "currentText": "Welcome back User",
    "differences": [
      { "type": "missing_punctuation", "detail": "Comma removed after 'back'" }
    ]
  },
  "findings": [
    { "category": "pixel", "description": "Header banner shifted down by ~12px", "severity": "medium" },
    { "category": "text", "description": "Comma removed in greeting copy", "severity": "low" }
  ],
  "score": {
    "pixelScore": 38,
    "textScore": 15,
    "combinedNumeric": 29,
    "band": "Medium",
    "weights": { "pixel": 0.6, "text": 0.4 }
  },
  "reportUrl": "/api/v1/reports/rg_7bd44e/download"
}
```

For `subMode = "pixel"`, the response omits `textDiff` and reports `score.numeric` only (no weighting). For `subMode = "text"`, it omits `pixelDiff` symmetrically. `source` and `sourceMeta` are omitted (or `source: "manual"` with no `sourceMeta`) for manually uploaded runs.

**Partial-fetch failure (req 4.5):** the request fails as a whole with a specific error naming which side failed (see §6.6). No partial/pending comparison record is created — the flow is fully synchronous (§7.3). The tester is expected to resubmit with `source: "manual"` (or a corrected key/ID) for the failed side.

### 6.5 `GET /api/v1/reports/:comparisonId/download`
Streams the generated, self-contained HTML report (`Content-Type: text/html`, `Content-Disposition: attachment`). Valid once per `TEMP_FILE_TTL_MINUTES`; after TTL or first successful download-and-cleanup pass, returns `404`.

### 6.6 Error Envelope
All non-2xx responses share this shape:

```json
{
  "error": {
    "code": "INVALID_FILE_TYPE",
    "message": "Uploaded file must be one of: image/png, image/jpeg, image/webp",
    "statusCode": 400
  }
}
```

| HTTP | Code | Meaning |
|---|---|---|
| 400 | `INVALID_FILE_TYPE` | Upload fails MIME allow-list |
| 400 | `FILE_TOO_LARGE` | Upload exceeds `MAX_UPLOAD_SIZE_MB` |
| 400 | `MISSING_REQUIRED_FIELD` | Required field/file absent |
| 400 | `INVALID_JSON` | Request body is not valid JSON (`source=integration` uses a JSON body) |
| 400 | `INVALID_SUBMODE` | `subMode` not one of pixel/text/hybrid, or that sub-mode is disabled via `.env` |
| 400 | `INVALID_SOURCE` | `source` not one of manual/integration, or `source=integration` requested while `ENABLE_JIRA_INTEGRATION` or `ENABLE_TESTRAIL_INTEGRATION` is `false` |
| 404 | `COMPARISON_NOT_FOUND` | Unknown/expired `comparisonId` |
| 404 | `REPORT_EXPIRED` | Report already downloaded or TTL passed |
| 404 | `JIRA_ISSUE_NOT_FOUND` | `jiraIssueKey` does not exist or is inaccessible with configured credentials |
| 404 | `JIRA_BASELINE_ATTACHMENT_NOT_FOUND` | No attachment on the issue matches `JIRA_BASELINE_ATTACHMENT_PATTERN` |
| 404 | `TESTRAIL_CASE_NOT_FOUND` | `testRailCaseId` does not exist or is inaccessible with configured credentials |
| 404 | `TESTRAIL_RESULT_ATTACHMENT_NOT_FOUND` | The latest result for the case has no matching/eligible attachment |
| 502 | `JIRA_FETCH_FAILED` | JIRA API call failed after retries (network/auth/5xx) |
| 502 | `TESTRAIL_FETCH_FAILED` | TestRail API call failed after retries (network/auth/5xx) |
| 502 | `VISION_MODEL_ERROR` | Groq vision call failed after retries |
| 502 | `REPORT_MODEL_ERROR` | Groq reporting call failed after retries |
| 500 | `INTERNAL_ERROR` | Unhandled server error |

All `JIRA_*` / `TESTRAIL_*` error messages are written to be directly actionable by the tester (e.g., "Could not retrieve executed screenshot from TestRail for case C4521 — please upload manually."), satisfying req 4.5's "meaningful error message + manual fallback" requirement.

---

## 7. Core Workflows

### 7.1 Current Sprint (New Feature) Workflow
1. Client uploads single image → `POST /comparisons/new-feature`.
2. Server validates type/size (`upload.middleware.ts`), writes to `TEMP_UPLOAD_DIR`.
3. `visionAnalysis.service.ts` sends the image to `GROQ_VISION_MODEL` with a structured checklist prompt (layout, alignment, text, spacing, colour, missing/extra elements) and requests structured JSON findings back.
4. `scoring.service.ts` converts findings into a numeric score using per-category severity weights (configurable — see §8.5).
5. `reporting.service.ts` sends findings + score to `GROQ_REPORT_MODEL` to produce a short human-readable summary/narrative and confirm/adjust final severity band.
6. `reportRenderer.service.ts` builds the standalone HTML (table + Chart.js pie) to `TEMP_REPORT_DIR`.
7. Response includes findings, score, and `reportUrl`. Uploaded image and temp working files are scheduled for cleanup per TTL.

### 7.2 Regression Workflow — Manual Upload
1. Client uploads baseline + current image + selects sub-mode, `source=manual` → `POST /comparisons/regression`.
2. Server validates both files, checks the selected `subMode` is enabled via `.env` (`ENABLE_REGRESSION_*_SUBMODE`).
3. Branch by sub-mode:
   - **pixel** → `pixelDiff.service.ts` only (Resemble.js).
   - **text** → `textExtraction.service.ts` only (Groq vision model used purely for text extraction, then string-diff).
   - **hybrid** → both run in parallel, then combined by `scoring.service.ts` using `PIXEL_WEIGHT` / `TEXT_WEIGHT`.
4. `reporting.service.ts` (Groq reporting model, always distinct from the vision model) turns raw diff output into findings + narrative + final severity band.
5. `reportRenderer.service.ts` builds the HTML report.
6. Response returned; temp files cleaned up per TTL/lifecycle.

### 7.3 Regression Workflow — JIRA/TestRail Fetch (req 4.1–4.6)
1. Tester selects the Regression menu, switches the **source** to "Fetch from JIRA/TestRail" (only offered when `config.modes.integrations.fetchModeAvailable` is `true`), enters a **JIRA Issue Key** and a **TestRail Case ID**, and picks a sub-mode.
2. Client calls `POST /comparisons/regression` with `source="integration"`, `jiraIssueKey`, `testRailCaseId`, `subMode`.
3. `regression.controller.ts` checks both `ENABLE_JIRA_INTEGRATION` and `ENABLE_TESTRAIL_INTEGRATION` are `true`; otherwise returns `INVALID_SOURCE`.
4. `screenshotFetch.service.ts` runs both fetches (independently, not dependent on each other):
   - `jiraClient.ts` → `GET /rest/api/2/issue/{jiraIssueKey}` → filters attachments by `JIRA_BASELINE_ATTACHMENT_PATTERN` → downloads the matching attachment into `TEMP_UPLOAD_DIR` as the baseline image. If the field configured by `JIRA_TESTRAIL_CASE_FIELD_ID` is present, its value is compared against the supplied `testRailCaseId` and a non-blocking warning is included in the response if they differ (mapping validation per req 4.3), but the supplied `testRailCaseId` always wins.
   - `testRailClient.ts` → `GET /index.php?/api/v2/get_results_for_case/{run}/{case}` (or case-only lookup, resolving the most recent run/result for that case) → takes the latest result → downloads its attachment (optionally filtered by `TESTRAIL_RESULT_ATTACHMENT_PATTERN`) into `TEMP_UPLOAD_DIR` as the current image.
5. Both calls go through `integrationClient.ts`, which applies exponential backoff (`*_MAX_RETRIES`, `*_BASE_BACKOFF_MS`) and a hard timeout (`*_TIMEOUT_MS`) per attempt, mirroring `groqClient.ts`.
6. **If either fetch fails** after retries (or returns no matching attachment), the entire request fails synchronously with the specific error code from §6.6 (`JIRA_FETCH_FAILED`, `JIRA_BASELINE_ATTACHMENT_NOT_FOUND`, `TESTRAIL_FETCH_FAILED`, `TESTRAIL_RESULT_ATTACHMENT_NOT_FOUND`, etc.). No partial comparison record is created. The tester resubmits — typically switching to manual upload for the failed side (req 4.5).
7. **If both fetches succeed**, execution continues exactly as in §7.2 step 3 onward (pixel/text/hybrid branch, scoring, reporting, HTML render) with no further tester action (req 4.4). The response includes `source: "integration"` and `sourceMeta` (§6.4) but is otherwise byte-for-byte the same shape as a manual-upload response (req 4.6).
8. Fetched images are written to the same `TEMP_UPLOAD_DIR` and cleaned up on the same `TEMP_FILE_TTL_MINUTES` sweep as manually uploaded images — no separate cache or longer-lived storage is introduced.

---

## 8. Module Design

### 8.1 Upload & Validation (`upload.middleware.ts`)
- Uses `multer` with `diskStorage` targeting `TEMP_UPLOAD_DIR`.
- Validates MIME type against `ALLOWED_IMAGE_TYPES` and size against `MAX_UPLOAD_SIZE_MB` — both read from `.env` at request time (no hardcoded literals).
- Rejects with `INVALID_FILE_TYPE` / `FILE_TOO_LARGE` before any processing starts.
- Only invoked when `source=manual`; bypassed for `source=integration`, where `screenshotFetch.service.ts` writes files directly into `TEMP_UPLOAD_DIR` after download, applying the same size/type checks against the downloaded bytes.

### 8.2 Pixel Diff (`pixelDiff.service.ts`)
- Wraps `pixelmatch` (diff algorithm) + `sharp` (decode PNG/JPEG/WebP to raw RGBA, encode the diff PNG) — see §2 for why this replaces Resemble.js. Always deterministic — never calls an LLM.
- Reads `PIXEL_MISMATCH_THRESHOLD_PERCENT` (maps directly to pixelmatch's per-pixel `threshold` option), `PIXEL_IGNORE_ANTIALIASING` (maps to pixelmatch's `includeAA`), `PIXEL_IGNORE_COLORS` (pre-converts both images to luminance-only before diffing), and `PIXEL_IGNORE_REGIONS` from `.env`.
- If the baseline/current canvases differ in size, the current image is resized onto the baseline's dimensions before diffing (`dimensionsMismatched` is reported alongside the result).
- Produces: mismatch percentage, a diff image (served via `GET /reports/:id/diff-image`), and a list of flagged bounding-box regions (mismatched pixels clustered via coarse-grid flood-fill), each individually banded into a severity.

### 8.3 Vision Analysis (`visionAnalysis.service.ts`)
- Calls `GROQ_VISION_MODEL` via `groqClient.ts`.
- New Feature: single-image checklist prompt → structured JSON findings across the six fixed categories.
- Regression (hybrid/text sub-modes): same model reused purely for text/layout extraction — not a second model.

### 8.4 Text Extraction & Diff (`textExtraction.service.ts`)
- Requests structured text extraction (with rough on-image position) from the vision model for both baseline and current images.
- Runs a string/sequence diff (e.g. line- and word-level) between the two extractions.
- Outputs granular differences (`missing_text`, `added_text`, `changed_text`, `missing_punctuation`, etc.).

### 8.5 Scoring Engine (`scoring.service.ts`)
- Pure, deterministic, `.env`-driven — no LLM involved in the arithmetic itself (the reporting model only narrates/confirms).
- New Feature: numeric score = weighted count/severity of findings across the six checklist categories.
- Regression pixel-only: numeric score derived directly from mismatch percentage.
- Regression text-only: numeric score derived from diff density/severity.
- Regression hybrid: `combinedNumeric = round(pixelScore * PIXEL_WEIGHT + textScore * TEXT_WEIGHT)`. Startup validation errors out if `PIXEL_WEIGHT + TEXT_WEIGHT != 1.0`.
- Band mapping: `SEVERITY_BAND_LOW_MAX`, `SEVERITY_BAND_MEDIUM_MAX`, `SEVERITY_BAND_HIGH_MAX` from `.env`; anything above `HIGH_MAX` = `Critical`.
- Scoring logic is identical regardless of image `source` — the engine only ever sees resolved image files, never JIRA/TestRail identifiers.

### 8.6 Reporting (`reporting.service.ts`)
- Calls `GROQ_REPORT_MODEL` — always a distinct model/env-var from `GROQ_VISION_MODEL`.
- Input: raw findings + computed score. Output: human-readable narrative summary, per-finding plain-language explanation, and a confirmation (or bounded adjustment) of the severity label for edge cases.
- This step never re-derives the numeric score from scratch — it narrates and labels what `scoring.service.ts` already computed, keeping scoring deterministic and reproducible.

### 8.7 Report Renderer (`reportRenderer.service.ts`)
- Builds a single, self-contained `.html` file — no external app dependency to view it.
- Includes: findings table (category, description, severity), a Chart.js pie chart (severity distribution or category distribution) loaded from a CDN `<script>` tag, and the overall numeric + band score prominently displayed.
- For integration-sourced runs, also renders `sourceMeta` (JIRA issue key, TestRail case ID) in a header block for traceability.
- Written to `TEMP_REPORT_DIR/{comparisonId}.html`; served once by `GET /reports/:id/download`, then marked for cleanup.

### 8.8 Groq Client Wrapper (`groqClient.ts`)
- Central point for all Groq calls (vision + reporting).
- Exponential backoff: retries up to `GROQ_MAX_RETRIES` with delay `GROQ_BASE_BACKOFF_MS * 2^attempt`; hard timeout `GROQ_TIMEOUT_MS` per attempt.
- On exhaustion, throws a typed error mapped to `VISION_MODEL_ERROR` / `REPORT_MODEL_ERROR`, surfaced to the client with a graceful message rather than a raw stack trace.

### 8.9 Integration Client Wrapper (`integrationClient.ts`)
- Shared retry/backoff HTTP wrapper used by both `jiraClient.ts` and `testRailClient.ts`, structurally mirroring `groqClient.ts` so all outbound calls in the system share one resilience pattern.
- Instantiated per-integration with its own config: JIRA uses `JIRA_MAX_RETRIES` / `JIRA_BASE_BACKOFF_MS` / `JIRA_TIMEOUT_MS`; TestRail uses `TESTRAIL_MAX_RETRIES` / `TESTRAIL_BASE_BACKOFF_MS` / `TESTRAIL_TIMEOUT_MS`.
- On exhaustion, throws a typed error mapped to `JIRA_FETCH_FAILED` / `TESTRAIL_FETCH_FAILED` (distinct from "not found" cases, which are 404s raised directly by the calling client once a successful-but-empty response is parsed).

### 8.10 JIRA Client (`jiraClient.ts`)
- Auth: HTTP Basic using `JIRA_EMAIL` + `JIRA_API_TOKEN` (Atlassian Cloud convention).
- `getIssue(issueKey)` → `GET {JIRA_BASE_URL}/rest/api/2/issue/{issueKey}` — throws `JIRA_ISSUE_NOT_FOUND` on 404.
- `getBaselineAttachment(issueKey)` → lists issue attachments, filters by `JIRA_BASELINE_ATTACHMENT_PATTERN` (glob-style match against filename), downloads the match — throws `JIRA_BASELINE_ATTACHMENT_NOT_FOUND` if none match.
- `getMappedTestRailCaseId(issueKey)` → reads `JIRA_TESTRAIL_CASE_FIELD_ID` off the issue payload, used only for the non-blocking mismatch warning described in §7.3 step 4.
- Never invoked when `ENABLE_JIRA_INTEGRATION=false`; the controller layer rejects `source=integration` before this client is reached.

### 8.11 TestRail Client (`testRailClient.ts`)
- Auth: HTTP Basic using `TESTRAIL_USERNAME` + `TESTRAIL_API_KEY` (TestRail's native REST auth convention).
- `getLatestResultForCase(caseId)` → resolves the most recent test result for the given case (via `get_results_for_case`, taking the newest entry) — throws `TESTRAIL_CASE_NOT_FOUND` on 404.
- `getCurrentAttachment(caseId)` → from the latest result, optionally filters attachments by `TESTRAIL_RESULT_ATTACHMENT_PATTERN` (empty = take the first attachment), downloads it — throws `TESTRAIL_RESULT_ATTACHMENT_NOT_FOUND` if none available.
- Never invoked when `ENABLE_TESTRAIL_INTEGRATION=false`; the controller layer rejects `source=integration` before this client is reached.

### 8.12 Screenshot Fetch Orchestrator (`screenshotFetch.service.ts`)
- Single entry point invoked by `regression.controller.ts` when `source="integration"`.
- Runs the JIRA and TestRail fetches (they are independent of each other and can run concurrently), collects both results, and surfaces the first failure with its specific error code — no partial state is retained or returned (req 4.5, §7.3 step 6).
- On full success, hands the two downloaded file paths to the same pixel/text/hybrid branch used by manual uploads (§7.2 step 3), guaranteeing pipeline parity (req 4.6).

### 8.13 Temp File Manager (`tempFileManager.ts`)
- Creates per-request working directories under `TEMP_UPLOAD_DIR` / `TEMP_REPORT_DIR`, used identically for manual uploads and integration-fetched images.
- Background sweep (interval timer) deletes anything older than `TEMP_FILE_TTL_MINUTES`.
- Deletes a report's file immediately after a successful download response finishes streaming.

---

## 9. Frontend Architecture

### 9.1 State (`ComparisonWizardContext.tsx`)
React Context + `useState`, no external state library. Shape:

```ts
interface WizardState {
  menuPath: 'new-feature' | 'regression' | null;
  subMode: 'pixel' | 'text' | 'hybrid' | null;
  source: 'manual' | 'integration';
  baselineImage: File | null;
  currentImage: File | null;
  jiraIssueKey: string;
  testRailCaseId: string;
  isSubmitting: boolean;
  result: ComparisonResult | null;
  error: ApiError | null;
}
```

### 9.2 Component/Page Flow
1. `HomePage` — fetches `GET /config/modes`; renders only the enabled top-level menu options (Current Sprint / Regression).
2. `NewFeaturePage` — single `ImageUploader`, submits to `/comparisons/new-feature`.
3. `RegressionPage`:
   - `SourceSwitch` — a top-level tab: **Manual Upload** vs **Fetch from JIRA/TestRail**. The fetch tab is rendered only when `config.modes.integrations.fetchModeAvailable` is `true`; otherwise only Manual Upload is shown.
   - Manual tab: `SubModeSelector` (only renders sub-modes enabled per `/config/modes`) + two `ImageUploader`s (baseline, current).
   - Fetch tab: `SubModeSelector` + `IntegrationFetchForm` with two required text inputs — **JIRA Issue Key** and **TestRail Case ID**.
   - Both tabs submit to the same `POST /comparisons/regression`, differing only in `source` and payload shape (multipart vs JSON).
4. `ResultsPage` — renders findings table, score/band, `sourceMeta` (when present), and `ReportDownloadButton` that hits `GET /reports/:id/download`.

### 9.3 Client-Side Validation
- `ImageUploader` pre-checks file type/size against `VITE_ALLOWED_IMAGE_TYPES` / `VITE_MAX_UPLOAD_SIZE_MB` before submitting, as a fast first pass — the backend remains the source of truth and re-validates independently.
- `IntegrationFetchForm` requires both the JIRA Issue Key and TestRail Case ID fields to be non-empty before enabling submit; it performs no format validation beyond that (the backend surfaces `JIRA_ISSUE_NOT_FOUND` / `TESTRAIL_CASE_NOT_FOUND` for invalid values).
- On a `JIRA_*` or `TESTRAIL_*` error response, `ResultsView`/error UI surfaces the server's `message` verbatim and offers a one-click "Switch to Manual Upload" action that pre-selects the Manual tab, satisfying the fallback requirement (req 4.5).

---

## 10. Security & Privacy Notes
- No authentication layer on the tool itself in this scope (internal tool assumption) — flagged as a Non-Goal below if that changes.
- No image or report is retained beyond its TTL; nothing is written to a database — this applies equally to images sourced from JIRA/TestRail.
- Groq, JIRA, and TestRail credentials live only in `server/.env`, never exposed to the client bundle or included in any frontend/API response.
- JIRA calls use HTTP Basic auth (email + API token); TestRail calls use HTTP Basic auth (username + API key). Both are transmitted only over HTTPS to `JIRA_BASE_URL` / `TESTRAIL_BASE_URL`.
- CORS locked to `CLIENT_ORIGIN` from `.env`.
- No JIRA/TestRail credentials, tokens, or raw API responses are ever logged; `requestLogger.middleware.ts` redacts `Authorization` headers.

---

## 11. Non-Goals / Out of Scope (v1)
- No browser automation (no Playwright/Puppeteer) — image sourcing is manual upload or direct JIRA/TestRail attachment retrieval only.
- No user authentication/authorization on the tool itself.
- No persistent report history, dashboard, or database.
- No baseline repository/versioning independent of JIRA — the baseline is always the latest attachment matching the configured pattern at request time.
- No multi-tenant support; a single JIRA site and a single TestRail instance per deployment (one `JIRA_BASE_URL`, one `TESTRAIL_BASE_URL`).
- No asynchronous job queue/polling for integration fetches — the fetch-and-compare flow is fully synchronous (§7.3); very slow JIRA/TestRail responses are bounded only by `JIRA_TIMEOUT_MS` / `TESTRAIL_TIMEOUT_MS`.
- No external mapping-rules file or rules engine — JIRA↔TestRail correlation is a single configurable field (`JIRA_TESTRAIL_CASE_FIELD_ID`) used for validation only, not as the primary lookup path.

---

## 12. Suggested Implementation Phases
1. **Phase 1 — API core:** Express scaffold, `.env` loading/validation, health + config endpoints, upload middleware, temp file manager.
2. **Phase 2 — Pixel diff:** Resemble.js integration end-to-end for regression pixel sub-mode (manual upload), testable via curl/Postman.
3. **Phase 3 — Vision + text:** Groq vision integration for New Feature checklist and regression text-extraction sub-mode.
4. **Phase 4 — Scoring + hybrid:** Scoring engine, hybrid weighting, severity bands.
5. **Phase 5 — Reporting + HTML render:** Groq reporting model integration, Chart.js HTML report generation, download endpoint.
6. **Phase 6 — JIRA/TestRail integration:** `integrationClient.ts`, `jiraClient.ts`, `testRailClient.ts`, `screenshotFetch.service.ts`; extend `POST /comparisons/regression` with `source=integration`; verify partial-failure error paths and TTL parity with manual uploads.
7. **Phase 7 — Frontend:** React wizard (including `SourceSwitch` / `IntegrationFetchForm`) built against the now-complete, tested API.
8. **Phase 8 — Resilience polish:** Retry/backoff verification (Groq and integrations), error envelope consistency, TTL cleanup verification.

Each phase should be demoable independently via API calls before the next phase starts, per the API-first instruction.
