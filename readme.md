# Cascade

A self-hosted REST middleware that routes AI prompts across multiple free providers — Gemini, Groq, Mistral, and Cerebras. Automatically cascades through a configurable array of models, escalating on complexity and falling back on failure, so your app always gets an answer.

**Docs:** [cascade.yzzy.online](https://cascade.yzzy.online)  
**Demo:** [cascade.yzzy.online/demo](https://cascade.yzzy.online/demo/)  
**By:** [yzzy.online](https://yzzy.online)

---

## How it works

A `difficulty` float (0.0–1.0) maps to a starting position in the MODELS array. Cascade then:

1. Calls the model at that index
2. If the model returns `too_complex` → climbs toward more capable models
3. If the model errors or rate limits → falls back toward cheaper models
4. Tracks visited models to guarantee every model is tried exactly once before giving up

---

## Quick Start

### 1. Fork and clone

Fork the repo on GitHub so you have your own copy to modify and deploy, then clone your fork:

```bash
git clone https://github.com/YOUR_USERNAME/cascade
cd cascade
npm install
```

### 2. Configure environment variables

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

### 3. Run locally

```bash
npm start
# Middleware listening on port 10000 at 0.0.0.0
```

### 4. Deploy to Render (free)

Push your fork to GitHub, create a new Web Service on Render pointing at your fork, set the start command to `npm start`, and add your env vars in the dashboard.

---

## API

### POST /ask-ai

```js
fetch("https://your-service.onrender.com/ask-ai", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    secret: "your-secret-here",
    difficulty: 0.5,       // float 0.0 (cheapest) → 1.0 (most capable)
    prompt: "Your prompt here."
  })
});
```

**Response:**
```json
{
  "state": "complete",
  "package": {
    "package": { }
  },
  "answeredBy": {
    "index": 1,
    "provider": "groq",
    "model": "llama-3.3-70b-versatile"
  }
}
```

### GET /wake

Returns `200 "Full Sweep Online"`. Use to warm the server after a cold start.

### GET /health

Returns uptime, model count, key pool sizes, and config.

---

## Configuration

### Model Array

Edit the `MODELS` array in `cascade.js` to add, remove, or reorder models. The cascade adapts automatically:

```js
const MODELS = [
  { provider: 'cerebras', model: 'llama3.1-8b' },             // 0.0 — fastest
  { provider: 'groq',     model: 'llama-3.3-70b-versatile' }, // 0.11
  { provider: 'mistral',  model: 'mistral-small-latest' },    // ...
  { provider: 'gemini',   model: 'gemini-2.5-pro' },          // 1.0 — most capable
];
```

### Key Rotation

Multiple keys per provider, separated by `|`:

```env
GEMINI_KEY=keyA|keyB|keyC
```

On a 429 or auth error, Cascade rotates to the next key before marking the model as failed.

### Origin Whitelist

Edit `ALLOWED_ORIGINS` in `cascade.js`. Empty array = accept from everywhere:

```js
const ALLOWED_ORIGINS = [];                          // open
const ALLOWED_ORIGINS = ['https://yourdomain.com']; // restricted
```

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MY_APP_SECRET` | — | Required on every /ask-ai request |
| `GEMINI_KEY` | — | Google AI key(s), pipe-separated |
| `GROQ_KEY` | — | Groq key(s), pipe-separated |
| `MISTRAL_KEY` | — | Mistral key(s), pipe-separated |
| `CEREBRAS_KEY` | — | Cerebras key(s), pipe-separated |
| `PORT` | 10000 | Server port. Render sets automatically. |
| `PROVIDER_TIMEOUT_MS` | 15000 | Per-request timeout in ms |
| `RATE_LIMIT_MAX` | 30 | Max requests per window per IP |
| `RATE_LIMIT_WINDOW_MS` | 60000 | Rate limit window in ms |

---

## Supported Providers

| Provider | Key env var |
|---|---|
| Gemini | `GEMINI_KEY` |
| Groq | `GROQ_KEY` |
| Mistral | `MISTRAL_KEY` |
| Cerebras | `CEREBRAS_KEY` |

---

## License

MIT — free to use, self-host, and modify.