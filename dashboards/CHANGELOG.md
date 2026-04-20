# Dashboard Changelog

## v2 — 2026-04-18
- Restructured around MerchantBots vs Jeff clients (client_id filtering)
- Goal banner: two progress bars (meetings toward 30, positive replies toward 100)
- Daily tab fully rebuilt with 4 sub-sections:
  - A: Yesterday's snapshot (5 KPI cards vs 15-day sending-day average)
  - B: Daily summary table (16 days, sorted most recent first, bar chart on positive replies)
  - C: Sequence step breakdown (email 1/2/3 stacked % bars + positive replies)
  - D: Deliverability monitor (line chart: email 1 reply rate, total reply rate, bounce rate)
- All queries now use CRM metrics as source of truth
- Other tabs (Weekly, Series, Pacing, Jeff Clients, Experiments) stubbed as placeholders
- Switched from SL metrics to CRM metrics throughout

## v1 — 2026-04-17
- Initial dashboard build
- 5 tabs: Daily Trend, Weekly Review, Series Leaderboard, Goal Pacing, Experiments
- Dark theme, goal banner at top, inline bar charts
- Data: Apr 1–16 daily, 3 complete weeks + partial current week, series breakdown
