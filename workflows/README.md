# Tickle Charge – Reddit lead scraping (n8n)

Import both JSON files into n8n (replace the existing workflows).

## Before activating

1. **Proxy password** – in *Receive and Start Scraping → Fetch RSS Feed (with Proxy) → Options → Proxy*,
   replace `YOUR_DECODO_USER:YOUR_DECODO_PASSWORD` with the real Decodo credentials
   (removed so they are not stored in git).
2. **Sheet columns** – add these headers to the *Scraping Result - New Prompt* tab (any position):
   `qualificationScore`, `leadTier`, `suggestedResponse`.
3. Sort/filter that tab by `leadTier` = Hot / Warm to get the sales list.

## Why the sheet was barely updating

| Problem | Evidence in the sheet | Fix |
|---|---|---|
| 7-day (168 h) cutoff on niche subreddits | *Scraping Metadata*: most feeds show 0–2 posts < 168 h | Window is now 30 days (`MAX_AGE_HOURS` in *Extract RSS Items*). Already-analyzed posts are remembered, so the longer window does not re-run GPT on the same posts |
| Only 17 narrow keywords, several in the wrong subreddits (e.g. "GCC payment problem" in r/gambling, "compliant" in r/fintech) | Irrelevant rows (consumer gamblers, AI-for-advisors) | 44 feeds / ~150 phrases across ~27 subreddits + site-wide Reddit search, OR-grouped 3 phrases per request (edit in *Generate Search URLs*) |
| Only 25 results per feed | `total posts` capped at 25 | `limit=100&t=month` on every feed URL |
| Prompt disqualifies anything without an explicitly written $1M/month volume – almost no Reddit post states volume | `volumeEstimate` = "Not mentioned in post" on every row | Posts now get a `qualificationScore` (0-100) and `leadTier` (Hot/Warm/Cold/Disqualified). Unknown volume = Warm, not Disqualified. Hard disqualifiers (countries, industries, pre-revenue) stay strict |
| If the AI returned bad JSON, the parser returned 0 items and the post loop stopped silently – the rest of the feed was never written | – | Parser always returns one row; AI node retries and continues on error |
| Every skipped old post wrote a "No" row to *Logs* | *Logs* has 25k rows | Skipped posts go to a No-op node; one log row per feed remains |
| Irrelevant posts sent to GPT | – | Cheap keyword pre-filter (`RELEVANCE_TERMS`) before the AI |
| `Extract Metadata` crashed on blocked/empty feeds and counted single-post feeds as 0 | – | Handles missing / single `entry` |
| Upsert matched on `postTitle` (titles repeat across posts) | – | Matches on `postUrl` |
