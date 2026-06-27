# Widget Data-Wiring Spec

For each widget: **Source** (existing app helper, or new table to build), **Inputs**, and **Status** — `READY` (data already in app), `CAPTURE` (we log it but needs aggregation/endpoint), or `NEW` (needs a new table/feature).

Filters that apply to most widgets: `REP` (filter by `owner_id`/`rep_id`) and `RANGE` (date window: today / this week / this month / this quarter).

---

## Sales (rep activity is logged — confirmed)
| Widget | Source | Inputs | Status |
|---|---|---|---|
| `kpis` Activity totals | activity log + deals | count of calls, leads, proposals; sum won revenue, all in range | CAPTURE |
| `leaderboard` Rep leaderboard | per-rep aggregates | per rep: calls, leads, proposals, sits, won, revenue → close% = won/proposals | CAPTURE |
| `callsTrend` Calls by day | call log | count of calls grouped by day in range | CAPTURE |
| `sources` Leads by source | leads table | leads grouped by `source` | CAPTURE |
| `funnel` Sales funnel | leads + deals | leads → contacted → proposals → won counts | READY (deals) + CAPTURE (leads/contacted) |
| `speed` Speed-to-lead | leads table | avg(first_contact_at − created_at), minutes | CAPTURE (need timestamps) |

> Activity model suggestion: an `activities` table (`type` = call/lead/proposal/visit, `rep_id`, `company_id`, `created_at`, `source`, `outcome`). Most sales widgets are simple `group by` queries over it.

## Value drivers (PE / consultant lens)
| Widget | Source | Inputs | Status |
|---|---|---|---|
| `avgTicket` Avg ticket | jobs/contracts | avg(contract_total); split residential/commercial; 6-mo trend | READY-ish (avg of `estimate_total`/contract) |
| `grossMargin` Gross margin | job costing | (revenue − COGS) / revenue, blended; vs target | NEW (needs actual job costs) |
| `ebitda` EBITDA | finance/GL | EBITDA $ and margin; prior period | NEW (GL/QBO integration) |
| `revGrowth` Revenue growth | invoices/GL | revenue per month, 6-mo; % vs prior | READY (`financeSummary`) |
| `revPerCrew` Revenue per crew | revenue ÷ crews | revenue / active crew count; / tech count | READY + headcount |
| `marketing` Marketing efficiency | spend + jobs | spend, leads, won jobs → CAC, CPL, ROAS | NEW (ad-spend source / GHL) |
| `backlog` Backlog | jobs | sum(contract value) where signed & not complete; ÷ weekly throughput | READY (`companyJobs` by stage) |
| `recurring` Recurring revenue | membership plans | active members, MRR, churn | NEW (memberships table) |
| `goalPacing` Goal pacing | revenue vs target | period revenue / monthly target | READY + target setting |
| `concentration` Customer concentration | invoices | revenue share of top N customers | READY (`financeSummary` by account) |

## Growth / Capacity
| Widget | Source | Inputs | Status |
|---|---|---|---|
| `pipelineCoverage` Pipeline coverage | deals + target | open pipeline $ ÷ monthly target | READY (`companyDeals`) + target |
| `utilization` Crew utilization | schedule | booked hours ÷ available capacity, per crew | NEW (scheduling/time data) |

## People
| Widget | Source | Inputs | Status |
|---|---|---|---|
| `recruiting` Recruiting funnel | ATS/hiring | applicants → screened → interviewed → hired; open roles | NEW (hiring pipeline) |
| `turnover` Turnover & labor cost | HR/payroll | annualized turnover, labor cost % of revenue, avg tenure | NEW (HR/payroll) |
| `quota` Quota attainment | revenue + quotas | per rep: revenue ÷ quota | CAPTURE + quota field |

## Cash flow
| Widget | Source | Inputs | Status |
|---|---|---|---|
| `cash13` 13-week cash flow | bank/GL forecast | weekly projected cash balances (13) + safety floor | NEW (cash forecast / QBO) |
| `dso` Collections (DSO) | invoices + payments | days sales outstanding; target; prior | READY (invoice/payment dates) |
| `jobCostVariance` Job-cost variance | estimates vs actuals | avg(actual margin − estimated); # jobs over; $ leakage | NEW (job costing) |

## Strategy
| Widget | Source | Inputs | Status |
|---|---|---|---|
| `ltvCac` LTV : CAC | finance + marketing | lifetime gross profit ÷ CAC; payback | NEW (LTV model + spend) |
| `ownerScore` Owner-dependency score | manual/survey | scored checklist of functions running without owner | NEW (manual input / settings) |
| `serviceMix` Revenue by service line | invoices/jobs | revenue grouped by service line | READY (tag jobs with line) |

## EOS / Traction (mostly new tables — these are EOS objects)
| Widget | Source | Inputs | Status |
|---|---|---|---|
| `rocks` Quarterly Rocks | rocks table | rock, owner_id, quarter, % complete, status | NEW |
| `scorecard` Company scorecard | scorecard table | metric, owner, goal, weekly actual, status | NEW |
| `oneYearPlan` 1-Year Plan | annual plan | revenue/profit goal vs actual + measurables | NEW (+ ties to finance) |
| `l10pulse` L10 meeting pulse | meetings table | rating /10, attendance, on-time, streak | NEW |
| `issues` Issues list (IDS) | issues table | issue, priority, status, solved_at | NEW |
| `todos` To-dos (7-day) | todos table | todo, owner, done, meeting_id | NEW |
| `peopleAnalyzer` People Analyzer (GWC) | people table | per person: core-values rating, GWC (get/want/capacity) | NEW |
| `eosComponents` Components checkup | survey | 6 component scores (Vision/People/Data/Issues/Process/Traction) | NEW (periodic input) |
| `coreValues` Core values | settings | list of company core values | NEW (settings field) |

> Suggested EOS schema (one company, leadership team): `rocks`, `scorecard_metrics` (+ `scorecard_entries` weekly), `issues`, `todos`, `meetings` (L10 ratings), `people_gwc`, `eos_checkups`, and a `core_values` / `vto` settings blob. All `company_id`-scoped with the same RLS pattern as the rest of the app.

## Operations / Finance / Reputation
| Widget | Source | Inputs | Status |
|---|---|---|---|
| `jobs` Jobs by stage | jobs | count by production stage | READY (`companyJobs`) |
| `dispatch` Today's dispatch | jobs + crews + schedule | today's jobs ↔ crew assignment + status | CAPTURE (needs crew assignment) |
| `weather` 5-day forecast | weather API | forecast for company location; flag rain/wind | NEW (weather API) |
| `revenue` Revenue & receivables | finance | invoiced, collected, AR aging buckets | READY (`financeSummary`) |
| `reviews` Reviews & reputation | reviews integration | rating, count, new this week, requests sent | NEW (Google/GHL reviews) |

---

## Build-order recommendation
1. **READY widgets first** — they wire to existing helpers with no new data (jobs, revenue, pipeline coverage, backlog, funnel-from-deals, concentration, avg ticket, goal pacing, rev growth).
2. **CAPTURE next** — stand up the `activities` table so the whole Sales group + leaderboard light up (this is the headline ask).
3. **NEW by ROI** — EOS tables (high owner value, simple CRUD), then job-costing (unlocks gross margin / variance / EBITDA), then memberships, then external integrations (weather, reviews, ad spend, QBO cash).
