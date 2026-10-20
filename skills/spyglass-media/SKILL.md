---
name: spyglass-media
description: Find paid Meta ads, organic Instagram posts, and TikTok videos with creative metadata, hooks, angles, and performance signals. Use for competitive ad research, discovering winning creatives on a topic, or pulling a brand's ad library. Supports corpus semantic search, taxonomy filtering, and brand-level retrieval.
allowed-tools: mcp__plugin_spyglass_spyglass__getMedia, mcp__plugin_spyglass_spyglass__searchTaxonomies, mcp__plugin_spyglass_spyglass__getAccounts, mcp__plugin_spyglass_spyglass__getBrandMediaInsights, mcp__plugin_spyglass_spyglass__getBrandCampaigns
---

Use the connected Spyglass tools by their action names in Cowork; the plugin-scoped tool names in the front matter apply to Claude Code. Follow the live tool schema. Indexing instructions apply only if both `addAccounts` and `getAccountsIndexStatus` are available and the user has authorized indexing; otherwise explain the account is not currently indexed and continue with available accounts.

# spyglass media

Search for paid Meta ads, organic Instagram posts, and TikTok videos across the Spyglass corpus, or pull a specific account's library with performance sorting and creative metadata.

For a topic, product, or creative format, start with a short phrase in `getMedia` (`query` only), lock the product vertical with `searchTaxonomies` when neighboring categories bleed in, or discover the leading brands with `getAccounts` (`smart` mode) before opening their libraries. For a known brand or creator, resolve the account with `getAccounts`, check `getBrandMediaInsights` or `getBrandCampaigns` for high-level clusters, and pull `getMedia` by `accountId` with no `query`.

## 1. Pick the right mode before calling

`getMedia` runs three different engines behind one tool. Mixing them returns a confident, wrong set.

- **Corpus semantic search (`query` only, no `accountId`)**: Finds creatives across the entire database by visual or spoken similarity.
- **Brand library (`accountId` only, no `query`)**: A direct database table read of one account's ads or posts, sorted by performance or recency.
- **Theme inside a brand (`accountId` + `query`)**: Vector search restricted to one account. Use this only when the user asks how a specific brand advertises a specific product line, ingredient, or angle (e.g., "Primal Queen's iron ads" or "Amazon's pet food ads"). Never add a `query` when the user just wants a brand's general library.

Passing neither `accountId` nor `query` fails validation immediately.

## 2. How to find content on a topic (3-way fallback)

Do not run one `getMedia` call and treat whatever comes back as the whole answer, and do not fire 6 near-identical rephrasings in a loop. Try these three approaches in order when a search comes back thin, off-topic, or dominated by one brand:

1. **Say it the way the ad says it (`query` only)**
   Write a 2–5 word phrase a viewer would see on screen or hear in the hook: `"hydrogel eye patches"`, `"before and after acne"`, `"interactive product quiz"`.
   - If a topic has distinct angles, run 2–3 short, distinct queries (`"peptide serum routine"`, `"copper peptides before and after"`) with `limit: 25`.
   - Leave `sortBy` empty on discovery searches so results stay ranked by vector similarity.

2. **Lock the product category with `searchTaxonomies` (`query` + `filters.taxonomy`)**
   Semantic search bleeds across neighboring categories (`"serum"` pulls hair serum; `"loader"` pulls toy trucks).
   - Call `searchTaxonomies` with the product noun first (free).
   - Pass an exact returned path (e.g., `Beauty > Skincare > Serums`) in `filters.taxonomy` alongside your short `query`.

3. **Find the accounts in the niche, then open their libraries (`getAccounts` → `getBrandMediaInsights` → `getMedia`)**
   Corpus search surfaces scattered posts, not a cohesive campaign. When you need the real leaders in a niche—or when semantic search is noisy—call `getAccounts` in `smart` mode with a concrete market description (`"DTC electrolyte hydration powder brands"`).
   - Pick the 2–3 most relevant accounts.
   - Check `getBrandMediaInsights` (`HOOK` or `AD_ANGLE`) or `getBrandCampaigns` (`CAMPAIGN` or `OFFER`) first to see their creative clusters and active promos.
   - Then pull `getMedia` by `accountId` (with no `query`, `limit: 25–30`) to get the standout creatives.

## 3. Channel, sort, and status rules

- **Match `sortBy` to the channel:**
  - `MOST_IMPRESSIONS` and `LONGEST_RUN` are **strictly** valid when `channel` is `['META_AD']` alone. Passing them when `channel` is null or includes `INSTAGRAM` or `TIKTOK` fails validation.
  - `HIGHEST_OUTLIER` (best for breakout/viral posts relative to that account's median), `MOST_VIEWS`, `HIGHEST_ENGAGEMENT`, `MOST_LIKES`, and `MOST_COMMENTS` are organic only (`INSTAGRAM`, `TIKTOK`).
- **Use performance sorts for "winning", "best", or "top" requests:**
  - For paid Meta ads: `channel: ['META_AD']`, `sortBy: 'MOST_IMPRESSIONS'` (or `'LONGEST_RUN'`), `status: 'ALL'`.
  - For organic posts: `channel: ['INSTAGRAM', 'TIKTOK']`, `sortBy: 'HIGHEST_OUTLIER'` or `'MOST_VIEWS'`.
- **When a brand has `0` Meta ads:** If `channel: ['META_AD']` with `status: 'ALL'` returns `totalCount: 0`, the brand has no indexed Meta ads. Do **not** retry the same `META_AD` call in a loop. Tell the user they have no indexed paid ads, and if organic content is useful, make a separate call with `channel: ['INSTAGRAM', 'TIKTOK']` and an organic sort.

## 4. Cap limits and warn before wide fan-outs

`getMedia` returns every fetched row inline and bills per creative.
- **Keep pulls small (`limit: 25–50`).** Leaving `limit` off defaults to 200 on semantic searches and 2,000 on brand libraries (~$1 and ~200k tokens in a single call).
- **Warn before a wide fan-out.** If a user gives you 10+ brands or asks to scan an entire category's roster, do **not** pull 100–500 rows per brand in one turn.
  - Either warn the user first (state how many accounts, the limit per account, and that it is a broad pull, then wait for confirmation) or run a pilot on the top 3–5 accounts (`limit: 20–30` each) and offer to expand.
  - To check whether an account has active ads or how many ads it runs without pulling the library, pass `limit: 1` and read `totalCount` from the response header.

## Critical Pitfalls

- **Setting `channel: ['INSTAGRAM']` when the user asks for "Instagram ads":** In Spyglass, `channel` is the source feed, not the ad placement. **All paid ads on Facebook and Instagram live under `META_AD`.** `INSTAGRAM` and `TIKTOK` are strictly organic profile posts. If someone asks for "paid Instagram ads" or "IG ad creatives", use `channel: ['META_AD']`.
- **Hallucinating `filters.taxonomy` without calling `searchTaxonomies` first:** Taxonomy paths must match exact indexed strings (e.g., `Beauty > Skincare > Serums`). Guessing a path like `"Skincare"` or `"Health > Supplements"` silently returns `0` results and makes it look like nobody advertises in that vertical. Always call `searchTaxonomies` first, and do not confuse `taxonomy` (product vertical) with `category` (creative format like `"UGC"` or `"Testimonial"`).
- **Contradicting `dateRange` and `daysRun` (e.g., `last-30-days` + `daysRun: { min: 60 } ends up empty`):** `dateRange` filters by when the creative **started** (`startAt`), not when it was active. An ad that started in the last 30 days cannot have run for 60+ days. When filtering for long-running ads (`daysRun.min` or `sortBy: 'LONGEST_RUN'`), omit `dateRange` or make sure the start window is wider than `daysRun.min`.
- **Leaving `status` at default (`ACTIVE`) on historical or all-time queries:** `status` defaults to `ACTIVE` on Meta ads. If the user asks for "all-time winners", "ads from last Black Friday", or "what they ran in 2025", leaving `status` unset silently drops every completed campaign. Set `status: 'ALL'`.
- **Adding `sortBy` to a semantic discovery search:** On `query` searches, Spyglass fetches the top vector matches first and then re-sorts them by `sortBy` (defaulting to `MOST_RECENT` when `sortBy` is set). That buries the closest semantic matches under tangentially related newer or higher-view posts. Omit `sortBy` when searching by `query` unless the user explicitly asked to rank a large pool (`limit: 50–100`).
- **Prompt-stuffing `query`:** Never put instructions, brand names, dates, platforms, demographics, or performance words in `query` (e.g., `"best performing Black Friday makeup static ads for mature women 50+"`). Embeddings only match what is shown or said in the creative; put structural constraints in `filters`.
- **Passing both `preset` and `startDate`/`endDate` in `dateRange`:** Use a `preset` OR explicit `startDate`/`endDate`, never both (sending empty strings `""` alongside a preset fails validation). Note that `last-year` is the prior calendar year; use explicit dates for a rolling 12 months.
- **Combining `filters.location` with `accountId`:** `location` filters by the brand's HQ country, not where an ad was targeted. Never set `location` when you already have an `accountId`.
- **Padding top-N lists with duplicate ad variants:** Brands often run 5–20 ad IDs with the exact same video or image to test different copy, landing pages, or audiences. Collapse duplicates into one entry by shared `hook`, `caption`, or `coreAngle` so you don't list the same creative five times—but explicitly call out the variant count (e.g., "running 12 active copies of this video"), because heavy duplication is a strong signal that the brand is actively scaling or testing that winner.
