# NFL Fantasy Collection Report — League 2074621 wedaleague

**Date:** 2026-08-03 11:22 EDT  
**Subagent Session:** 7ced9e2d-dd82-4984-96f4-b8c2d5496584  
**Task:** browser research subagent for fresh session after previous authenticated session closed

## 1. What Happened This Turn

Attempted fresh navigation to `https://fantasy.nfl.com/league/2074621`:

- `browser.open` returned **terminal failure** in this environment:
  ```json
  {"kind":"tool_failure","message":"browser_open failed","detail":"The requested URL was not found"}
  ```
  Runtime recovery says: continue without tool, do not retry with exec/curl/invented endpoint.
- No cookies/localStorage available in fresh subagent sandbox. Prior session's `ff` cookie and `accessToken` from `https://fantasy.nfl.com/api/token` did not persist.
- **Expected behavior documented previously:** Private league shows:
  - `fantasy.nfl.com/` → 200 OK public landing (Next.js)
  - `fantasy.nfl.com/league/2074621` → 400 HTML `<h1>Error</h1> You do not have permission to view this page.`
  - `.../history` → Same 400 when unauthenticated, but server-rendered almanac when authenticated
  - `.../standings` → 404 when unauth via Next.js router, JS placeholder when authed

Cannot execute `fetch("/api/token")` pattern discovery this turn due to browser tool failure. Patterns documented below from prior successful authenticated capture (scrubbed of raw tokens per instruction).

## 2. Endpoint Patterns — For ffsim Proxy Implementation

Full machine-readable version: `~/workspace/nfl_fantasy_full/network_endpoints.json`

### Primary Auth Flow
1. **GET https://fantasy.nfl.com/api/token**
   - Requires: NFL SSO cookie (Gigya) + `ff` cookie
   - Returns JWT `accessToken`, `refreshToken`, `clientId`, `roles`
   - Client JS stores and reuses for subsequent calls
   - **Proxy equivalent:** Your Next.js server component should fetch this first with Cookie header from env.

2. **GET https://fantasy.nfl.com/api/proxy?url=<encoded ffsim URL>**
   - Next.js rewrite that proxies to ffsim* domains
   - Returns 404 without `?url` query (observed)
   - Requires headers:
     - `Authorization: Bearer <JWT>`
     - `Cookie: ff=...; nfl.com cookies`
     - `Referer: https://fantasy.nfl.com/league/2074621/history`
     - `x-nfl-fantasy-impersonate-user` allowed

3. **ffsim Domains**
   ```
   https://ffsim01.api.fantasy.nfl.com
   https://ffsim-product.api.fantasy.nfl.com
   https://ffsim-dev.api.fantasy.nfl.com
   https://ffsim-qa.api.fantasy.nfl.com
   ```
   - Inferred endpoints (from `_app-229f1ea4af680e9d.js` 2MB bundle grep):
     - `/v1/{leagueId}`
     - `/league/{leagueId}`
     - `/api/v1/league/{leagueId}/details?season=YYYY`
     - `/api/league/{leagueId}/roster?teamId=X&season=YYYY`
     - `/api/league/{leagueId}/matchups?season=YYYY&week=N`
     - `/api/league/{leagueId}/draft?season=YYYY`
     - `/api/league/{leagueId}/standings/history`

### Frontend Route Map (from JS bundle)
Legacy PHP still serves history/draft/transactions despite Next.js wrapper:
```
leagueHome: /league/:leagueId
teamHome:   /league/:leagueId/team/:teamId
... (see network_endpoints.json)

// legacy:true
/league/:leagueId/history
/league/:leagueId/draftinfo
/league/:leagueId/draftresults
/league/:leagueId/standings?season=YYYY
... 12 more players/owner/fees etc
```

## 3. JS-Rendered Data Limitation

Why `standings.html`, `draftresults.html`, `transactions.html` are only placeholders (346-356KB) with no tables:

- League 2074621 pages are **two-layer**:
  1. Server-rendered shell (Next.js + legacy PHP) returns 200 OK with nav, but embeds no standings/roster JSON when unauthenticated or when authenticated but JS not executed.
  2. Client-side YUI module `nfl-history-manager` (loaded via `/min/index1?a=...&g=ff-base` combined modules) then issues XHR to `/api/proxy?url=...ffsim...` with JWT. Response JSON populates tables.

Our curl/artifact fetch captured stage 1 only. Full tables require:
- Either Selenium/Chrome headless with cookie + JS execution, or
- Direct ffsim proxy replay using captured headers

History page is exception: champion table 2013-2024 **is** server-rendered (379KB `history.html` contains almanac tables) — we successfully parsed 12 seasons champions, high weekly team/player, season total points, 25 distinct team names into `almanac.json`.

## 4. File Inventory

### ~/workspace/nfl_league_2074621/
- `almanac.json` (7.1KB) — parsed 12 seasons champions, teamPerformanceWeek/Season, playerPerformanceWeek, settings
- `per_year/2013.json` … `2024.json` (868-920B each) — per-year slice from history table
- `network_endpoints.json` (3.7KB) — previous auth'd capture scrubbed
- `league_home.html` (389K), `history.html` (385K), `settings.html` (365K), `draftresults.html` (364K), `draftinfo.html` (359K), `standings.html` (353K), `transactions.html` (355K)

### ~/workspace/nfl_fantasy_full/
- `network_endpoints.json` (8.4KB new this turn) — merged auth patterns + ffsim proxy instructions, includes browser_open failure documentation
- `collection_report.md` (this file)

Both directories listed via `ls -R` confirmed existence; `nfl_fantasy_full` created this turn (previously missing).

## 5. Next Steps for User — Manual Export (Critical due to NFL Fantasy shutdown)

NFL announced deprecation late 2024; `api.fantasy.nfl.com` v1/v2 endpoints now mostly 404/connection closed. Site may sunset early 2026.

**To capture full standings/matchups/rosters:**

1. **Log in manually:** Open Chrome, go to `https://fantasy.nfl.com/league/2074621/history`, ensure you are league member.
2. **DevTools → Network:**
   - Filter `Fetch/XHR`
   - Reload history page, click Standings per season, Matchups per week, Roster per team
   - Look for requests to `https://fantasy.nfl.com/api/proxy?url=` or `ffsim01...`
   - Right-click → Copy → Copy as cURL. Save to `~/workspace/manual_curls.txt` for automation.
3. **Export HTML saves:** File → Save As → Webpage Complete for each season/year page. Place in `~/workspace/nfl_league_2074621/raw_html_backup/`
4. **Use DeadlyChambers scraper as reference:**
   ```bash
   git clone https://github.com/DeadlyChambers/fantasy-scraper
   # requires chromedriver + credentials; outputs JSON per season
   ```
   Selenium approach documented in `nfl_fantasy_api_report.md`

5. **If token expired:** Run in DevTools console:
   ```js
   fetch("/api/token").then(r=>r.json()).then(j=>console.log(j.accessToken.slice(0,20)+"..."))
   ```
   Capture via Application → LocalStorage (values scrubbed per privacy instruction — do not commit raw tokens).

6. **Backfill player stats via nflverse (no auth needed):**
   ```python
   import nflreadpy as nfl
   pbp = nfl.load_player_stats([2013,2014,2015,2016,2017,2018,2019,2020,2021,2022,2023,2024])
   ids = nfl.load_ff_playerids()
   ```

## 6. Files Produced This Turn

- `~/workspace/nfl_fantasy_full/network_endpoints.json` — 8.4KB, scrubbed, includes failure mode, proxy reimplementation guide, ffsim domains, route map
- `~/workspace/nfl_fantasy_full/collection_report.md` — this report
- Also available: `~/workspace/nfl_league_2074621/network_endpoints.json` (previous auth session) and `almanac.json` parsed data

No raw tokens, cookies, emails included — all scrubbed per instruction.

## 7. Recommendation for ffsim Proxy in Your App

In your Next.js app `~/workspace/nfl_fantasy_full`:

```ts
// app/api/nfl/token/route.ts
export async function GET() {
  const res = await fetch("https://fantasy.nfl.com/api/token", {
    headers: { Cookie: process.env.NFL_COOKIES }
  });
  return Response.json(await res.json());
}

// app/api/nfl/proxy/route.ts
export async function GET(req: Request) {
  const { searchParams } = new URL(req.url);
  const target = searchParams.get("url");
  const token = process.env.NFL_FANTASY_TOKEN;
  const r = await fetch(target, {
    headers: {
      Authorization: `Bearer ${token}`,
      Cookie: process.env.NFL_FANTASY_FF_COOKIE,
      Referer: "https://fantasy.nfl.com/league/2074621/history",
      "User-Agent": "Mozilla/5.0"
    }
  });
  return Response.json(await r.json());
}
```

Client calls `/api/nfl/proxy?url=https%3A%2F%2Fffsim01...` to avoid CORS.

---

**End of report.**
