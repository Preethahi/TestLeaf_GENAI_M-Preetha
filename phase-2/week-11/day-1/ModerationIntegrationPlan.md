# Moderation Layer (AI-Shield) Integration Plan

Status: **Implemented and verified** (2026-07-11). This document was previously marked "Implemented" while no code actually existed; that version was replaced with a "Planned" revision grounded in AI-Shield's real source code, and the plan below was then executed phase by phase (backend → backend verification → frontend → frontend verification → docs), with explicit user approval between each phase. See Verification for what was actually exercised, including two gaps that could not be tested in this environment (full route-level `/api/chat` testing, and browser-based UI interaction) — both are called out rather than glossed over.

## Context

`IntegrationPlanprompt.md` requires the Test Case Generator backend to route every request through AI-Shield's moderation API before any content reaches an LLM. AI-Shield is a **real, existing sibling service** at `ai-shield-moderation-layer/ai-shield/` (Express + TypeScript, runs on `PORT=4000`), not an external black box — its combined pipeline endpoint and response contract were read directly from its source:

- `POST {MODERATION_BASE_URL}/api/moderate` (`ai-shield/src/routes/moderate.route.ts`) — accepts `multipart/form-data` or JSON body `{ text?: string, model?: string }` plus an optional `file` field. Runs PII, CII, secret, toxic, injection, and token detectors when `text` is present; runs the file detector when a `file` is present. Returns **HTTP 422** when the overall action is `BLOCK`, **HTTP 200** otherwise.
- Confirmed response shape (`ai-shield/src/types/index.ts`):
  ```json
  {
    "status": "success",
    "action": "BLOCK",
    "detectorResults": [
      {
        "detector": "secret",
        "triggered": true,
        "action": "BLOCK",
        "findings": [{ "type": "openai_key", "value": "sk-abc...", "masked": "[SECRET_REDACTED]", "severity": "high", "position": { "start": 10, "end": 44 } }],
        "message": "Request blocked: secrets detected in payload"
      },
      { "detector": "toxic", "triggered": false, "action": "ALLOW", "findings": [] }
    ],
    "sanitizedContent": "...",
    "originalContent": "...",
    "metadata": { "tokenEstimate": 29, "processingTimeMs": 198, "timestamp": "..." }
  }
  ```
  Overall `action` is `BLOCK > MASK > ALLOW` priority across all detector results (`ai-shield/src/engine/decision.engine.ts#resolveAction`).
- The file detector (`ai-shield/src/detectors/file.detector.ts`) returns the **same** `DetectorResult` shape as text detectors (it flows through the same `DecisionEngine.buildResponse()`), so `/api/moderate` is a single, unified endpoint for both text and file checks — no separate `/api/detect/file` client is needed.
- **Known constraint**: `ai-shield/rules/file-rules.json` allowlists only `.txt/.pdf/.docx/.csv/.json/.md/.xlsx` (`defaultAction: "BLOCK"` for anything else, `maxFileSizeBytes: 5242880`). It does **not** include image types. Screenshots (PNG/GIF), a first-class feature of the generator's `/api/upload` vision path, must **not** be sent through this file check, or every screenshot upload would be blocked. See Design Decision 5 below.

## Design decisions

1. **Gate strictly on `action === "BLOCK"`.** `ALLOW` and `MASK` both proceed with the original, unmodified text — `sanitizedContent` is not substituted in place of the original before sending to the LLM. Reconstructing/using AI-Shield's masked text is a deliberate non-goal here (see Open Follow-ups); the literal requirement only mandates stopping the LLM call on block.
2. **Never forward raw `value` fields.** Only `type`, `masked`, and `severity` from each `Finding` are surfaced to the frontend or logged — the raw matched secret/PII value is dropped server-side immediately after the AI-Shield response is parsed, so a blocked request's error payload can't itself leak the sensitive content that triggered it.
3. **Fail closed.** If AI-Shield is unreachable, times out, or returns an unparseable/unexpected body, the request is treated as blocked with a distinct "moderation service unavailable" message (HTTP 503) — never silently passed through to the LLM.
4. **`MODERATION_ENABLED` toggle lives in `backend/.env` only**, server-side single source of truth. When `false`, the backend skips the AI-Shield call entirely (zero added latency) and treats everything as allowed. No frontend toggle or frontend `.env` change.
5. **Moderation gates three routes**: `POST /api/chat` and `POST /api/generate-testcases` (text, before the LLM call) and `POST /api/upload` (uploaded **documents only** — PDF/DOCX/XLSX — before parsing). **Screenshots (PNG/GIF) skip file-moderation entirely** — they aren't in AI-Shield's file allowlist and would always be blocked otherwise. This is a documented, intentional gap: image *content* is never sent to AI-Shield in this integration (the vision-model path has no AI-Shield equivalent today); only a later `/api/chat` turn's accompanying text (if any) is moderated.
6. **Status codes**: `422` for a moderation block (mirrors AI-Shield's own convention for `action === "BLOCK"`), `503` for moderation-service-unavailable. Both are distinguishable from the existing generic `502` (LLM/chain failure) already used by these routes.

## Backend changes

**`backend/.env`** — new block:
```
# --- Moderation (AI-Shield) ---
MODERATION_ENABLED=false
MODERATION_BASE_URL=http://localhost:4000
MODERATION_TIMEOUT_MS=5000
```
Defaults to `false`: no AI-Shield instance runs automatically alongside this repo, and fail-closed semantics mean defaulting to `true` would block every request until AI-Shield is confirmed reachable.

**`backend/src/config/env.ts`**
- Add an `optionalBool(name, fallback)` helper (parallel to the existing `required`/`optional` string helpers).
- Add `moderationEnabled: optionalBool("MODERATION_ENABLED", false)`, `moderationBaseUrl: optional("MODERATION_BASE_URL", "http://localhost:4000")`, `moderationTimeoutMs: Number(optional("MODERATION_TIMEOUT_MS", "5000"))`.
- No changes to `validateEnv()` — all three have safe defaults, so a missing `.env` block doesn't break startup.

**`backend/src/services/moderationService.ts`** (new file — new `services/` folder, sibling to `factories/`, `tools/`, `memory/`)
- Types (colocated, matching how `chains/testCaseChain.ts` colocates its Zod schema/type with the functions that use it): `ModerationFinding { type: string; masked?: string; severity?: string }` (raw `value` intentionally omitted per Decision 2), `ModerationDetectorResult { detector: string; triggered: boolean; action: "ALLOW" | "MASK" | "BLOCK"; findings: ModerationFinding[]; message?: string }`.
- `class ModerationBlockedError extends Error { detectors: ModerationDetectorResult[] }`.
- `class ModerationUnavailableError extends Error {}`.
- `callAiShield(payload: { text?: string; model?: string; file?: { buffer: Buffer; filename: string; mimeType: string } })` (internal) — POSTs to `${env.moderationBaseUrl}/api/moderate`, JSON body when no file, multipart `FormData`/`Blob` when a file is present, using `AbortSignal.timeout(env.moderationTimeoutMs)` (native `fetch`, no new dependency — Node 18+/`@types/node ^22` already supports this, matching the repo's existing lack of an HTTP client library like axios). Any network error, timeout, non-200/422 status, or response that doesn't parse into the expected shape throws `ModerationUnavailableError` with a descriptive message; this is logged via `logError("Moderation", ...)`.
- `moderateText(text: string, model?: string): Promise<void>` — no-op immediately if `!env.moderationEnabled`. Otherwise calls `callAiShield({ text, model })`; if `action === "BLOCK"`, throws `ModerationBlockedError` with sanitized (`value`-stripped) detector results; `ALLOW`/`MASK` return normally (original text is used downstream, unchanged).
- `moderateFile(buffer: Buffer, filename: string, mimeType: string): Promise<void>` — same no-op gate; calls `callAiShield({ file: { buffer, filename, mimeType } })`; same block/pass semantics.
- Both log the outcome via `logInfo("Moderation", ...)` (action + which detectors triggered, no raw values) for observability, consistent with existing route/chain logging via `utils/logger.ts`.

**`backend/src/routes/chatRoute.ts`**
- After existing `domain`/`message`/`imageDataUrl` validation, before calling `chatGenerateTestCases()`: `await moderateText(message)` wrapped so `ModerationBlockedError` → `res.status(422).json({ blocked: true, error: "Your message was blocked by the content moderation layer...", detectors: err.detectors })`, and `ModerationUnavailableError` → `res.status(503).json({ error: err.message })`. Both checked before the existing generic `catch` that returns `502`.
- `message` is always required and non-empty by existing validation, so every `/api/chat` call has text to moderate even when a screenshot is attached — image *binary* content itself is still never sent to AI-Shield (Decision 5's gap applies to the image bytes, not the accompanying text).

**`backend/src/routes/testCaseRoute.ts`**
- Text branch (`hasText`): `await moderateText(requirementsText)` before `generateTestCases()`, same error branches as above.
- Image branch (`hasImage`): `additionalContext` is optional. If present, moderate it (`await moderateText(additionalContext)`); if absent, there is no text to check and the request proceeds — the image itself is not moderated (same documented gap as chatRoute).

**`backend/src/routes/uploadRoute.ts`**
- After `detectKind()` and `validateFileSize()` succeed, **only when `kind !== "image"`** (i.e., `pdf`/`docx`/`excel`): `await moderateFile(file.buffer, file.originalname, file.mimetype)` before calling the matching parser. Same `422`/`503` error branches. Images skip this call entirely per Decision 5.

**`backend/src/chains/testCaseChain.ts`** — no change. Moderation is enforced at the route layer (request validation is already owned by the route files, not the chain), keeping the chain focused on RAG + prompt + model orchestration.

**`backend/src/app.ts`** — add `ModerationBlockedError`/`ModerationUnavailableError` branches to the existing tail error-handling middleware (alongside the current `MulterError`/`UnsupportedFileTypeError` branches) as a defense-in-depth safety net, in case a future call site forgets to catch them locally.

## Frontend changes (implemented only after backend changes are verified — API before UI, per the mandatory instruction)

**`frontend/src/services/api.ts`**
- Add `ModerationDetector` type (mirroring the backend's sanitized shape: `detector`, `triggered`, `message?`, `findings: { type, masked?, severity? }[]`).
- Extend `ApiError` with an optional `detectors?: ModerationDetector[]`, or add a `ModerationBlockedError extends ApiError` subclass.
- Update `handleResponse<T>()` to detect `body.blocked === true` and throw the moderation-aware error instead of a generic `ApiError`.
- Update `uploadFile()`'s `XMLHttpRequest` `onload` handler the same way, since it has its own separate response-parsing path (not routed through `handleResponse`).

**`frontend/src/components/ChatWindow.tsx`**
- Extend `ChatMessage` with `role: "user" | "assistant" | "error" | "blocked"` and an optional `detectors` field.
- In `handleSend()`'s catch block, branch on the moderation error type: push `{ role: "blocked", text: err.message, detectors: err.detectors }` and fire a distinct toast, instead of the generic error bubble. Keep the existing input/attachment-restoration behavior for both paths.
- In the render loop, add a `"blocked"` branch rendering a new notice (see below); suppress the avatar for it the same way it's already suppressed for `"error"`.

**`frontend/src/components/ModerationBlockedNotice.tsx`** (new file, mirroring how `TestCaseResultViewer.tsx` is its own component rather than inline JSX)
- Renders a headline ("Blocked by content moderation"), user-friendly guidance text, and the list of triggered detectors (name + message + masked findings), reusing existing card/badge visual language from `App.css`/`index.css` rather than inventing new component styling.

**`frontend/src/components/FileUpload.tsx`** — no structural change; its existing error-chip display already surfaces `ApiError.message`, which will now include the moderation block reason for document uploads.

**`frontend/src/App.css` / `index.css`** — add `.chat-bubble--blocked`, `.moderation-detector-list`/`-item`/`-name`/`-message`/`-findings`, reusing the existing `--color-danger*` design tokens.

No frontend `.env` change — moderation is enforced/toggled entirely server-side (Decision 4).

## Documentation updates

- **`README.md`**: add an `AI-Shield` external node + edges (from `ChatRoute`/`TCRoute`/`UploadRoute`) to the mermaid diagram; add `moderationService.ts` to the backend file-by-file table; note moderation gating in the API Endpoints table; add a moderation step + block short-circuit to the relevant request-flow write-ups (§6); add `MODERATION_ENABLED`/`MODERATION_BASE_URL`/`MODERATION_TIMEOUT_MS` to §7 Environment Variables.
- **`INSTALL.md`**: add a step noting that when `MODERATION_ENABLED=true`, an AI-Shield instance must be running and reachable at `MODERATION_BASE_URL` (e.g. `cd ../../ai-shield-moderation-layer/ai-shield && npm install && npm run dev`, port 4000) before starting the generator's backend — fail-closed means an unreachable AI-Shield blocks every chat/generate/document-upload request.

## Sequencing

1. Backend: env vars → `moderationService.ts` → `chatRoute.ts` → `testCaseRoute.ts` → `uploadRoute.ts` → `app.ts` safety net. Fully testable via curl/Postman against the running AI-Shield instance, independent of any frontend change.
2. Verify backend (see Verification below).
3. Frontend: `api.ts` → `ChatWindow.tsx` → `ModerationBlockedNotice.tsx` → CSS.
4. Docs: `README.md`, `INSTALL.md`.

This satisfies "incorporate API changes first, then proceed with UI" literally: frontend work consumes a contract (`blocked`/`detectors` in the JSON body) that only exists once the backend half is implemented and verified.

## Verification

With AI-Shield running (`cd ai-shield-moderation-layer/ai-shield && npm run dev`, port 4000) and the generator backend running with `MODERATION_ENABLED=true`:

1. `MODERATION_ENABLED=false` (default) → chat/generate/upload behave exactly as today; no AI-Shield calls (confirm via backend logs — no `"Moderation"` log lines).
2. Clean text via `/api/chat` → generation proceeds normally; a `"Moderation"` log line shows `ALLOW`.
3. Text matching a BLOCK detector (e.g. the Postman collection's "Ignore previous instructions..." injection sample, or the "mixed threats" combined-pipeline sample) via `/api/chat` and `/api/generate-testcases` → `422` response with `detectors`; confirm via logs that no `"Model"` log line from `modelFactory.ts` fires (LLM never invoked).
4. A `.exe`-renamed or oversized document via `/api/upload` → `422`, parser never runs.
5. A PNG/GIF screenshot via `/api/upload` → succeeds exactly as today, confirming the image-skip path (Decision 5) works and doesn't regress the existing vision feature.
6. Stop AI-Shield, retry any of the above → `503` "moderation service unavailable", no silent pass-through to the LLM.
7. Frontend: repeat scenario 3/4 through the UI — confirm the blocked chat bubble/upload chip shows the reason and detector details, and a normal successful generation still renders through `TestCaseResultViewer` unchanged.

## Open follow-ups (explicitly out of scope for this pass)

- `MASK`-action findings are not redacted before reaching the LLM (Decision 1) — a future enhancement could substitute AI-Shield's `sanitizedContent` for the original text on `MASK`.
- Image *content* is never moderated (Decision 5) — closing this gap requires either an AI-Shield rules change (allowlisting image types) or a different detector path for vision requests; both are outside this repo.
- No automated tests are added in this pass; verification is manual (curl/Postman + UI walkthrough) per the Verification section above.
