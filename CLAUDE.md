# Jeff Analytics — Claude Code context

You are helping the sales lead run outbound analytics for two businesses on the Jeff platform.

## Business context

Two businesses, one platform:

- **Jeff** — an outbound sales platform that runs cold email campaigns via Smartlead for Amazon marketing agencies. Multiple clients.
- **MerchantBots** — our own Amazon marketing agency. Commission-only, 4–6% of gross sales, targets early-stage sellers under $30,000/month revenue. Jeff is the outbound engine powering MerchantBots sales.

Strategic direction: phasing out external Jeff clients over time, scaling MerchantBots. Eventually Jeff becomes MerchantBots' internal sales tool.

## Client list

### MerchantBots clients (primary focus)

| Client | ID | Role |
|---|---|---|
| Equal Collective | 108917 | Parent company — IS MerchantBots |
| DTC Retail | 82914 | Originated the MerchantBots model |
| Nexus | 51431 | Infrastructure partner — sends volume, no meetings |
| Mr. Prime | 33748 | Infrastructure partner — sends volume, no meetings. Being phased out. |

**Query filter for MerchantBots:** `client_id=108917,82914,51431,33748`

Infrastructure partners (Nexus, Mr. Prime) contribute to volume metrics (prospects_reached, emails_sent, replies) but naturally return 0 for outcome metrics (meetings, positive replies). No need for separate queries.

### Jeff clients (secondary tracking)

| Client | ID | Goal |
|---|---|---|
| AMZ Ads | 25946 | 3 meetings/week |
| Riverguide | 25948 | 3 meetings/week |
| Accelo Brand | 223329 | 3 meetings/week |
| The Alfi | 340115 | 3 meetings/week |

**Query filter for Jeff clients:** `client_id=25946,25948,223329,340115`

### Excluded

| Client | ID | Reason |
|---|---|---|
| Keywords | 223328 | Not a Jeff-analytics client, doing their own thing |

### When the client list changes

The user will inform Claude when clients are added, removed, or reclassified. Update the table above and the query filters accordingly.

## Goals

- **MerchantBots: 30 meetings/week and 100 positive replies/week.** These are the primary goals. Goal pacing banner always shows both.
- **Jeff clients: 3 meetings/week per client** (12 total across 4 clients). Secondary tracking in the Jeff Clients tab.

## MCP tools available

The **Jeff Analytics MCP** exposes the outbound funnel:
- `smartlead_metric_definitions` — full catalog of metrics, dimensions, filters. Call this FIRST in any session where you're unsure what's queryable.
- `smartlead_query_funnel` — the workhorse. CSV output. Supports grouping and filtering.
- `smartlead_list_clients`, `smartlead_list_campaigns`, `smartlead_list_domains` — for resolving IDs to names.

## The funnel

```
prospects_reached
   ↓  (deliverability)
emails_delivered
   ↓  (engagement — reply_rate is the current proxy, will be refined later)
crm_positive_replies  (CRM is source of truth for outcomes)
   ↓  (conversion)
crm_meetings_booked   (CRM is source of truth for outcomes)
```

### Source of truth rules
- **CRM metrics are always the primary metrics** for positive replies and meetings booked. Use `crm_positive_replies` and `crm_meetings_booked` in all dashboards and analyses.
- **SL metrics (`sl_positive_replies`, `sl_meetings_booked`) exist for QA purposes only.** Their job is to cross-check that CRM is syncing correctly. If SL and CRM numbers diverge significantly, flag it as a sync issue.
- **Reply rate** (`reply_rate`) is the current indicator of deliverability at the emails_delivered step. This will eventually be replaced with a more specific deliverability-focused reply metric — when the MCP is updated, swap it in.

### Key rates
- `crm_booking_rate` = crm_meetings_booked / prospects_reached. End-to-end conversion. **This is the north star.**
- `crm_positive_reply_rate` = crm_positive_replies / prospects_reached. Signal quality.
- `reply_rate` = total_replies / prospects_reached. Deliverability proxy (to be refined).


## Dimensions that matter

- `series` / `sub_series` — different campaign archetypes. **Series 13 is currently the breakout** (~10x booking rate vs. others as of week of Apr 6). Always segment by series when analyzing.
- `client_id` — the Smartlead client running the campaign.
- `domain` — sending domain. Matters for deliverability debugging.
- `sequence_number` — which step of the email sequence. Useful for deciding if follow-ups are earning their keep.

## Date modes — don't get these wrong

- `activity` (default): each event bucketed by its own timestamp. Use for "what happened on day X".
- `cohort`: all downstream events bucketed by the lead's email_1 send date. Use for "of leads we contacted in week X, how many eventually converted". **Cohort mode is the right call when measuring conversion rate trends** because it avoids the lag artifact where this week's positive replies are mostly from last week's sends.

## Granularity rules
- `daily` — any dates
- `weekly` — date_from must be Monday, date_to must be Sunday
- `monthly` — date_from must be 1st, date_to must be last day

## How to work with the user

- **Run the query first, then talk.** Don't ask permission for read-only queries. Just pull the data.
- **Surface anomalies unprompted.** If one series is 10x others, say so even if not asked.
- **Always contrast.** Every number is meaningless without a comparison — vs. last week, vs. other series, vs. goal.
- **Be direct about broken data.** PostHog metrics are 0 — when you see that, call it out.
- **No apologies, no hedging.** "The booking rate dropped" not "it appears the booking rate may have dropped".

## Dashboard

### Hosting and update workflow
- **GitHub repo**: `Aryansethi-25/jeff-analytics-dashboard-v1` (public)
- **Team URL**: `https://aryansethi-25.github.io/jeff-analytics-dashboard-v1/dashboard.html`
- **Permanent file**: `dashboard.html` at the repo root — this is what the team sees. Always overwrite this file with the latest data.
- **Archive**: `dashboards/YYYY-MM-DD-vN.html` — dated snapshot copy. Create one each time the dashboard is updated.

### Update process ("update dashboard")
1. Pull fresh data from MCP for all sections per the specs below.
2. Overwrite `dashboard.html` with the new data.
3. Save an identical copy in `dashboards/` with today's date and version.
4. Git add, commit, and push to `origin/main`.
5. GitHub Pages auto-deploys within ~1 minute. Team refreshes to see new data.

When the user says "show analytics" or similar vague requests, run the queries described below and present the data conversationally.

### Dashboard sections

Each section below defines exactly what to query and how to display it. When building or updating the dashboard, follow these specs precisely.

#### Section 1: Goal Pacing Banner (always visible at top, MerchantBots only)
- **Query**:
  - Metrics: `crm_meetings_booked,crm_positive_replies`
  - Granularity: `daily`
  - Client IDs: `108917,82914,51431,33748`
  - Dates: current week Monday through today
  - Date mode: `activity`
- **Display**:
  - Two large numbers side by side: **Meetings This Week** and **Positive Replies This Week** (sum across all days returned)
  - Progress bar under meetings showing progress toward 30/week goal
  - Progress bar under positive replies showing progress toward 100/week goal
  - Status badge on each: ahead (green) / on pace (amber) / behind (red) based on where the count stands relative to the day of the week

#### Section 2: Daily Trend (tab, MerchantBots, activity mode only)

##### Sub-section A: Yesterday's Snapshot
- **Query**:
  - Metrics: `prospects_reached,total_emails_sent,total_replies,crm_positive_replies,crm_meetings_booked`
  - Granularity: `daily`
  - Client IDs: `108917,82914,51431,33748`
  - Dates: last 16 days through yesterday (for KPI cards, show yesterday's numbers compared to 15-day sending-day average)
  - Date mode: `activity`
- **Display**:
  - 5 KPI cards: Prospects Reached, Emails Sent, Total Replies, Positive Replies, Meetings Booked
  - Each card shows yesterday's number and comparison to 15-day sending-day average
  - Non-sending day definition: any day where prospects_reached < 100. Exclude these from the average calculation.
  - Funnel bar chart for yesterday: Prospects → Replies (with reply rate) → Positive Replies (with % of prospects) → Meetings (with booking rate)

##### Sub-section B: Daily Summary Table
- **Query**: same as sub-section A (reuse the data)
- **Display**:
  - 16 rows, **sorted most recent date at top**
  - Columns: Date, Day, Prospects Reached, Emails Sent, Total Replies, Positive Replies (with inline bar chart), Meetings, Reply Rate, Positive Reply Rate
  - No booking rate column
  - Inline bar chart on the Positive Replies column (scaled to max day)
  - Weekends muted
  - Highlight any day with positive replies >2x the median

##### Sub-section C: Sequence Step Breakdown
- **Query**:
  - Metrics: `email_1_sent,email_2_sent,email_3_sent,crm_positive_replies`
  - Granularity: `daily`
  - Client IDs: `108917,82914,51431,33748`
  - Dates: last 16 days through yesterday
  - Date mode: `activity`
- **Display**:
  - Table with one row per day, sorted most recent at top
  - Columns: Date, Email 1 Sent (absolute), Email 2 Sent (absolute), Email 3 Sent (absolute), Stacked % Bar (showing percentage split of email 1/2/3 with labels like "58% / 28% / 14%"), Positive Replies
  - The stacked bar shows the mix visually; the absolute numbers show the volume
  - Positive Replies column placed next to the stacked bar so the user can eyeball correlation between email 1 volume/mix and positive reply count
- **Business question answered**: "Are we sending enough email 1s? On days with higher email 1 mix, do we get more positive replies?"

##### Sub-section D: Deliverability Monitor
- **Query**:
  - Metrics: `email_1_sent,replies_email_1,bounce_rate,reply_rate`
  - Granularity: `daily`
  - Client IDs: `108917,82914,51431,33748`
  - Dates: last 16 days through yesterday
  - Date mode: `activity`
- **Display**:
  - Line graph trending over 16 days:
    - Line 1: Email 1 reply rate (`replies_email_1 / email_1_sent`) — primary deliverability signal
    - Line 2: Total reply rate — for comparison
    - Line 3: Bounce rate — delivery failures
  - X-axis: dates. Y-axis: percentage. Weekend days can be gaps or dotted lines.
- **Business question answered**: "Are my emails landing in inboxes? Is deliverability improving or degrading?"

#### Section 3: Weekly Review (tab, MerchantBots)

##### Sub-section A: Week-over-Week Table (toggle: activity / cohort mode)
- **Query (activity mode)**:
  - Metrics: `prospects_reached,total_emails_sent,total_replies,crm_positive_replies,crm_meetings_booked,reply_rate,crm_positive_reply_rate,crm_booking_rate`
  - Granularity: `weekly`
  - Client IDs: `108917,82914,51431,33748`
  - Dates: last 6 complete weeks (Mon–Sun) + current partial week
  - Date mode: `activity`
- **Query (cohort mode)**: same parameters but `date_mode=cohort`
- **Display**:
  - Toggle switch at top: Activity / Cohort. Swaps the entire table data.
  - Table with most recent week at top
  - Columns: Week, Prospects, Emails, Replies, Reply Rate, Positive Replies, Pos. Reply Rate, Meetings, Booking Rate, Pos. Reply to Booking Rate
  - Pos. Reply to Booking Rate = crm_meetings_booked / crm_positive_replies (calculated in HTML, not from API)
  - Label clearly which mode is active
- **Saturday rule**: when building on a Saturday, treat the current Mon–Fri as the most recent week (it is complete for sending purposes)

##### Sub-section B: Funnel Diagnostic (activity mode only)
- **Query**: reuse activity mode data from sub-section A (most recent week + 6-week averages)
- **Date mode**: `activity`
- **Display**:
  - Vertical funnel visualization (styled like the ICAP funnel image — colored boxes with arrows between them)
  - 4 steps top to bottom:
    1. **Prospects Reached** — this week's count vs 6-week avg count
    2. **Emails Delivered** — reply rate this week vs 6-week avg reply rate
    3. **Positive Replies** — positive reply rate this week vs 6-week avg
    4. **Meetings Booked** — booking rate this week vs 6-week avg
  - Each box shows: this week's rate, 6-week avg rate, and a status color:
    - Green: this week >= 6-week avg
    - Amber: this week is 10-30% below 6-week avg
    - Red: this week is >30% below 6-week avg
  - Also show Pos. Reply to Booking Rate as a label between steps 3 and 4
- **Business question answered**: "At a glance, where in the funnel did we win or lose this week?"

##### Sub-section C: Series Performance Table (activity mode)
- **Query**:
  - Metrics: `prospects_reached,total_emails_sent,total_replies,crm_positive_replies,crm_meetings_booked,reply_rate,crm_positive_reply_rate,crm_booking_rate`
  - Granularity: `weekly`
  - Client IDs: `108917,82914,51431,33748`
  - Group by: `series`
  - Dates: last 4 complete weeks (pull all 4 individually to support the toggle buttons)
  - Date mode: `activity`
- **Display**:
  - 4 toggle buttons: **Last 1 week** | **Last 2 weeks** | **Last 3 weeks** | **Last 4 weeks**
  - Each button aggregates the selected number of recent weeks into one row per series
  - Table columns: Series, Prospects, Emails, Replies, Reply Rate, Positive Replies, Pos. Reply Rate, Meetings, Booking Rate, Pos. Reply to Booking Rate
  - Pos. Reply to Booking Rate = crm_meetings_booked / crm_positive_replies (calculated in HTML)
  - Sorted by booking rate descending
  - Fade out series with 0 meetings in the selected range
- **Business question answered**: "Which series should I scale this week? Is the performance real or a fluke?"

##### Sub-section D: Series Trend Lines (cohort mode)
- **Query**:
  - Metrics: `crm_positive_reply_rate,crm_booking_rate,crm_positive_replies,crm_meetings_booked`
  - Granularity: `weekly`
  - Client IDs: `108917,82914,51431,33748`
  - Group by: `series`
  - Dates: last 6 complete weeks
  - Date mode: `cohort`
- **Display**:
  - 3 line charts, one for each rate:
    1. Booking Rate by series (weekly trend)
    2. Positive Reply Rate by series (weekly trend)
    3. Pos. Reply to Booking Rate by series (weekly trend, calculated in HTML)
  - Each line = one series, different color per series
  - Toggle buttons to show/hide individual series (all visible by default, click to turn off)
  - 6 data points per line (one per week)
  - Pos. Reply to Booking Rate = crm_meetings_booked / crm_positive_replies per series per week
- **Business question answered**: "How is each series trending over time? Is a series improving or degrading?"

#### Section 4: Jeff Clients (tab, activity mode only)

##### Sub-section A: This Week at a Glance
- **Query**:
  - Metrics: `prospects_reached,crm_positive_replies,crm_meetings_booked`
  - Granularity: `daily`
  - Client IDs: `25946,25948,223329,340115`
  - Group by: `client_id`
  - Dates: current week Monday through today
  - Date mode: `activity`
- **Display**:
  - Combined total at top: X / 12 meetings across all Jeff clients (progress bar)
  - One card per client (AMZ Ads, Riverguide, Accelo Brand, The Alfi)
  - Each card shows: meetings this week vs 3/week goal (progress bar), positive replies, prospects reached
  - Green if >= 3 meetings, red if 0, amber otherwise

##### Sub-section B: Combined Jeff Performance (last 4 weeks)
- **Query**:
  - Metrics: `prospects_reached,total_emails_sent,total_replies,crm_positive_replies,crm_meetings_booked,reply_rate,crm_positive_reply_rate,crm_booking_rate`
  - Granularity: `weekly`
  - Client IDs: `25946,25948,223329,340115`
  - Dates: last 4 complete weeks (Mon–Sun) + current partial week
  - Date mode: `activity`
- **Display**:
  - All Jeff clients clubbed into one aggregate row per week (not individual client rows)
  - Most recent week at top
  - Columns: Week, Prospects, Emails, Replies, Reply Rate, Positive Replies, Pos. Reply Rate, Meetings, Booking Rate, Pos. Reply to Booking Rate
  - Pos. Reply to Booking Rate = crm_meetings_booked / crm_positive_replies (calculated in HTML)
- **Business question answered**: "How is the Jeff book of business performing overall week over week?"

##### Sub-section C: Jeff Series Breakdown (activity mode)
- **Query**:
  - Metrics: `prospects_reached,total_emails_sent,total_replies,crm_positive_replies,crm_meetings_booked,reply_rate,crm_positive_reply_rate,crm_booking_rate`
  - Granularity: `weekly`
  - Client IDs: `25946,25948,223329,340115`
  - Group by: `series`
  - Dates: last 4 complete weeks
  - Date mode: `activity`
- **Display**:
  - 4 toggle buttons: **Last 1 week** | **Last 2 weeks** | **Last 3 weeks** | **Last 4 weeks** (aggregate selected weeks per series)
  - All Jeff clients combined — series 6 from AMZ Ads and series 6 from Riverguide merge into one "Series 6" row
  - Columns: Series, Prospects, Emails, Replies, Reply Rate, Positive Replies, Pos. Reply Rate, Meetings, Booking Rate, Pos. Reply to Booking Rate
  - Sorted by booking rate descending
  - Fade out series with 0 meetings
- **Business question answered**: "Which series is working across Jeff clients? Where should we shift volume?"


### Dashboard design rules
- Dark theme (bg: #0f1117, cards: #1a1d27)
- Self-contained HTML, no external dependencies
- Tabs for navigation, goal banner always visible
- Color coding: green = good/above target, red = bad/below target, amber = warning/watch
- All numbers formatted with commas. Rates as percentages with 2 decimal places.
- End each section with a callout highlighting the key insight or action item
