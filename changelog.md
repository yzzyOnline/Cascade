# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [v1.0.0] - 2026-03-02

### Added

- **Cascade routing** — `difficulty` float (0.0–1.0) maps requests to a starting model; automatically escalates toward more capable models on `too_complex` and falls back toward cheaper models on errors or rate limits
- **Visited-model tracking** — guarantees every model is tried exactly once before giving up, preventing infinite loops
- **Key rotation** — multiple API keys per provider separated by `|`; rotates automatically on 429 or auth errors before marking a model as failed
- **Origin whitelist** — `ALLOWED_ORIGINS` array in `cascade.js` restricts which domains can call the middleware. Empty array allows all origins
- **Wake endpoint** — `GET /wake` returns `200 "Full Sweep Online"` for cold-start warmup with uptime monitors
- **Health endpoint** — `GET /health` returns uptime, model count, key pool sizes, and current config
- **Configurable model array** — add, remove, or reorder models in the `MODELS` array in `cascade.js`; the cascade adapts automatically
- **Rate limiting** — per-IP request limiting via configurable `RATE_LIMIT_MAX` and `RATE_LIMIT_WINDOW_MS`
- **Provider timeout** — configurable `PROVIDER_TIMEOUT_MS` per request

### Supported providers

- Gemini (`GEMINI_KEY`)
- Groq (`GROQ_KEY`)
- Mistral (`MISTRAL_KEY`)
- Cerebras (`CEREBRAS_KEY`)