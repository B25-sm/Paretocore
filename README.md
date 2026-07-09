# Pareto · Core Finder

Enter any field, tool, or skill → get the **20% that accounts for ~80% of real-world impact**, split into timeless fundamentals and the current (source-backed) landscape, with confidence labelled honestly and every live claim linked to a source you can check.

Static frontend + one serverless function. No build step, no frontend dependencies, and the function has no npm dependencies. Deploys free on Vercel.

## Structure

```
pareto-core/
├─ index.html          # the whole UI (vanilla HTML/CSS/JS)
├─ api/
│  └─ analyze.js       # serverless function: Tavily search → Groq synthesis
├─ vercel.json         # bumps the function timeout to 60s
└─ .env.example        # which env vars to set
```

## How it works

1. The browser posts `{ field, goal, live }` to `/api/analyze`.
2. If `live` is on, the function fires two parallel **Tavily** queries for current-landscape sources — one for "most important tools/skills," one for adoption/survey data — then merges, dedupes (by URL and by domain), and ranks them.
3. It passes those sources + your input to **Groq** (`openai/gpt-oss-120b`) with a strict "learning-prioritization analyst" system prompt.
4. It parses the model's JSON (defensively, with retries).
5. The response streams back as newline-delimited JSON events — `{type:"sources"}` as soon as search finishes, then `{type:"result"}` once synthesis completes — so the browser can show sources immediately and fill in the rest as it arrives.

Keys stay server-side in environment variables. Nothing sensitive is exposed to the browser, which also sidesteps the CORS walls that Tavily/Groq put up against direct browser calls.

## Setup

You need two keys:

- **Groq** — free tier, no card: <https://console.groq.com/keys>
- **Tavily** — free tier (generous credit allowance): <https://app.tavily.com>

### Run locally

```bash
npm i -g vercel        # if you don't have it
cp .env.example .env.local   # then paste your real keys into .env.local
vercel dev
```

`vercel dev` serves `index.html` and runs the `/api` function together, so live search works locally. (A plain static server won't — the `/api` route needs the Vercel runtime.)

### Deploy

Import the repo at <https://vercel.com/new>, or:

```bash
vercel            # first deploy (preview)
vercel --prod     # production
```

Then add `GROQ_API_KEY` and `TAVILY_API_KEY` under **Project → Settings → Environment Variables** and redeploy.

## Config knobs (`api/analyze.js`)

- `GROQ_MODEL` — one line. If Groq deprecates the current model, swap it here.
- `SEARCH_DEPTH` — `"basic"` (1 Tavily credit/search) or `"advanced"` (2 credits, higher relevance).
- `PER_QUERY_RESULTS` — results requested per Tavily query.
- `MAX_RESULTS` — cap on sources fed to the model after merging + deduping.

**Tavily credit cost:** each live run now fires **two** Tavily queries in parallel — one for "most important tools/skills," one for adoption/survey data — then dedupes by URL and by domain before ranking. At `SEARCH_DEPTH = "advanced"` that's **4 credits per live run** (up from 2). Drop to `"basic"` if you're on a tight free-tier budget.

## Notes

- **The "Fetch live sources" toggle is the point of the two-tier design.** Leave it on for fast-moving topics (frameworks, tooling). Turn it off for timeless fundamentals (math, algorithms, design patterns) — the function skips the search entirely and the model returns stable core only. This is deliberate: don't spend a web request re-confirming that recursion matters.
- **This is a prioritization aid, not an oracle.** The system prompt is tuned to refuse invented numbers, label confidence, flag conflicting sources, and say when it's guessing — but always follow the source links before you commit a syllabus to it.
- **The JSON is yours to reuse.** Every response is a clean structured object (there's a "Copy JSON" button on results); drop it into whatever UI you're building on top.

## Want a "bring your own key" public demo instead?

Right now keys live on the server (correct for a tool you host for yourself or a cohort). If you'd rather let each visitor paste their own keys — so you're not paying for their usage — the function can accept keys in the request body with the env vars as fallback. Say the word and I'll wire that variant.
