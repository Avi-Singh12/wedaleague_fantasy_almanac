# 2018 Season – wedaleague (2074621)

This folder contains the archived data for the 2018 season.

**Present:**
- `almanac.json` – champion, team high / player high weekly, season total high, settings slice (parsed from server-rendered history page)

**Planned / future files (once ffsim API capture completes):**
- `standings.json` – final regular-season standings, playoff bracket
- `rosters.json` – rosters per team / per week
- `matchups.json` – weekly head-to-head scores
- `draft.json` – offline draft board
- `transactions.json` – waivers, trades, adds/drops

See `/league/network_endpoints.json` for endpoint pattern to fetch those via authenticated `/api/proxy?url=ffsim` proxy.

No credentials are stored in this repo – auth used transiently in browser only.
