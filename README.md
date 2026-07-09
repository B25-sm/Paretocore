# Pareto · Core Finder

Enter any field, tool, or skill → get the **20% that accounts for ~80% of real-world impact**, split into timeless fundamentals and the current (source-backed) landscape, with confidence labelled honestly and every live claim linked to a source you can check.

Static frontend + one serverless function. No build step, no frontend dependencies, and the function has no npm dependencies. Deploys free on Vercel.

## Structure

```
pareto-core/
├─ index.html          # the whole UI (vanilla HTML/CSS/JS)
├─ api/
│  ├─ analyze.js       # serverless function: Tavily search → Groq synthesis
│  └─ ask.js           # scoped follow-up Q&A on an analysis already produced
├─ vercel.json         # bumps the function timeout to 60s
└─ .env.example        # which env vars to set
```

## How it works

1. The browser posts `{ field, goal, live }` to `/api/analyze`.
2. If `live` is on, the function fires two parallel **Tavily** queries for current-landscape sources — one for "most important tools/skills," one for adoption/survey data — then merges, dedupes (by URL and by domain), and ranks them. If the goal implies career/hiring intent (mentions "hired," "job," "salary," "career," "interview," etc.), a **third** query for job demand/compensation fires alongside them, kept in its own pool so it can't get crowded out by the other two.
3. It passes those sources + your input to **Groq** with a strict "learning-prioritization analyst" system prompt (see `GROQ_MODELS` below).
4. It parses the model's JSON (defensively, with retries).
5. The response streams back as newline-delimited JSON events — `{type:"sources"}` as soon as search finishes, then `{type:"result"}` once synthesis completes — so the browser can show sources immediately and fill in the rest as it arrives.

### Market signal (job demand / compensation)

When the goal implies career intent, the result gets a fourth section — **"Market signal"** — covering things like job demand and typical compensation, each entry citing a real source (BLS, Glassdoor, Coursera, LinkedIn, etc.) and date. This was a deliberate line to hold: the ask that prompted it was for a "hiring probability %" and a "real-time money graph," and both of those are unbuildable honestly — no dataset publishes a probability of being hired, and comp/demand data comes from periodic surveys and aggregators, not a live feed. The system prompt explicitly forbids inventing a hiring-probability figure even if asked directly, and forbids computing/averaging/estimating a number yourself — only stating a number when a cited source actually states it. When the goal doesn't imply career intent, or the job-market search comes back with nothing usable, `market_signal` is an empty array and the section simply doesn't render — no placeholder, no guess.

Keys stay server-side in environment variables. Nothing sensitive is exposed to the browser, which also sidesteps the CORS walls that Tavily/Groq put up against direct browser calls.

### Ask a follow-up (`api/ask.js`)

Once a result renders, an "Ask a follow-up" box appears. This is **deliberately not a general chatbot** — the product brief calls that out as a non-goal. Each question is a single, stateless request: the browser sends the field/goal, the analysis already produced (`stable_core`, `current_landscape`, `market_signal`, `explicitly_deprioritized`, `caveats`), and a lightweight source list (title/url/date, no excerpts) back to `/api/ask`. The model answers using only that — no new search, no new facts, and if the question asks about something the analysis didn't cover, it says so instead of guessing. No conversation history is kept or resent between questions. Citations are checked server-side against the sources actually provided; a citation for a URL that wasn't in that list gets silently dropped before reaching the browser, whatever the model claims. It shares the same accuracy-first `GROQ_MODELS` fallback chain as `/api/analyze` — kept as a self-contained duplicate in `api/ask.js` rather than a shared helper file, so there's no dependency on Vercel's file-based function bundling behaving a particular way for non-route files.

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

- `GROQ_MODELS` — an ordered fallback chain, not a single model. On a rate limit (429), model-unavailable (404), or a model that never returns parseable JSON, the request cascades to the next model in the list before failing. Only comparable-capability models are listed (`openai/gpt-oss-120b` → `llama-3.3-70b-versatile` → `qwen/qwen3-32b`) — smaller/faster models are deliberately left out so a rate limit never silently trades accuracy for uptime. The response always reports which model actually served it (see the footer / `model` field), never just the configured primary. If you add a model to the chain, verify it first — Groq's `reasoning_effort` param, for example, is `openai/gpt-oss-*`-only and 400s on llama/qwen models (already handled conditionally in `callGroq`, but a new model may have its own quirks). Current free-tier limits: console.groq.com/docs/rate-limits.
- `SEARCH_DEPTH` — `"basic"` (1 Tavily credit/search) or `"advanced"` (2 credits, higher relevance).
- `PER_QUERY_RESULTS` — results requested per Tavily query.
- `MAX_RESULTS` — cap on sources fed to the model after merging + deduping (main pool only).
- `MARKET_MAX_RESULTS` — cap on the separate job-market/compensation pool.
- `CAREER_INTENT_RE` — the keyword regex that decides whether a goal triggers the third job-market query.

**Tavily credit cost:** a normal live run fires **two** Tavily queries in parallel — one for "most important tools/skills," one for adoption/survey data — then dedupes by URL and by domain before ranking. At `SEARCH_DEPTH = "advanced"` that's **4 credits per live run**. If the goal implies career intent, a **third** query fires too, bringing it to **6 credits**. Drop to `"basic"` if you're on a tight free-tier budget (2 or 3 credits respectively).

## Notes

- **The "Fetch live sources" toggle is the point of the two-tier design.** Leave it on for fast-moving topics (frameworks, tooling). Turn it off for timeless fundamentals (math, algorithms, design patterns) — the function skips the search entirely and the model returns stable core only. This is deliberate: don't spend a web request re-confirming that recursion matters.
- **This is a prioritization aid, not an oracle.** The system prompt is tuned to refuse invented numbers, label confidence, flag conflicting sources, and say when it's guessing — but always follow the source links before you commit a syllabus to it.
- **The JSON is yours to reuse.** Every response is a clean structured object (there's a "Copy JSON" button on results); drop it into whatever UI you're building on top.

## Want a "bring your own key" public demo instead?

Right now keys live on the server (correct for a tool you host for yourself or a cohort). If you'd rather let each visitor paste their own keys — so you're not paying for their usage — the function can accept keys in the request body with the env vars as fallback. Say the word and I'll wire that variant.
