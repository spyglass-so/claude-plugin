---
name: spyglass-accounts
description: Resolve brands and creators by name, handle, ID, or category brief, and index missing accounts into Spyglass. Use whenever a request names specific brands/creators, asks to discover competitors or influencers in a niche, or needs an account added before pulling its media, insights, campaigns, or landing pages.
allowed-tools: mcp__spyglass__getAccounts, mcp__spyglass__addAccounts, mcp__spyglass__getAccountsIndexStatus, mcp__spyglass__searchTaxonomies
---

# spyglass accounts

Find brands and creators in the Spyglass corpus with `getAccounts`, or index missing accounts on demand with `addAccounts` and `getAccountsIndexStatus` so their ads, organic posts, insights, campaigns, and landing pages become searchable.

For specific named brands or creators, always start with `getAccounts` in `brand_name` mode, verify that the returned row's `summary` matches the intended company, and call `addAccounts` if the account is missing. For discovering competitors or creators in a category, use `getAccounts` in `smart` mode with a concrete market description and demographic/follower filters.

## 1. Pick the right `getAccounts` mode

`getAccounts` has three modes (`brand_name`, `smart`, `ids`):

- **`brand_name` (exact / substring lookup on names, handles, and domains)**:
  Use whenever the user names one or more specific brands, creators, `@handles`, or domains.
  - Batch **all** named accounts into a single `brand_name` call (`brandNames` accepts up to 100 strings).
  - For each brand, include the exact name plus a spaced/unspaced variant and bare domain or handle if known (e.g., `["SotoTennis", "Soto Tennis", "sototennis.com"]`).
  - Omit `type` if you are unsure whether an account is indexed as a `BRAND` or `CREATOR`.

- **`smart` (vector discovery by market, niche, or audience)**:
  Use when discovering competitors, brands in a vertical, or creators in a niche (`"DTC electrolyte hydration powder brands"`, `"K-beauty skincare creators"`).
  - Write `query` as a self-contained market brief with concrete nouns (product type, category, audience, price tier). Never pass a vague word like `"competitors"` or `"similar brands"`.
  - When finding competitors for a known brand, resolve the reference brand first, read its `summary`, and build the `smart` query from its actual product category—then exclude the reference brand from your answer.

- **`ids` (re-hydration by ID)**:
  Use when you already have Spyglass UUIDs or numeric Facebook Ad Library IDs from prior tool calls or citations and need their profile summaries or follower counts.

## 2. Verify matches before pulling media

`brand_name` matches substrings (`ILIKE %name%`), and `smart` always returns the closest semantic neighbors even when a specific brand is not in the database.

- **Check `summary` for substring and homonym collisions:**
  - Short names frequently match unrelated accounts: `"Anessa"` matches `"Danessa Myricks Beauty"` and `"Joicy Vanessa"`; `"Chery"` matches `"Cheryl Porter"`; `"Tesa"` matches `"tastesalud"`; `"Collider"` (the non-alcoholic beer) matches `"Collider"` (`collider.com`, the entertainment news site).
  - Always read the `name`, `summary`, and `location` fields before passing an `id` to `getMedia`. Reject rows whose industry or product does not match the user's request.
- **Do not mistake `smart` fallback results for a named account:**
  - If `brand_name` returns `0` matches and you retry with `smart`, `smart` will return 20 accounts in the same category even if the target brand is not indexed. Only use a row from that retry if its `name` is actually the brand the user asked for.

## 3. When and how to index missing accounts (`addAccounts` + `getAccountsIndexStatus`)

If a named brand or creator is not in Spyglass (or `brand_name` only returned a collision in the wrong industry/country), index it with `addAccounts`:

1. **Gather `websiteUrl`, `igUsername`, or both:**
   - For an **account**, provide its storefront domain, Facebook page URL, or Facebook Ad Library URL as `websiteUrl` (e.g., `"https://drinkcollider.com"`, `"https://www.facebook.com/larocheposay"`, or `"https://www.facebook.com/ads/library/?view_all_page_id=15087023444"`) and/or its Instagram handle as `igUsername` (e.g., `"drinkcollider"`) if known.
2. **Batch all missing accounts into one `addAccounts` call:**
   - `addAccounts` accepts 1–20 accounts at once, starts indexing them in parallel, and immediately returns each account's `id`, `name`, `websiteUrl`, `igUsername`, and `status` (`"ready"` or `"indexing"`).
   - Never call `addAccounts` in a one-by-one loop for 5 different brands; pass all 5 in a single `accounts` array.
3. **Poll for indexing to finish with `getAccountsIndexStatus`:**
   - For any account returned with `status: "indexing"`, pass its `id` in `accountIds` to `getAccountsIndexStatus`.
   - `getAccountsIndexStatus` polls twice per call (30 seconds apart) and returns each account's updated `status` (`"ready"` or `"indexing"`).
   - If any account is still `"indexing"`, call `getAccountsIndexStatus` again (up to 10 times total) with the remaining indexing `accountIds` until `status` is `"ready"`.
4. **Use the returned `id` directly:**
   - Once an account's `status` is `"ready"`, pass its `id` straight to `getMedia`, `getBrandMediaInsights`, `getBrandCampaigns`, or `getBrandLandingPages`.
   - If `addAccounts` errors (e.g., unreachable domain or nonexistent handle) or an account is still `"indexing"` after 10 status checks, do **not** call `addAccounts` again—note that the account is still syncing or unavailable and continue with the accounts that are ready.

## 4. Creator discovery filters

When scouting creators (`type: 'CREATOR'`), put demographic and size constraints in the dedicated filter fields—**never** inside `query`:

- **Size & ranking:** Use `minFollowers`, `maxFollowers`, and `sortBy` (`HIGHEST_ENGAGEMENT`, `MOST_VIEWS`, or `MOST_FOLLOWERS`). Note that `minFollowers`/`maxFollowers` filter by Instagram follower count.
- **Demographics:** Use `gender` (`male`, `female`), `ageRange` (`18-24`, `25-34`, `35-44`, `45-54`, `55+`), `ethnicity`, `location` (country name), and `language`.
- **Zero-result fallback:** If a `smart` creator search with strict demographic/location filters returns `0` accounts, retry once without the demographic filters and filter the returned summaries yourself.

## Critical Pitfalls

- **Putting Instagram URLs into `websiteUrl` on `addAccounts` for a brand:** Passing `websiteUrl: "https://www.instagram.com/brand"` routes the account into the creator pipeline (`type: 'CREATOR'`) with `websiteUrl: null`. Put the Instagram handle in `igUsername` (without `@`) and either pass the brand's actual domain, Facebook profile/page URL (`facebook.com/{profileId}`), or Facebook Ad Library URL (`facebook.com/ads/library/?view_all_page_id=...`) in `websiteUrl`, or omit `websiteUrl` entirely.
- **Sending empty strings `""` for `websiteUrl` or `igUsername` in `addAccounts`:** Passing `websiteUrl: ""` triggers an unreachable-website error. Omit any field you do not have.
- **Appending country names or product words to `brandNames`:** In `brand_name` mode, each string is matched literally against the account's name, handle, and domain. Searching `brandNames: ["Pantene Indonesia", "Collider non-alcoholic beer"]` returns `0` results if the account is named `"Pantene"` or `"Drink Collider"`. Always include the bare brand name (`"Pantene"`, `"Collider"`) alongside any regional variant.
- **Including `@` or full URLs in `brandNames`:** Social handles in Spyglass are stored without `@` and prefix-matched (`handle%`). Passing `"@halazubari"` or `"https://instagram.com/halazubari"` will fail to match `igUsername`. Strip `@` and URL prefixes first (`"halazubari"`).
- **Calling `getAccounts` more than 2–3 times for the same question:** One `brand_name` call (with all names batched) plus at most one `smart` discovery call is enough. If an account still isn't found, call `addAccounts` or tell the user it isn't indexed instead of firing repeated `getAccounts` queries.
