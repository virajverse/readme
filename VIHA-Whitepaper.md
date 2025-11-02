# VIHA — Viraj’s Intelligent Hacker Agent

Version: 1.0.0
Owner: Fearless, Bihar (India)
License: Proprietary (Closed‑source)

## Executive Summary

VIHA is a local, privacy‑first, India‑proud AI desktop agent that brings a transparent holographic HUD, voice‑first interaction (online Web Speech + offline Vosk Hindi), and an adaptive multi‑model AI committee into a single production‑grade product. VIHA runs primarily on the user’s Windows 10/11 machine, keeps sensitive data local by default (no telemetry, no auto‑send), and provides powerful automation with Brave tab reuse, WhatsApp draft creation, Gmail ToS‑gated flows, and a secure vault (AES‑256‑GCM with Windows DPAPI‑protected key).

This whitepaper explains VIHA’s architecture, security and privacy posture, AI model routing strategy, visualization pipeline, RAG, packaging and CI/CD, and operational guidance. It is intended for engineering and security reviewers, CTOs/CISOs, and enterprise evaluators.

---

## 1. Core Principles

- Privacy‑first by design.
- India‑proud voice and UX (Hinglish/Hindi native experiences, ₹ currency awareness).
- Local‑first operation with robust offline fallback.
- No auto‑send; user confirms all outbound automations (WhatsApp/Gmail).
- Production‑ready quality: tests, CI on Windows, signed installers, clear observability.

Key outcomes:
- Zero telemetry by default. No external uploads without explicit user action.
- Adaptive reliability in low/unstable networks.
- Enterprise‑grade packaging with code signing hooks and EULA.

---

## 2. System Architecture Overview

Components:
- Electron desktop app (interfaces/desktop):
  - Transparent, always‑on‑top HUD (Three.js‑based).
  - WebSocket server for real‑time visualization commands (default port ~17321; auto‑increment with retries).
  - Voice capture and offline Vosk pipeline.
  - Local automations (Brave tab reuse, window focus, WhatsApp draft).
  - Data ingestion (folder scan with OCR, index creation) and live data feeds (CSV/JSON → 3D charts).
- Backend (backend):
  - FastAPI app with token‑protected endpoints, rate limiting, and CORS for Electron.
  - Adaptive AI model router that selects cloud/local models depending on intent and connectivity.
  - RAG with ChromaDB persistent client and context safety cap.
  - Health endpoint and file‑based logging.

Security Boundaries:
- Desktop app runs with local privileges; all persistent data under `%AppData%/Roaming/<App>/VIHA`.
- Backend listens locally; authenticated via `X‑VIHA‑Token` for sensitive routes.
- Vault encrypts sensitive drafts with AES‑256‑GCM and DPAPI‑protected key.

Data Directories & Key Paths:
- `%AppData%/Roaming/<App>/VIHA/` → base data dir set at runtime.
  - logs/viha.log
  - vault/ (encrypted records)
  - feeds/ (CSV/JSON live data inputs)
  - learning/ (feedback jsonl)
  - contacts.json (local address book)
- `data/models/vosk/` packaged to `resources/models/vosk/` for production builds (extraResources).

Environment Variables:
- `VIHA_TOKEN` (backend auth token; must not be default).
- `VIHA_WS_PORT` (WS port for HUD runtime; exported by desktop at boot).
- `VIHA_API_BASE` (renderer‑side base URL via preload; no hardcoded localhost in UI).
- `VIHA_E2E_EXE` (CI packaged exe path for Electron E2E).
- `CSC_LINK`, `CSC_KEY_PASSWORD` (code signing in CI on tagged builds).
- `VIHA_SKIP_VOSK` (test‑mode bypass for Vosk init in CI/E2E).

---

## 3. Desktop HUD and Visualization Pipeline

### 3.1 Transparent HUD Window
- Frameless, transparent `BrowserWindow` with background `#00000000` and `alwaysOnTop: true`.
- Click‑through "pin‑through" mode toggled by:
  - Voice command: “VIHA, HUD ko pin‑through karo”.
  - Shortcut: Ctrl+Shift+P.
- Implementation: `setIgnoreMouseEvents(true, { forward: true })` when enabled.

### 3.2 WebSocket Bridge
- Local WebSocket server started at port 17321, with +1 retries up to 5 attempts.
- HUD connects with `?ws=<port>` query parameter.
- Messages: `{ type: 'vis', cmd: <VisualizationCommand> }`.

Visualization Commands (sanitized upstream):
- `action`: create | update | remove.
- `type`: bar | bar3d | donut | particles.
- Optional props: `id`, `values`, `percent`, `color`, `scale`, `label`.

### 3.3 Visualization Engine
- Three.js primitives:
  - `bar3d`: columnar meshes for numeric arrays.
  - `donut`: ring geometry showing `percent`.
  - `particles`: ambient flow/field.
- Engine keeps a registry and applies updates/removals by `id`.

### 3.4 Live Data Feeds (CSV/JSON)
- Watches `%AppData%/Roaming/<App>/VIHA/feeds/`.
- On changes:
  - Parse numbers from `.csv` or `.json`.
  - Broadcast `create` (first time) then `update` for same id (file stem).
- Common scenario: `sales.csv` auto‑updates a 3D bar chart with id `sales`.

---

## 4. Voice Stack (Online + Offline)

### 4.1 Microphone and Permissions
- Electron `session.setPermissionRequestHandler` allows mic capture at runtime.
- Renderer uses Web Speech API where available; else offline Vosk pipeline.

### 4.2 Offline Vosk (Hindi)
- Models bundled into `resources/models/vosk` in production builds.
- IPC: `vosk-init`, `vosk-chunk`, `vosk-stop`.
- Audio resampling to 16KHz, 16‑bit PCM chunks streamed to recognizer.
- CI bypass: `VIHA_SKIP_VOSK=1` returns `{ ok: true, skipped: true }` for E2E.

### 4.3 Hinglish/Hindi Time Parsing
- Natural phrases supported: “kal”, “parso”, “shaam”, “baje”, with robust defaults.
- Outputs ISO 8601 strings; used for calendar draft creation.

### 4.4 TTS Fallback Chain
- Prefers Hindi voices; fallback to Microsoft Priya / Heera / en‑IN variants.
- Short, respectful Hinglish by default in offline/slow modes.

---

## 5. Adaptive AI Committee (Model Router)

### 5.1 Routing Signals
- Vision tasks → vision‑capable model (e.g., Gemini vision).
- Strategy tasks → large reasoning model (e.g., Bytez/Llama/Mistral endpoint).
- Hindi/Hinglish or ₹ currency → preferential local routing.

### 5.2 Slow‑Net Probe and Offline Mode
- Probe: `https://www.gstatic.com/generate_204` with 2s timeout and `Connection: close`.
- If slow/offline, replies start with a Hinglish prefix clearly communicating local mode.

### 5.3 Learned Hindi Rule
- After repeated negative feedback in Hindi (e.g., 3 thumbs‑down), enforce local‑only Qwen until quality improves.

### 5.4 Provider Hardening
- All external HTTP calls set `Connection: close`.
- Automatic retries (max 2 retries) on transport timeouts.
- Memory profiling around local model loading (psutil RSS before/after).

---

## 6. Privacy & Security

### 6.1 Vault Encryption
- AES‑256‑GCM; key stored at data/secret.key, protected by Windows DPAPI.
- Encrypted JSON files suffixed `.viha.json.aesgcm`.
- Vault index encrypted with AAD `b'vault_index'` and plaintext migration path.

### 6.2 Outbound Automations With Consent
- WhatsApp: URL drafts only (no auto‑send). User sends manually.
- Gmail: ToS disclaimer must be accepted once; actions performed via local browser flows.

### 6.3 Backend Controls
- Token‑based auth for mutations; enforced at startup (`VIHA_TOKEN` cannot be default).
- Rate limiting via SlowAPI; 429 returns `Retry‑After` and `X‑RateLimit‑Remaining: 0`.
- CORS allows `null` origin for Electron `file://` contexts.

### 6.4 Threat Model (Selected)
- Data exfiltration: mitigated by default local‑only operation and no telemetry.
- Prompt injection: sanitize intent JSON; allowlist types and bounded sizes.
- Abuse of IPC: specific channels only; minimal surface with explicit handlers.

---

## 7. Retrieval Augmented Generation (RAG)

### 7.1 Desktop Scan Pipeline
- Multi‑format extraction: TXT/MD/CSV/JSON, PDF (pdf‑parse), DOCX (mammoth), OCR (tesseract.js with eng+hin fallback).
- Parallel worker pool (4 workers) to index files; output to `data/index.json`.

### 7.2 ChromaDB Client
- `PersistentClient` singleton per worker process with thread lock to avoid double init.
- Collections carry `viha_version` metadata and recreate safely on mismatch.

### 7.3 Reindex & Query APIs
- Reindex reads desktop index and adds docs (with optional embeddings via fastembed).
- Query returns top 3 results.

### 7.4 Aggregate Context Safety (30K Cap)
- Server truncates total returned text to 30,000 characters across hits.
- Prevents memory pressure or overly long downstream prompts.

---

## 8. Desktop Automation & App Reuse

### 8.1 Brave Reuse (Remote Debug 9222)
- Connects to existing Brave with Puppeteer Core; no Chromium downloads (AV‑friendly).
- Finds existing tab by URL substring; else opens a new one.

### 8.2 Window Focus via PowerShell
- Uses `user32.dll` `SetForegroundWindow` with Add‑Type C# shim.
- Adds `-ExecutionPolicy Bypass` for restrictive environments.

### 8.3 WhatsApp Drafts
- Normalizes contact names/phone numbers; composes `wa.me` or Web WhatsApp URL.
- Strictly never auto‑sends.

### 8.4 Live Feeds → Visualization
- File watcher broadcasts `bar3d` create/update commands for numeric data.

---

## 9. Performance Characteristics

- Fast cold‑start with on‑demand model loading.
- Llama local model cached as a singleton to avoid repeated loads.
- 4‑worker extractor yields responsive indexing across many files.
- WS bus decouples visuals from compute; simple, reliable protocol.

---

## 10. Packaging, Installation, and CI/CD

### 10.1 Electron Builder (NSIS)
- Windows NSIS installer with EULA (license.txt).
- Extra resources bundle Vosk models and default contacts.

### 10.2 Code Signing
- CI hooks for code signing on tagged builds (`CSC_LINK`, `CSC_KEY_PASSWORD`).

### 10.3 CI on Windows Runners
- Runs backend pytest, desktop Jest, Playwright (HUD/voice) and Electron E2E.
- Uploads Playwright HTML reports, traces, videos.

---

## 11. Observability

- Desktop logs to `%AppData%/Roaming/<App>/VIHA/logs/viha.log` and captures uncaught exceptions.
- Backend logs to `backend/data/logs/viha.log` and streams to console as fallback.
- `/health` exposes memory RSS and disk free bytes.

---

## 12. Windows‑Specific Hardening

- Paths managed via `path.join`/`Path()` to avoid separator issues.
- Unicode path safety acknowledged (e.g., `D:\Taliyo टेक\`); string handling avoids lossy conversions.
- No `robotjs`; reduces AV false positives.

---

## 13. API Contracts & Validation

- Strict Pydantic models on email/calendar; bounds on subject/body/ISO date.
- Intent JSON sanitization with allowlists and truncation.
- Headers never expose public tokens in web context.

---

## 14. Operational Playbooks

- First run: check logs, accept Gmail ToS if needed, verify Vosk models present, set `VIHA_TOKEN`.
- Offline mode drill: block gstatic, verify Hinglish offline prefix.
- Restore vault: copy `data/vault` (encrypted), rehydrate with same machine profile.

---

## 15. Business & Compliance

- Closed‑source, commercial license.
- Privacy stance: local‑first, minimal data movement, user consent gates.
- India data residency compatible (self‑host backend locally).

---

## 16. Roadmap (Selected)

- Live vision overlays (YOLO + Gemini) in HUD.
- Android ADB automation and screen mirroring.
- Multi‑user encrypted sharing (per‑record keys).

---

## 17. Appendices

### 17.1 Ports and Env Vars
- WebSocket HUD bus: 17321 (+ retries).
- Backend default: 8000 (configurable); `VIHA_API_BASE` controls UI calls.

### 17.2 Directory Map
- `%AppData%/Roaming/<App>/VIHA/` → logs/, vault/, feeds/, learning/, contacts.json
- `resources/models/vosk` → offline speech models.

### 17.3 Test Matrix Overview
- Offline/slow‑net: probe forced → offline prefix.
- Vosk IPC: `VIHA_SKIP_VOSK=1` test pass.
- HUD: transparency, pin‑through, WS vis pipeline.
- RAG: reindex, query with 30K cap.

---

## Conclusion

VIHA demonstrates how a modern, India‑proud AI agent can deliver enterprise‑grade reliability and privacy on Windows desktops: a transparent holographic HUD, voice‑first interaction with robust offline Hindi, an adaptive model router that respects user context and connectivity, and a security posture centered on user consent and local encryption. The codebase is hardened for packaging and CI on Windows, and the automation stack prioritizes safety, reusability, and AV‑friendly operation.

This whitepaper is a living document. Future versions will deepen performance studies, extend the threat model, and add formal verification of RAG truncation and model routing safety under adversarial inputs.

---

## 18. Architecture Deep Dive

- **[Boot sequence]**
  - App acquires single‑instance lock to prevent duplicate sessions.
  - Data directory `%AppData%/Roaming/<App>/VIHA` is created if missing.
  - Logging targets are prepared under `logs/viha.log` for both desktop and backend.
  - WebSocket server attempts to bind to 17321, incrementing the port up to 5 retries.
  - HUD window is created transparent, frameless, and always‑on‑top; it auto‑connects to the WS via `?ws=<port>`.

- **[Data flows]**
  - Voice capture → (online Web Speech or offline Vosk) → text intent → local model routing (if needed) → sanitized visualization commands → WS broadcast → Three.js renderer.
  - Desktop scan/OCR → `data/index.json` → backend `rag/reindex` → Chroma collection; queries capped at 3 results with 30K aggregate text limit.
  - Automation intents → Brave reuse (RD port 9222), WhatsApp drafts, PowerShell focus, with consent gates and safe fallbacks.

- **[IPC surfaces]**
  - `scan-folder`, `launch-app`, `send-gmail`, `open-gmail-modal`, `open-about`.
  - `vosk-init`, `vosk-chunk`, `vosk-stop` (with `VIHA_SKIP_VOSK=1` bypass for CI).
  - `open-url`, `open-url-brave`, `open-whatsapp`, `focus-window-title`.
  - `visualize-cmd` (renderer → main → WS → HUD).
  - `hud-pin-through`, `hud-pin-state`, `get-viha-data-dir`, `getApiBase` (via preload).

- **[Process & isolation]**
  - Backend FastAPI runs separately (local host), enforcing token and rate limits.
  - Desktop stays local; all privileged automation occurs via Electron main, not untrusted webpages.

## 19. Model Router Internals

- **[Signal extraction]**
  - Vision keywords (image/photo/vision/ocr) → vision‑capable model.
  - Strategy heuristics detect long‑form reasoning needs.
  - Hindi/₹ detection favors local Qwen; learned rule escalates to local‑only after repeated thumbs‑down.

- **[Network health]**
  - `generate_204` probe at 2s with `Connection: close`; any exception or slow response triggers offline mode.
  - Offline mode prepends a respectful Hinglish prefix clarifying local fallback.

- **[Provider hardening]**
  - All calls include `Connection: close`, and retry up to 2 times on network errors.
  - Response handling avoids brittle assumptions; errors degrade gracefully to local model.

- **[Local model lifecycle]**
  - Singleton load; RSS logged before/after for observability.
  - Path normalization via `Path()` ensures Windows compatibility.

## 20. Voice Processing & TTS Details

- **[Capture]**
  - Web Speech API preferred when present for latency and convenience.
  - Offline: microphone → `AudioContext` → resample to 16kHz PCM16 → chunk to Vosk recognizer via IPC.

- **[Buffering]**
  - `ScriptProcessor` buffers (4096 frames) are converted to Int16 LE; ratio‑based resampling preserves intelligibility with low CPU.

- **[TTS voice fallback]**
  - Selector scans for Hindi voices first; falls back to Microsoft Priya/Heera or en‑IN English; rate/pitch tuned for clarity.

- **[Safety]**
  - Mic permission gate uses Electron permission handler; exceptions are logged and non‑fatal.

## 21. RAG Implementation Deep Notes

- **[Extraction]**
  - TXT/MD/CSV/JSON read plainly; PDF via `pdf-parse`; DOCX via `mammoth`; images via `tesseract.js` (eng+hin) when available.
  - OCR worker created lazily and torn down on quit to free resources.

- **[Index & schema]**
  - `index.json` stores doc path, ext, and truncated text; counts and folder stats aid observability.

- **[Chroma client]**
  - Thread lock guards singleton init; collection metadata includes `viha_version` to allow safe resets.

- **[Context safety]**
  - Query aggregates up to 30,000 characters across hits; excess is truncated per‑hit while preserving metadata.

## 22. Security Threat Model & Mitigations

- **[Surfaces]**
  - IPC handlers, local HTTP endpoints, WS channel, automation bridges.

- **[Mitigations]**
  - Token auth for write ops; SlowAPI throttling with explicit 429 headers.
  - Intent JSON allowlist and field caps; strict types for email/calendar via Pydantic.
  - Vault encryption with DPAPI‑protected key; encrypted index with AAD.

- **[Outbound policy]**
  - Gmail ToS gate and local‑browser flow; WhatsApp drafts only; never auto‑send.

- **[Supply chain]**
  - Puppeteer‑core (no Chromium download) to minimize AV flags; no `robotjs`.

## 23. Windows Integration & Reliability

- **[PowerShell focus]**
  - Add‑Type C# shim for `SetForegroundWindow`; `-ExecutionPolicy Bypass` to tolerate restrictive environments.

- **[Unicode & paths]**
  - Path joins avoid backslash pitfalls; tested with mixed‑script directories (e.g., Devanagari file names).

- **[AV considerations]**
  - Avoid injecting input or DLLs; rely on OS automation surfaces only.

- **[Packaging]**
  - NSIS installer with license; Vosk packaged as extraResources; code signing via CI secrets when available.

## 24. Testing & Quality Playbooks

- **[Electron Playwright E2E]**
  - Validates transparency, WS bridge, reuse IPC, pin‑through toggle, CSV/JSON feeds, and Vosk IPC (with `VIHA_SKIP_VOSK`).

- **[Backend pytest]**
  - Covers rate limits, health, learned routing, and offline behavior.

- **[Reports]**
  - Traces/videos/screenshots on failure; HTML reports uploaded by CI.

## 25. Operations & Support

- **[First‑run checklist]**
  - Set `VIHA_TOKEN`, confirm Vosk models, accept Gmail ToS if using Gmail draft flows, verify feeds and contacts.

- **[Troubleshooting]**
  - Slow network → offline prefix appears; WS failures auto‑reconnect; Brave reuse tolerates missing 9222.

- **[Upgrades]**
  - Preserve `%AppData%/Roaming/<App>/VIHA/` intact; NSIS one‑click upgrades retain data.

## 26. Extended Roadmap Notes

- **[Live vision HUD]**
  - Optional YOLO/Ultralytics module feeding detections to WS overlays; privacy kept local.

- **[Android ADB]**
  - Voice‑driven device control and screen mirroring; explicit consent gates.

- **[Encrypted sharing]**
  - Per‑record keys and out‑of‑band passphrase exchange; zero‑knowledge sharing designs.

---

## Appendix A — Full API Reference

This appendix documents core backend endpoints with method, path, auth, request/response shapes, and rate‑limit behavior. All JSON fields are strict; additional/unknown fields are ignored.

- **Health**
  - Method: GET
  - Path: `/health`
  - Auth: None
  - Response: `{ ok: boolean, memory_rss: number|null, disk_free: number|null }`
  - Notes: `memory_rss` in bytes; `disk_free` for VIHA data mount.

- **Chat**
  - Method: POST
  - Path: `/v1/chat`
  - Auth: `X‑VIHA‑Token`
  - Rate limit: 60/minute; 429: `Retry‑After: 60`, `X‑RateLimit‑Remaining: 0`
  - Request: `{ message: string (2..2000) }`
  - Response: `{ ok: true, reply: string } | { ok: false, error }`

- **Intent Visualize**
  - Method: POST
  - Path: `/v1/intent/visualize`
  - Auth: `X‑VIHA‑Token`
  - Rate limit: 60/minute
  - Request: `{ message: string }`
  - Response: `{ ok: true, cmd }` where `cmd` is sanitized visualization JSON (allowlisted fields and types)

- **RAG**
  - Reindex:
    - POST `/v1/rag/reindex` (token + 30/min)
    - Response: `{ ok: boolean, count?: number, error?: string }`
  - Query:
    - POST `/v1/rag/query` (no auth for local eval; can be gated in prod)
    - Request: `{ query: string }`
    - Response: `{ ok: true, hits: [{ text: string, meta: any }...] }` (aggregate 30K char cap)

- **Email Drafts**
  - POST `/v2/email/draft` (token + 60/min)
    - Request: `{ to: EmailStr, subject: string<=200, body: string<=10000 }`
    - Response: `{ ok: true, id, path }`
  - GET `/v2/email/draft/{draft_id}` (token + 120/min)
    - Response: `{ ok: true, id, draft } | { ok: false, error }`

- **Calendar Drafts**
  - POST `/v2/calendar/draft` (token + 60/min)
    - Request: `{ title: string<=200, when: ISO8601 }`
    - Response: `{ ok: true, id, path }`
  - GET `/v2/calendar/draft/{draft_id}` (token + 120/min)
    - Response: `{ ok: true, id, draft } | { ok: false, error }`

- **Drafts Index**
  - GET `/v2/drafts?types=...&since=...&limit=...&offset=...` (token + 120/min)
  - DELETE `/v2/drafts` (token + 30/min)
    - Request: `{ ids: string[] }`
    - Response: `{ ok: true, removed: string[], missing: string[] }`

- **Learn Feedback**
  - POST `/v1/learn/feedback` (token + 60/min)
  - Request: `{ message: string, thumbs: 'up'|'down', meta?: object }`
  - Response: `{ ok: true }`

---

## Appendix B — Module by Module Walkthrough

- `interfaces/desktop/main.js`
  - App bootstrap, data dirs, logging, WS server (port autodiscovery), HUD window creation.
  - IPC handlers (scan, launch, gmail modal/send, vosk init/chunk/stop, open-url, brave reuse, whatsapp drafts, focus window title, visualize broadcast, HUD pin-through, data dir discovery).
  - CSV/JSON feeds watcher → `bar3d` create/update over WS.
- `interfaces/desktop/hud/VisualizationEngine.js`
  - Registry of objects; `applyCmd` for `create|update|remove`; accepts `bar3d` alias.
- `interfaces/desktop/index.html`
  - Voice UI, Hinglish time parsing, TTS voice fallback, intent routing to backend visualize.
- `backend/main.py`
  - Token startup check, CORS (null origin), rate limit middleware, health endpoint, routers.
- `backend/core/ai/model_router.py`
  - Signal extraction, slow-net probe, learned Hindi rule, provider retries, local LLM lifecycle.
- `backend/core/security/vault.py` / `vault_index.py`
  - AES‑256‑GCM + DPAPI, encrypted index, plaintext migration, AAD usage.
- `backend/core/rag/chroma_client.py`
  - PersistentClient singleton guarded by a lock; collection metadata/versioning.
- `interfaces/desktop/automation/*`
  - Brave reuse (puppeteer-core), WhatsApp compose, PowerShell focus with `-ExecutionPolicy Bypass`.

---

## Appendix C — Troubleshooting & Runbooks

- **Offline Mode**
  - Symptom: Replies begin with Hinglish offline prefix.
  - Checks: DNS/proxy; latency to gstatic; provider API keys.
  - Workaround: Force local-only mode for demos.

- **WS/HUD Connectivity**
  - Symptom: No visualizations.
  - Checks: `VIHA_WS_PORT`, firewall rules, HUD query param; console logs in HUD.
  - Recovery: HUD auto‑reconnect; restart app if port binding failed.

- **Vosk Model Issues**
  - Symptom: `{ ok: false, error: 'model folder not found' }`.
  - Checks: `resources/models/vosk` exists; local `data/models/vosk` in dev.
  - CI: Use `VIHA_SKIP_VOSK=1`.

- **Brave Reuse**
  - Symptom: New Brave instance launched instead of reuse.
  - Checks: Remote debugging enabled; correct port; no profile locks.

- **Vault Access**
  - Symptom: Cannot read previous drafts after migration.
  - Checks: DPAPI per‑machine vs per‑user context; plaintext `index.json` migrated/deleted.

---

## Appendix D — Deployment, Signing, and Installer

- **Build**
  - `npm --prefix interfaces/desktop run build:electron` (Windows NSIS)
- **Signing**
  - Tag builds in CI use `CSC_LINK`, `CSC_KEY_PASSWORD` for code signing.
- **Installer**
  - EULA shown; one‑click with elevation allowed; data stays under userData.

---

## Appendix E — Extended Test Matrix

- **Electron E2E**
  - Transparency + backgroundColor `#00000000` assertion.
  - WS broadcast round‑trip (renderer → main IPC → WS → HUD).
  - Pin‑through toggle via Ctrl+Shift+P; click event simulated.
  - Feeds: create/update `sales.csv` and assert HUD vis id `sales` type `bar|bar3d`.
  - Vosk IPC: `viha.voskInit()` returns `{ ok: true }` with `VIHA_SKIP_VOSK=1`.

- **Backend pytest**
  - Rate limit: ensures 429 headers; token requirement.
  - Learned Hindi routing: enforce local after negative feedbacks.
  - Health: returns memory/disk metrics.

---

## Appendix F — Glossary

- **HUD**: Heads‑Up Display — transparent overlay for visualizations.
- **RAG**: Retrieval Augmented Generation.
- **DPAPI**: Windows Data Protection API, used to protect secrets at rest.
- **WS**: WebSocket.
- **E2E**: End‑to‑end tests.

---

## Appendix G — Performance Benchmarks & Profiling Methodology

- **Methodology**
  - Hardware: Windows 11 Pro, 16 GB RAM, i7‑class CPU, SSD.
  - Tools: psutil (RSS), Node process metrics, Playwright traces, Windows Resource Monitor.
  - Scenarios: cold start, HUD render loop idle, CSV feed spikes, RAG reindex 1k files, local LLM first load.

- **Key Metrics**
  - Cold start time (Electron to HUD visible): 1.8–3.2s typical.
  - WS bus latency (renderer → HUD): < 20ms on localhost.
  - CSV feed update to visual frame: 30–60ms (parsing + render scheduling).
  - RAG reindex 1k mixed files: 45–120s depending on PDF/OCR concentration.
  - LLM local load: depends on model size; track RSS delta in logs.

- **Optimizations**
  - Debounced file watcher (200ms) reduces churn.
  - Engine only updates changed object materials/geometry.
  - Optional embeddings caching in reindex loop (future enhancement).

---

## Appendix H — Threat Scenarios & Controls

- **Prompt Injection via Intent**
  - Control: `_sanitize` allowlist, caps for values/percent/scale/label.
  - Outcome: only known keys/types pass; rejects arbitrary JS/URLs.

- **Unauthorized API use**
  - Control: token check at startup; enforced `Depends(require_token)` on write routes.
  - Rate limiting: SlowAPI with clear headers and consistent 429.

- **Local Data Exfiltration**
  - Control: No telemetry; explicit user actions for any outbound flows.
  - Vault encryption: even drafts stored at rest are encrypted.

- **IPC Abuse**
  - Control: tight set of handlers; no generic eval/execute; error handling returns structured responses.

---

## Appendix I — Code Path Annotations (Selected)

- `interfaces/desktop/main.js`
  - `createHudWindow()` → transparent BrowserWindow, query param `ws` injection.
  - `startFeedWatcher()` → `.csv`/`.json` parse → `{ type:'vis', cmd }` broadcast.
  - `ipcMain.handle('visualize-cmd')` → centralized WS send path.

- `backend/api/v1/intent.py`
  - `_extract_json()` → robust fenced/inline JSON capture; errors safe to None.
  - `_sanitize()` → caps `values` length (32), `label` length (80), color/scale normalization.

---

## Appendix J — Configuration & Tuning Guide

- **Environment**
  - `VIHA_API_BASE`: Set to LAN backend when deploying in shared office.
  - `VIHA_TOKEN`: rotate per machine/user; store in Windows Credential Manager if desired.

- **Models**
  - Place GGUF under `data/models` with clear naming; ensure paths match config.
  - For Vosk, prefer compact Hindi models for faster disk I/O.

- **RAG**
  - Exclude large binary assets from scan to reduce noise.
  - Use JSON feeds for structured dashboards for ultra‑low overhead updates.

---

## Appendix K — Incident Response & Recovery

- **Corrupted Vault File**
  - Symptom: decrypt error.
  - Action: restore from backup; verify DPAPI context (same user/machine).

- **Stuck Brave Reuse**
  - Action: restart Brave with `--remote-debugging-port=9222` and retry; or fallback to default browser.

- **Excessive Memory on LLM Load**
  - Action: check logs for RSS delta; switch to smaller GGUF or disable local LLM for the session.

---

## Appendix L — Compliance Mapping (Indicative)

- **Data Residency**
  - All core operations remain local; optional backend can be self‑hosted on‑prem.

- **Consent**
  - Gmail ToS gate; WhatsApp drafts only.

- **Access Control**
  - Token‑based auth; recommend per‑user tokens and local OS protections.

---

## Appendix M — Frequently Asked Questions (FAQ)

- Can VIHA run fully offline?
  - Yes. Voice via Vosk (Hindi), HUD, CSV/JSON feeds, RAG, local LLM routing.

- Does VIHA send my data to cloud by default?
  - No. Outbound flows require explicit actions and are visible (browser automation).

- What if my antivirus flags automation?
  - We use puppeteer‑core (no Chrome download) and Windows APIs; typically AV‑friendly.

---

## Appendix N — Extended Glossary & Conventions

- **A2D**: Audio‑to‑Data — pipeline from mic buffers to recognized text.
- **HUD Cmd**: Sanitized JSON directive for visualization engine.
- **Local‑first**: Prefers local resources, with cloud only when explicitly chosen.

---

## Appendix O — Hindi Time Parsing Examples (100+)

Below are representative examples that the Hinglish/Hindi time parser is designed to interpret. Each maps to a relative ISO datetime with heuristics for morning/noon/evening/night and explicit “baje”.

1. kal subah 8 baje
2. kal subah 10:30 baje
3. kal dopahar 1 baje
4. kal dopahar 2:15 baje
5. kal shaam 7 baje
6. kal shaam 6:45 baje
7. kal raat 9 baje
8. kal raat 11:10 baje
9. parso subah 9 baje
10. parso shaam 7:30 baje
11. aaj shaam 7 baje
12. aaj dopahar 3 baje
13. aaj raat 10 baje
14. aaj subah 7 baje
15. kal 5 baje (evening default)
16. parso 8 baje (evening default)
17. monday shaam 7 baje
18. tuesday subah 8 baje
19. wednesday dopahar 2 baje
20. thursday raat 9 baje
...
97. shaam 6:15 baje (today)
98. subah 7 baje (today)
99. raat 11 baje (today)
100. dopahar 12:30 baje (today)

Heuristics:
- “subah” → 8:00 default; “dopahar/noon” → 13:00; “shaam” → 19:00; “raat” → 22:00.
- Explicit HH[:MM] baje overrides default; AM/PM handles “12” edges.
- Relative words: aaj=0, kal=+1, parso=+2; day‑of‑week optional extension.

---

## Appendix P — Case Studies & Usage Patterns

- Sales Ops: CSV `sales.csv` updated hourly by ERP export; HUD shows bar3d trend; voice: “isko cyan kar do”, “size badhao”.
- Support NOC: Particles flow for network traffic; voice toggles colors by severity.
- Founder Daily: “kal shaam 7 baje meeting draft banao” → calendar draft; “Gmail bhejo” guarded by ToS gate.

---

## Appendix Q — OWASP ASVS Mapping (Indicative)

- V2 Auth: Token required for mutations (passes).
- V3 Session: Local‑only; no persistent web session tokens (n/a).
- V5 Access Control: Rate limit + auth (passes).
- V6 Input Validation: Pydantic + intent sanitizer (passes).
- V10 Malware: No DLL injection/robotjs; puppeteer‑core only (passes).

---

## Appendix R — Performance Tuning Deep Dive

- Renderer budget: keep frame time under 16ms; heavy geometry updates throttled.
- WS batching: group multiple updates per animation frame where possible.
- OCR: only when needed; terminate worker on quit.

---

## Appendix S — Playwright/E2E Scenarios (Expanded)

- Pin‑through toggle via Ctrl+Shift+P; click coordinates; validate environment flag.
- Feeds race: open WS, then write CSV, assert vis with id stem.
- Vosk IPC bypass using `VIHA_SKIP_VOSK=1` returns `{ ok: true }`.

---

## Appendix T — Hardware Profiles & Expectations

- Entry (8 GB RAM): use smaller GGUF; disable heavy particle counts.
- Mid (16 GB RAM): default settings; smooth HUD at 60 FPS.
- High (32 GB+): larger models, richer visuals.

---

## Appendix U — Onboarding & Admin Playbook

1. Install signed NSIS build; verify EULA.
2. Set strong `VIHA_TOKEN`; restart backend.
3. Validate HUD + WS; run Playwright smoke if needed.
4. Copy Vosk models if offline voice is required.
5. Configure contacts.json; test WhatsApp/Gmail drafts.

---

## Appendix V — Developer Onboarding

- Repos layout overview; key scripts; how to run tests.
- Coding standards: no token leaks, sanitize intents, try/catch all I/O.
- Adding new visual types: engine registry, sanitize shape, IPC round‑trip tests.

---

## Appendix W — Security Review Checklist

- [ ] `VIHA_TOKEN` not default; startup guard.
- [ ] Rate limits applied; 429 headers set.
- [ ] Intent sanitizer present and tested.
- [ ] Vault index encrypted; plaintext removed.
- [ ] No public tokens; preload‑only headers.

---

## Appendix X — Extended Threat Modeling

- Insider misuse: minimized by local‑only defaults and visible browser automations.
- Replay attacks on WS: local WS only; contents are visualization‑only.
- Path traversal: file scan honors walking with try/catch; no arbitrary delete.

---

## Appendix Y — Extended Glossary (Hindi/Hinglish)

- **Baje**: colloquial for “o'clock”; used with numeric time.
- **Shaam**: evening time bucket; default 19:00.
- **Dopahar**: noon/afternoon; default 13:00.

---

## Appendix Z — Index

AI Committee · 5, 19; Brave reuse · 8, 23; CSV Feeds · 3.4, 8.4; DPAPI · 6.1; HUD · 3; Intent Sanitization · 6.4, 13; Model Router · 5; RAG · 7; TTS · 4.4; Vosk · 4.2; WS Bus · 3.2; Vault · 6.1; Windows Hardening · 12; E2E · 24; CI · 10.3.

---

## Appendix DI — ASCII Diagrams (Placeholders)

### DI.1 System Topology

```
┌──────────────────────────────────────────────────────────────┐
│                       Windows 10/11 Host                     │
│                                                              │
│  ┌───────────────┐      IPC       ┌───────────────────────┐  │
│  │ Electron Main │◀──────────────▶│ Electron Renderer     │  │
│  │ (main.js)     │                │ (index.html)          │  │
│  └──────┬────────┘                └───────────┬───────────┘  │
│         │ WS:17321                                   │       │
│         ▼                                            ▼       │
│  ┌────────────────┐      vis cmds            ┌────────────────┐│
│  │ WS Server      │────────────────────────▶ │ HUD (Three.js) ││
│  │ (main.js)      │ ◀─────────────────────── │ Visualization  ││
│  └────────────────┘  hello/heartbeat         └────────────────┘│
│                                                              │
│  ┌───────────────────────┐   HTTP (local)   ┌───────────────┐ │
│  │ Backend (FastAPI)     │◀────────────────▶│ Renderer UI   │ │
│  │ Token + SlowAPI       │                  │ (getApiBase)  │ │
│  └───────────────────────┘                  └───────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

### DI.2 Voice (Web Speech + Vosk)

```
Mic → Web Speech (if available) → text
      └─fallback→ AudioContext → 16kHz PCM16 → IPC (vosk-chunk) → Vosk Recognizer → text
                                                                   └→ offline prefix + route
```

### DI.3 RAG Pipeline

```
Folder Scan (4 workers) → Extractors (pdf-parse, mammoth, tesseract.js) → data/index.json
    ↓
Backend /v1/rag/reindex → Chroma PersistentClient (locked singleton) → Collection (viha_version)
    ↓
/v1/rag/query (embed optional) → Top-3 → 30K aggregate cap → hits
```

### DI.4 Automation & Reuse

```
Renderer → IPC open-url-brave → Main → Puppeteer-Core connect :9222 → reuse tab | new tab
Renderer → IPC focus-window-title → Main → PowerShell Add-Type(user32) → SetForegroundWindow
Renderer → IPC open-whatsapp → Main → compose wa.me | web URL (no auto-send)
```

### DI.5 Security & Data

```
Vault (AES-256-GCM, DPAPI key)  ── stores encrypted drafts + index(AAD)
Logs  (data/logs/viha.log)      ── desktop + backend file logs
Token (startup enforced)         ── backend writes
CORS  (allow null)               ── Electron file://
Rate limit (SlowAPI)             ── 429 with Retry-After + X-RateLimit-Remaining

<!-- build: 2025-11-01T07:02:23.3433746+05:30 -->

