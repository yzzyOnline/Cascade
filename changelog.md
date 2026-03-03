# Changelog

All notable changes to Cascade are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/). Cascade uses [Semantic Versioning](https://semver.org/).

---

## [1.1.0] — 2026-03-02

### Changed

**Sweep engine rewrite** — The fallback logic in `executeFullSweep` has been replaced with two explicit navigation functions, making escalation and fallback behaviour intentional and predictable.

- `nextOnError(from)` — on a network error, rate limit, or timeout, Cascade now searches **downward** first (toward cheaper models), then upward if nothing is available below. Previously used a spiral outward that could revisit higher-capability models unexpectedly on failure.
- `nextOnTooComplex(from)` — on a `too_complex` response, Cascade now searches **upward only**. Previously the spiral could return a lower-index model after a `too_complex` signal, which was incorrect behaviour.

**Response shape fix** — `/ask-ai` no longer double-wraps the model response. `package` is now the model's value directly.

```diff
- "package": { "package": { ... } }
+ "package": { ... }
```

> ⚠️ **Breaking change** — update any client code that reads `data.package.package` to read `data.package` instead.

### Fixed

- `too_complex` escalation could previously stall or move to a lower-capability model due to the shared `nextUnvisited` function being used for both error and complexity cases. These paths are now separate.
- Error fallback could loop or stall when `currentIndex - 1` had already been visited, since it decremented blindly without checking the visited set. Now scans for the nearest unvisited lower index.

### Internal

- Added JSDoc-style comment to `difficultyToIndex` clarifying that out-of-range values are silently clamped.
- Updated `gemini-2.5-flash` comment from `"gemini 2.5 thinking"` to `"gemini 2.5 flash"`.

---

## [1.0.0] — Initial release

- Multi-provider cascade across Gemini, Groq, Mistral, and Cerebras
- `difficulty` float (0.0–1.0) maps to starting model index via `Math.round`
- Full sweep engine with visited-set guarantee — every model tried exactly once
- Key rotation per provider via `|`-separated env vars
- Per-IP rate limiting via `express-rate-limit`
- Origin whitelist via `ALLOWED_ORIGINS`
- Timing-safe secret validation via `crypto.timingSafeEqual`
- `GET /wake` — cold-start ping endpoint
- `GET /health` — uptime, model count, key pool sizes, and config
- `POST /ask-ai` — main prompt endpoint
- Configurable `PROVIDER_TIMEOUT_MS` per provider call
- `server.keepAliveTimeout = 125000` for Render compatibility