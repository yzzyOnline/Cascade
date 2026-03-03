# Cascade v1.1.0

Self-hosted REST middleware that routes AI prompts across multiple free providers — Gemini, Groq, Mistral, and Cerebras. Automatically cascades through a configurable model array, escalating on complexity and falling back on failure, so your app always gets an answer.

**[📖 Full Documentation](https://cascade.yzzy.online)** · **[🎮 Demo](https://cascade.yzzy.online/demo/)** · Made by [yzzy online](https://yzzy.online)

---

## Features

- 🔀 **Multi-Provider Cascade** — Routes prompts across Gemini, Groq, Mistral, and Cerebras automatically
- 🧠 **Difficulty Scaling** — A `0.0–1.0` float maps your prompt to the right starting model
- ⬆️ **Complexity Escalation** — If a model returns `too_complex`, Cascade climbs toward more capable models
- ⬇️ **Failure Fallback** — On errors or rate limits, Cascade falls back toward cheaper models
- 🔑 **Key Rotation** — Multiple API keys per provider, rotated automatically on 429s and auth errors
- 🛡️ **Rate Limiting** — Per-IP rate limiting on `/ask-ai` via `express-rate-limit`
- 🔒 **Origin Whitelist** — Restrict which domains can call your middleware
- 🟢 **Health & Wake Endpoints** — `/health` for status, `/wake` for cold-start prevention
- ✅ **Full Sweep Guarantee** — Every model is tried exactly once before returning an error

---

## Installation

Fork the repo on GitHub so you have your own copy to deploy, then clone your fork:

```bash
git clone https://github.com/YOUR_USERNAME/cascade
cd cascade
npm install
```

---

## Quick Start

### 1. Set up environment variables

Copy `.env.example` to `.env` and fill in your API keys:

```bash
cp .env.example .env
```

```env
MY_APP_SECRET=your-secret-here
GEMINI_KEY=your-gemini-key
GROQ_KEY=your-groq-key
MISTRAL_KEY=your-mistral-key
CEREBRAS_KEY=your-cerebras-key
```

### 2. Run locally

```bash
npm start
# Middleware listening on port 10000 at 0.0.0.0
# [Keys] GEMINI: 1 key(s) loaded
# [Keys] GROQ: 1 key(s) loaded
```

### 3. Deploy to Render (free)

Push your fork to GitHub, create a new Web Service on Render pointing at your fork, set the start command to `npm start`, and add your env vars in the dashboard. Your endpoint will be live at `https://your-service.onrender.com`.

> **Cold starts:** Render's free tier spins down after 15 minutes of inactivity. Call `GET /wake` to warm the server before your first prompt, or point an uptime monitor at it every 5 minutes.

### 4. Make your first request

```js
const res = await fetch("https://your-service.onrender.com/ask-ai", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    secret: "your-secret-here",
    difficulty: 0.5,
    prompt: "Return a JSON object with key 'status' and value 'ok'."
  })
});

const data = await res.json();
console.log(data.package);        // { status: "ok" }
console.log(data.answeredBy.model); // "llama-3.3-70b-versatile"
```

---

## API Reference

### POST /ask-ai

Send a prompt through the cascade. Returns the first successful model response.

**Request body:**

| Field | Type | Description |
|-------|------|-------------|
| `secret` | `string` | Must match `MY_APP_SECRET` env var. |
| `difficulty` | `float 0.0–1.0` | Starting position in the model array. `0.0` = cheapest/fastest, `1.0` = most capable. |
| `prompt` | `string` | Your prompt. Models are instructed to return a JSON object with `state` and `package` fields. |

**Response:**

```json
{
  "state": "complete",
  "package": { },
  "answeredBy": {
    "index": 1,
    "provider": "groq",
    "model": "llama-3.3-70b-versatile"
  }
}
```

**Error responses:**

| Status | Meaning |
|--------|---------|
| `403` | Invalid or missing secret. |
| `429` | Rate limit exceeded for your IP. |
| `503` | All models exhausted — no provider responded successfully. |

### GET /wake

Returns `200 "Full Sweep Online"`. Use this to warm the server after a cold start before sending your first prompt.

### GET /health

Returns server status, uptime, model count, key pool sizes, and config.

```json
{
  "status": "ok",
  "uptime": "0h 4m 21s",
  "models": 10,
  "keyPools": { "GEMINI": 1, "GROQ": 2, "MISTRAL": 1, "CEREBRAS": 1 },
  "rateLimit": { "max": 30, "windowMs": 60000 },
  "providerTimeoutMs": 15000
}
```

---

## How the Cascade Works

Cascade maps `difficulty` to a starting index in the MODELS array, then sweeps through models until one succeeds. A visited set guarantees every model is tried exactly once.

| Step | Trigger | Action |
|------|---------|--------|
| Start | — | Map `difficulty` → starting index |
| `too_complex` | Model can't handle the prompt | Climb toward more capable models |
| Error / 429 / 503 | Network failure or rate limit | Fall back toward cheaper models |
| Exhausted | All models visited | Return `503` |

**Key rotation** happens before a model is marked as failed — on a 429 or auth error, Cascade tries the next key in the pool first.

---

## Difficulty

```
index = Math.round(difficulty * (MODELS.length - 1))
```

The mapping is proportional regardless of array size — a 5-model array and a 20-model array both treat `0.5` as "start in the middle."

| Difficulty | Use when |
|------------|----------|
| `0.0 – 0.2` | Simple tasks — classification, short answers, data formatting |
| `0.3 – 0.5` | Medium tasks — summaries, Q&A, moderate reasoning |
| `0.6 – 0.8` | Complex tasks — code generation, long-form content, analysis |
| `0.9 – 1.0` | Hard tasks — deep reasoning, multi-step problems, nuanced generation |

---

## Model Array

`MODELS` in `cascade.js` is a plain array ordered from least to most capable. Add, remove, or reorder entries freely — the cascade adapts automatically. If a provider has no API key configured, it is skipped at runtime and treated as a failure.

```js
const MODELS = [
  { provider: 'cerebras', model: 'llama3.1-8b' },             // 0.00 — fastest, lightest
  { provider: 'groq',     model: 'llama-3.3-70b-versatile' }, // 0.11 — fast, reliable
  { provider: 'cerebras', model: 'llama-3.3-70b' },           // 0.22
  { provider: 'gemini',   model: 'gemini-2.0-flash' },        // 0.33
  { provider: 'groq',     model: 'llama-3.3-70b-specdec' },   // 0.44
  { provider: 'mistral',  model: 'mistral-small-latest' },    // 0.56
  { provider: 'groq',     model: 'llama-3.3-70b-versatile' }, // 0.67
  { provider: 'mistral',  model: 'mistral-large-latest' },    // 0.78
  { provider: 'gemini',   model: 'gemini-2.5-flash' },        // 0.89
  { provider: 'gemini',   model: 'gemini-2.5-pro' },          // 1.00 — most capable
];
```

**Supported providers:**

| Provider | Key env var | API |
|----------|-------------|-----|
| `gemini` | `GEMINI_KEY` | Google Generative Language |
| `groq` | `GROQ_KEY` | OpenAI-compatible |
| `mistral` | `MISTRAL_KEY` | OpenAI-compatible |
| `cerebras` | `CEREBRAS_KEY` | OpenAI-compatible |

---

## Key Rotation

Each provider supports multiple API keys separated by `|` in the env var. Cascade loads them into a pool at startup and rotates on 429 or auth errors.

```env
GEMINI_KEY=keyA|keyB|keyC
GROQ_KEY=keyOne|keyTwo
```

On startup the server logs how many keys were loaded per provider:

```
[Keys] GEMINI: 3 key(s) loaded
[Keys] GROQ: 2 key(s) loaded
```

Use `|` as the separator — it is guaranteed to never appear in API keys from any supported provider.

---

## Origin Whitelist

`ALLOWED_ORIGINS` in `cascade.js` controls which origins can call `/ask-ai`. An empty array disables origin checking entirely. The `secret` field provides a second layer of protection regardless.

```js
// Allow everyone (dev / testing)
const ALLOWED_ORIGINS = [];

// Restrict to your domain
const ALLOWED_ORIGINS = [
  'https://yourdomain.com',
  'https://yourgame.itch.io'
];
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `MY_APP_SECRET` | Yes | — | Secret key required on every `/ask-ai` request |
| `GEMINI_KEY` | If using Gemini | — | Google AI key(s), pipe-separated |
| `GROQ_KEY` | If using Groq | — | Groq key(s), pipe-separated |
| `MISTRAL_KEY` | If using Mistral | — | Mistral key(s), pipe-separated |
| `CEREBRAS_KEY` | If using Cerebras | — | Cerebras key(s), pipe-separated |
| `PORT` | No | `10000` | Server port. Render sets this automatically. |
| `PROVIDER_TIMEOUT_MS` | No | `15000` | Per-request timeout in ms |
| `RATE_LIMIT_MAX` | No | `30` | Max requests per window per IP |
| `RATE_LIMIT_WINDOW_MS` | No | `60000` | Rate limit window in ms |

---

## Known Limitations

**Prompt structure is your responsibility** — Models are instructed to return `{ state, package }` JSON, but what goes inside `package` depends entirely on how you phrase your prompt. Be explicit about the shape you want.

**`too_complex` is model-reported** — Whether a model escalates is up to the model itself, not a measurable metric. Some models may never return `too_complex` regardless of difficulty.

**No streaming** — `/ask-ai` is a blocking request. Cascade waits for a complete response before returning. For long prompts this can approach the `PROVIDER_TIMEOUT_MS` limit.

**Free tier cold starts** — On Render's free tier, the first request after 15 minutes of idle can take 30–60 seconds. Use `/wake` or an uptime monitor to prevent this.

**In-memory only** — No request logging or response caching. Each call is stateless.

---

## License

MIT © yzzy online