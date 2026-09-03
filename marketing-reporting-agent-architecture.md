# AI Marketing Reporting Agent — Workflow Architecture & Process Design

*Prepared as a portfolio-grade system design for an AI-assisted marketing reporting workflow, orchestrated in n8n.*

---

## 1. Executive Summary

**What it does:** This system pulls campaign performance data from Google Ads, Meta Ads, and GA4 on a schedule, standardizes it, calculates KPIs, detects which changes are actually *material* (not just noisy), asks an LLM to explain the material changes and propose actions, scores those recommendations by priority, and routes them to a human for approval before anything reaches a stakeholder — all while writing every step to an auditable log.

**Who it's for:** A marketing operations person, growth marketer, or small agency team who currently spends hours each week pulling exports, reconciling naming conventions, and writing the same "here's what happened and why" summary by hand.

**Problem it solves:** Manual, repetitive, error-prone reporting work — and more specifically, the gap between "here is a KPI table" and "here is what changed, why, and what to do about it."

**What enters the system:** Scheduled API pulls from ad platforms and GA4, plus a small human-maintained config (KPI targets, campaign metadata, known events like launches/pauses/budget changes).

**What leaves the system:** A validated KPI dataset, a set of human-approved insights and recommendations, an executive report, and a full audit trail — never an automatically-executed marketing change.

**Where AI is used:** Interpreting *validated, pre-filtered* material changes into plain-language hypotheses, generating draft recommendations, and writing the executive narrative. The LLM never does arithmetic, never sees raw unvalidated data, and never acts autonomously.

**Where humans stay involved:** Approving every recommendation before it's distributed, and making every decision that touches budget, campaigns, or client-facing communication. The system augments judgment; it doesn't replace it.

---

## 2. Reference Material: What the Bootcamp Gets Right (and Where It's Outdated)

I read all six lesson decks, the syllabus, and the reference images in the ZIP. A few things are worth naming explicitly since the prompt asked not to treat this as gospel.

**Principles worth keeping (they show up in the architecture below):**
- **Data → Information → Insight** (Lesson 3) is exactly the spine of this system: raw metrics are data, validated KPIs are information, and only the AI layer — working from validated information — produces insight.
- **Goal → Objectives → KPIs** (Lesson 3) is the backbone of Section 7's KPI framework.
- **Storytelling = Narrative + Visualization + Context** (Lesson 1) directly shapes the executive reporting layer: a KPI scorecard alone is visualization without narrative or context, which is the opposite of what the course argues for.
- The **metrics taxonomy image** (Awareness / Engagement / Conversion, plus Common Dimensions and Efficiency Metrics) is genuinely useful and is reused almost directly as the KPI hierarchy backbone in Section 7.
- **"Identify stakeholders, design around the audience"** (Lesson 6) drives the split between the executive layer and the analyst/operator layer in Section 13.
- The dashboard screenshot (DoD % change indicators, funnel view, a "Health Check" score) is a good visual precedent for the material-change and data-quality-gate concepts, even though it predates this project.

**Where the material is outdated or insufficient for this project:**
- The course still discusses **Universal Analytics alongside GA4** (the reference docx links a "GA/UA vs GA4" comparison). UA stopped processing data in mid-2023 and is fully retired now — GA4 is the only option, and this project should treat GA4 as the sole source with no UA fallback logic.
- Lesson 5 is built around **Google Data Studio**, which was rebranded **Looker Studio** in late 2022. Cosmetic, but worth naming so the portfolio doesn't read as stale.
- The GA4 screens in Lesson 2 reflect a **very early version of GA4** (the decks are copyright 2019–2020, and GA4 launched October 2020). Explorations, predictive metrics, and the reporting UI have all changed meaningfully since.
- **Nothing in the material covers API-based, programmatic access** to GA4 or Ads data — every lesson is GUI-driven ("how to click through the Analysis hub"). This project's entire premise (automated, API-based consolidation) is a step the bootcamp doesn't address at all.
- **No cross-platform methodology.** The course is Google-only (GA4, GTM, Looker Studio). It never discusses reconciling Meta Ads against Google Ads/GA4, which is a core requirement of Stage 2 below (different attribution windows, different conversion definitions, different currencies/timezones by ad account).
- **No treatment of data quality, anomaly detection, or statistical rigor.** "KPIs may change over time" (Lesson 3) is the closest the material gets to material-change detection — nowhere near what's needed for an automated system that has to decide, without a human in the loop yet, whether a change is worth surfacing.
- **No governance, human-in-the-loop, or AI-guardrail concepts** — unsurprising, since this predates LLM-based reporting agents entirely. Sections 10–14 below are built from scratch, not adapted from the course.

Net effect: the bootcamp material is a solid foundation for *what a KPI is* and *who a report is for*, and essentially silent on *how to build an automated, validated, AI-assisted pipeline*. That gap is the actual subject of this document.

---

## 3. What the System Should Actually Be

Before designing an "agent," it's worth being precise about what kind of system this is, because that word gets overused.

This is **a scheduled, deterministic data pipeline with one bounded LLM reasoning step in the middle**, gated by validation before the LLM and by human approval after it. It is not:
- An autonomous agent that decides what data to pull, when to run, or what actions to take.
- A multi-agent system where several LLMs negotiate or hand off to each other dynamically.
- A system that needs to "reason" about anything except the *content* of already-validated, already-flagged data.

Calling it an "AI Marketing Reporting Agent" is fine as a product name, but architecturally, the honest description is: **automation does the retrieval, transformation, and math; a single well-scoped LLM call does the interpretation; a human does the judgment.** That's a deliberate, defensible design choice — and stating it plainly is more credible to a technical hiring manager than dressing up a linear pipeline as "agentic."

Where it genuinely benefits from LLM reasoning: synthesizing multiple related signals into one coherent explanation, translating technical metrics into stakeholder language, and drafting hypotheses that require contextual judgment a fixed rule-set can't produce (e.g., "CPA rose *and* CTR fell *and* a budget increase happened three days ago" — a human-like read of that pattern is genuinely hard to hand-code, but easy to hand-code the underlying flags that feed it).

Where AI would be a mistake: computing percentage change, running the threshold check, deduplicating rows, formatting the Slack message, or deciding whether data quality passed. All of that is arithmetic and rules — LLMs are worse at it, slower, and non-auditable compared to a Code node.

---

## 4. Architecture Overview

```text
[Schedule Trigger]
        ↓
[Generate Run ID]
        ↓
 [Collect: Google Ads] [Collect: GA4] [Collect: Meta Ads*]
        ↓ (merge)
[Normalize Schema + Field Mapping]
        ↓
[Data Quality Gate]
   ├── HARD FAIL (auth broken, >X% rows missing) → [Alert + Log Run FAILED + Stop]
   ├── SOFT FAIL (minor nulls/dupes/timezone mismatch) → [Flag Caveats] → continue
   └── PASS → continue
        ↓
[Deduplicate + Store Raw Rows]  (tagged with run_id)
        ↓
[Calculate KPIs]  (CTR, CPC, CVR, CPA, ROAS, etc.)
        ↓
[Compare vs Prior Period + Rolling Baseline]
        ↓
[Material Change Filter]
   (% change, absolute change, min-volume floor, z-score if ≥8 periods of history,
    exclude last 1–2 days for conversion/attribution lag)
   ├── NO material change → [Standard KPI Report only]
   └── Material change found
        ↓
[Attach Context Flags]  (budget changes, launches/pauses, seasonality calendar, DQ caveats)
        ↓
[LLM Analysis Call]  → structured JSON: observation, evidence, hypothesis, confidence,
                        recommendation, requires_human_decision
        ↓
[Validate AI Output Schema]
   ├── Invalid/incomplete → retry once → still invalid → [Flag for manual analyst review]
   └── Valid → continue
        ↓
[Priority Scoring]  (Impact × Confidence ÷ Effort → P0 / P1 / P2)
        ↓
[Route by Category]
   ├── Auto-logged only (non-material, informational)
   ├── Human Review (insights & recommendations) → [Slack Approval Block]
   │        ├── Approved/Edited → included in report
   │        ├── Rejected → logged with reason, excluded
   │        └── No response in 24h → reminder → escalation
   └── Human Decision Required (budget/campaign/client-facing) → separate high-visibility approval
        ↓
[Assemble Executive Report]  (Scorecard, Wins, Concerns, Decisions Needed, Supporting Data link)
        ↓
[Push data to Looker Studio source] + [Send Slack/Email digest]
        ↓
[Log full run: data, calculations, AI output, human decisions]  (audit trail, keyed by run_id)
```
*Meta Ads is deferred to V2 (see Section 19) — the MVP runs on Google Ads + GA4 only.*

This diagram maps directly onto the n8n nodes in Section 5.

---

## 5. End-to-End Workflow (Stage Detail)

| Stage | Purpose | Input | Process | Output | Tool | Automation / AI / Human | Failure Mode | Recovery |
|---|---|---|---|---|---|---|---|---|
| 1. Trigger | Start the run on a schedule | — | Cron fires, run_id generated | run_id, timestamp | n8n Schedule Trigger | Automation | Trigger doesn't fire | Monitoring alert if no run in expected window |
| 2. Data Collection | Pull raw campaign data | API credentials, date range | Call Google Ads API + GA4 Data API (Meta in V2) | Raw JSON per platform | n8n HTTP/Google/Ads nodes | Automation | API auth failure, rate limit | Retry w/ backoff; skip source & flag partial run |
| 3. Normalization | Make platforms comparable | Raw JSON | Map fields to common schema, convert currency/timezone | Unified rows | n8n Code/Set nodes | Automation | Unmapped/new field | Log unmapped fields, don't silently drop |
| 4. Data Quality Validation | Prevent AI from analyzing bad data | Unified rows | Null/duplicate/volume/impossible-value checks | Pass/soft-fail/hard-fail verdict + caveats | n8n Code node (rules) | Automation | False positive/negative on checks | Manual review queue for hard fails |
| 5. KPI Calculation | Turn raw metrics into KPIs | Validated rows | Compute CTR, CPC, CVR, CPA, ROAS, etc. | KPI table | n8n Code node | Automation | Divide-by-zero, missing denominator | Return null + flag, not zero |
| 6. Performance Comparison | Establish what "normal" looks like | KPI table + history | Compare vs prior period + rolling avg | Delta table | n8n Code node / stored history | Automation | Insufficient history | Fall back to simple period-over-period only |
| 7. Material Change Detection | Filter signal from noise | Delta table | Apply thresholds, volume floor, z-score, lag exclusion | Flagged changes | n8n Code node | Automation | Threshold miscalibrated | Config table is human-editable, no redeploy needed |
| 8. Contextual Analysis | Attach "why this might be happening" flags | Flagged changes | Join against budget/status/seasonality logs | Enriched signal | n8n Merge/Code node | Automation | Context log incomplete | AI must say "insufficient context" rather than guess |
| 9. AI Insight Generation | Explain material changes | Enriched signal | LLM call, structured JSON output | Observation/hypothesis/confidence | Anthropic API (Claude) via n8n | AI | Hallucinated cause, malformed JSON | Schema validation + one retry, else route to analyst |
| 10. Recommendation Generation | Turn insight into action | Validated AI output | LLM proposes action w/ evidence, effort, urgency | Draft recommendation | Anthropic API (same call or chained) | AI | Vague/generic recommendation | Prompt requires evidence field to be non-empty |
| 11. Prioritization | Rank recommendations | Recommendations | Deterministic score = Impact × Confidence ÷ Effort | Priority tier (P0/P1/P2) | n8n Code node | Automation | Score doesn't match human intuition | Human can override tier at approval step |
| 12. Human Review | Judgment gate before distribution | Prioritized recommendations | Slack approval message | Approved/edited/rejected | Slack (n8n webhook) | Human | No response | Reminder at 24h, escalate at 48h |
| 13. Reporting Generation | Assemble stakeholder-ready output | Approved content + KPI data | Populate report structure | Executive report + analyst view | Looker Studio (reads from Sheets/DB) | Automation | Data source not refreshed | n8n confirms write success before marking run complete |
| 14. Distribution | Get the report to people | Final report | Send digest / share link | Delivered report | Slack/Email (n8n) | Automation | Delivery failure | Retry + log; report still viewable in Looker Studio directly |
| 15. Logging / Feedback | Auditability + improvement | Everything above | Write full run record | Audit log row | Google Sheets/DB | Automation | Log write failure | This is the one failure that halts the run — no silent gaps in the audit trail |

---

## 6. n8n Implementation

| Node | Type | Purpose | Deterministic or AI | Key config / notes |
|---|---|---|---|---|
| Schedule Trigger | Trigger | Weekly (or daily) kickoff | Deterministic | Cron; timezone pinned explicitly |
| Set Run ID | Function/Set | Unique ID for this run | Deterministic | `run_id = {{timestamp}}_{{uuid}}` |
| Google Ads node | Google Ads (native) | Pull campaign performance | Deterministic | OAuth, read-only scope, date range param |
| GA4 Data API (HTTP Request) | HTTP Request | Pull GA4 metrics/dimensions | Deterministic | Service account auth |
| Meta Ads (HTTP Request, V2) | HTTP Request | Pull Meta campaign data | Deterministic | Deferred to V2 |
| Merge | Merge | Combine platform outputs | Deterministic | Merge by date/campaign key |
| Normalize Schema | Code | Map to common field names, convert currency/TZ | Deterministic | Central mapping object, not per-node hardcoding |
| Data Quality Gate | Code + IF | Run validation rules, branch pass/soft/hard | Deterministic | See Section 9 checklist |
| Alert on Hard Fail | Slack/Email | Notify + halt | Deterministic | Includes run_id and specific failed check |
| Store Raw Rows | Google Sheets/Postgres | Persist validated raw data | Deterministic | Append-only, tagged with run_id |
| Calculate KPIs | Code | CTR, CPC, CVR, CPA, ROAS, etc. | Deterministic | Guard against divide-by-zero |
| Compare to Baseline | Code | vs. prior period + rolling avg | Deterministic | Reads stored history |
| Material Change Filter | Code + IF | Threshold/volume/z-score/lag logic | Deterministic | Config-driven, editable without redeploy |
| Attach Context | Code + Lookup | Join budget/status/seasonality log | Deterministic | Config table (Sheets/Airtable) |
| LLM Analysis | HTTP Request (Anthropic API) | Generate observation/hypothesis/recommendation | **AI** | Structured JSON output required; see Section 11 prompt |
| Validate AI Output | Code (JSON schema check) | Reject malformed/unsupported output | Deterministic | One retry on failure, then route to analyst |
| Priority Scoring | Code | Impact × Confidence ÷ Effort | Deterministic | Weights configurable |
| Route by Category | Switch | Auto-log / human review / human decision | Deterministic | See Section 14 |
| Slack Approval | Slack (interactive blocks) | Human approve/edit/reject | Human | Webhook callback back into n8n |
| No-Response Reminder | Wait + IF | Escalate stalled approvals | Deterministic | 24h reminder, 48h escalation |
| Assemble Report | Code/Set | Build executive + analyst payloads | Deterministic | Two payload shapes, one dataset |
| Write to Data Store | Google Sheets/Postgres | Feed Looker Studio | Deterministic | This is what Looker Studio reads — n8n never renders charts |
| Send Digest | Slack/Email | Distribute | Deterministic | Includes Looker Studio link |
| Log Run | Google Sheets/Postgres | Full audit record | Deterministic | The one step whose failure halts run-completion status |
| Error Workflow (global) | Error Trigger | Catch-all for any node failure | Deterministic | Separate n8n workflow subscribed to all others |

---

## 7. Data Architecture

**Raw / Performance data** (one row per platform/campaign/date, `run_id`-tagged)

| Field | Example |
|---|---|
| run_id, platform, campaign_id, campaign_name, date | — |
| spend, impressions, clicks, conversions, revenue | — |
| ctr, cpc, cvr, cpa, roas *(calculated)* | — |
| data_quality_flag | pass / soft_fail / hard_fail |

**Contextual / config data** (human-maintained, low write-frequency)

| Field | Example |
|---|---|
| campaign_id, business_objective, funnel_stage | e.g., "lead_gen", "consideration" |
| primary_kpi, kpi_target, benchmark_source | e.g., "CPA", 45.00, "Q2 plan" |
| budget_change_log | date, old_budget, new_budget |
| status_change_log | date, launched/paused/edited |
| seasonality_calendar | date ranges flagged (holidays, promos) |

**AI-generated data** (one row per material change, linked to run_id + campaign + KPI)

| Field | Example |
|---|---|
| run_id, campaign_id, kpi, detected_change | "+38% CPA vs 4-wk avg" |
| observation, evidence[] | plain-language + which fields supported it |
| hypothesis, confidence | "low / medium / high" |
| recommendation, expected_impact, effort, urgency | — |
| requires_human_decision (bool), human_decision, decided_by, decided_at | — |

Everything joins on `run_id`, so any number in the executive report can be traced back to the exact raw rows, calculation, AI call, and human decision that produced it.

---

## 8. KPI Framework

**Hierarchy:** Business Objective → Primary KPI → Secondary KPIs → Diagnostic Metrics. The system doesn't report every available metric — the config table in Section 7 defines which KPI is *primary* per campaign/objective, and only primary + secondary KPIs surface in the executive layer by default; diagnostics only appear when they're cited as evidence for a material change.

Built directly on the bootcamp's Awareness/Engagement/Conversion metrics taxonomy:

| Objective | Primary KPI | Secondary KPIs | Diagnostic Metrics |
|---|---|---|---|
| Awareness | Reach / Impressions | CPM, unique reach | Frequency, viewability |
| Traffic | Sessions / Clicks | CTR, CPC | Bounce rate, pages/session |
| Engagement | Engagement rate | Avg. engagement time, CTA clicks | Scroll depth, video completion |
| Lead Generation | Cost per Lead | Form conversion rate, MQL rate | Form abandonment, field drop-off |
| Ecommerce | ROAS / Revenue | Conversion rate, AOV | Cart abandonment rate, product-level CVR |
| Retention | Repeat purchase rate | Churn rate, LTV | Reactivation rate, email engagement |

Targets/benchmarks/baselines live in the config table (Section 7), not hardcoded in workflow logic — a marketer should be able to change a CPA target without touching n8n.

---

## 9. Material Change Detection

The framework deliberately never lets the AI declare a problem from a bare percentage. It moves through:

**Signal → Context → Interpretation → Confidence → Action**

**Deterministic layer (before any AI involvement):**
- % change vs. prior period **and** vs. a rolling N-period average (a single-period spike is noise; a sustained shift is signal)
- Absolute change threshold alongside percentage (CVR moving 0.5%→1% is a 100% relative move but negligible in absolute terms)
- Minimum-volume floor — below it, a change is marked "low volume, directional only" and excluded from AI analysis
- Z-score against rolling standard deviation, but only once ≥8 periods of history exist; otherwise fall back to simple thresholds
- Exclusion of the most recent 1–2 days from "material" scoring to account for conversion/attribution lag (shown as provisional in the standard report, never flagged as material)
- Known-event flags (budget change, launch, pause) pulled from the config log and attached to the signal *before* it reaches the AI

**AI layer receives only:** the signal (what changed, direction, magnitude), the context flags, and related metrics that moved alongside it (e.g., CPA up *and* CTR down *and* a budget increase three days prior). It is explicitly required to cite which supplied fields it used, and to output "insufficient evidence" rather than invent a cause when context flags are empty.

---

## 10. Data Quality & Validation Layer

A hard gate before anything reaches the AI. Checks include: missing data, duplicate records (hashed on platform+campaign+date), unexpected nulls, broken API connections, partial date ranges, sudden zeroes, impossible values (negative spend, CTR > 100%), currency/timezone mismatches, and unexpected volume swings (a 10x row-count change usually means a schema or account change, not real performance).

| Result | What happens |
|---|---|
| **Hard fail** (auth broken, >X% rows missing, impossible values on primary KPI) | Halt the pipeline for that source, alert immediately, mark the run partial, do not proceed to AI analysis for the affected campaigns |
| **Soft fail** (a few nulls, minor duplicate, timezone off by one) | Flag as a caveat that travels downstream — it must appear in the AI's context and, if relevant, in the executive report, never silently corrected and hidden |
| **Pass** | Proceed normally |

The AI is never allowed to analyze data flagged hard-fail, and any soft-fail caveats are passed into its prompt as known limitations it must acknowledge rather than paper over.

---

## 11. AI / Agent Architecture

**Option A — Single agent:** Overkill framing for what is actually a single-purpose reasoning call; "agent" implies autonomy this system doesn't need.

**Option B — Multi-agent system** (Data Quality Agent, Anomaly Agent, Insight Agent, Recommendation Agent, Executive Summary Agent): Adds orchestration complexity, latency, and cost with no clear benefit here, because the task decomposition is already fixed and linear — n8n nodes already do the "handoff" that multiple agents would otherwise negotiate. This is the option to explicitly avoid for the MVP, and to name as avoided in the portfolio write-up.

**Option C — Deterministic automation + a single bounded LLM reasoning layer:** ✅ Recommended. One LLM call (or two chained calls if a single prompt gets overloaded — analysis, then recommendation) that receives only validated, pre-filtered, context-enriched input and returns structured JSON. This is the right fit because the pipeline's stages are known in advance, don't require dynamic re-planning, and benefit from AI only at the interpretation step.

**Example system prompt (LLM Analysis node):**
```
You are a marketing performance analyst. You will be given a validated, pre-filtered
material change with supporting context. You must:
- Only use the fields provided. Never infer causes not evidenced by the input.
- Cite which specific fields led to your hypothesis.
- If context flags are empty or insufficient, say so explicitly instead of guessing.
- Distinguish observation (what happened) from hypothesis (why it might have happened).
- Assign confidence (low/medium/high) based on how directly the evidence supports the hypothesis.
- Output only valid JSON matching the provided schema. No prose outside the JSON.
```

A true agentic upgrade (an LLM that decides which follow-up data to pull when a signal is ambiguous) is a legitimate V3 idea — see Section 21 — but isn't justified for MVP.

---

## 12. AI Guardrails

Structure enforced in every AI call: **Observation → Evidence → Interpretation → Hypothesis → Confidence → Recommendation.**

- Only analyzes supplied, validated data — never raw or hard-fail-flagged data
- Required to cite the specific fields behind each observation
- Required to explicitly separate fact from hypothesis, and to state "insufficient data" rather than fabricate a cause
- Never recommends an action unsupported by the evidence fields it was given
- Output is schema-validated (Section 6, "Validate AI Output" node) before it can reach a human — malformed or unsupported-claim output triggers one retry, then routes to manual analyst review instead of silently passing through
- The AI never modifies raw data and never triggers a marketing action — its output is always a draft for a human decision

---

## 13. Recommendation Engine

Every recommendation carries: problem/opportunity, evidence, recommended action, why it matters, expected impact, effort, urgency, confidence, dependencies, and whether a human decision is required.

**Prioritization formula:**
```
Priority Score = (Impact × Confidence) ÷ Effort
```
Impact and Effort are 1–5 scales (AI-estimated but sanity-bounded by rules — e.g., confidence can only be "high" if the deterministic signal was large *and* context was present); Confidence is 0–1. Scores bucket into **P0 (act now)**, **P1 (this week)**, **P2 (backlog)**. Urgency is tracked as a separate axis for time-sensitive items (e.g., budget pacing near month-end) so a low-impact-but-urgent item doesn't get buried under a high-impact-but-not-time-sensitive one.

The AI is explicitly barred (via prompt + a lightweight keyword/specificity check in the validation node) from generic output like "optimize campaigns further" — a recommendation without a cited evidence field is rejected before reaching the human.

---

## 14. Executive Reporting Layer

**Executive layer:** Executive Summary, KPI Scorecard, Major Wins, Material Concerns, Key Insights, Recommended Actions, Items Requiring Decision. Short, narrative + context, no raw tables.

**Analyst/operator layer:** Full KPI detail, every detected change (even below the material threshold), raw data links, data-quality caveats, full audit trail. This is where "storytelling with analytics" (narrative + visualization + context) becomes concrete — the executive layer supplies narrative and context around the numbers the analyst layer already has.

---

## 15. Visualization Architecture

**n8n is the orchestration layer, not the visualization layer.** It writes structured rows to a data store (Google Sheets for MVP, Postgres/Supabase for V2+); **Looker Studio** (the current name for the tool the bootcamp calls Data Studio) reads from that store and renders the dashboards. n8n may generate the plain-text Slack digest directly — that's lightweight enough — but it should never hand-build an HTML dashboard; that reinvents charting badly. Airtable is a good fit specifically for the human-approval interface (kanban-style review), not as the primary time-series datastore.

---

## 16. Human-in-the-Loop Design

| Category | Examples | Mechanism |
|---|---|---|
| **Auto-approved** | Data pipeline runs, standard KPI reports with no material change, internal ops-channel posts | None — runs automatically |
| **Human Review Required** | AI-generated insights & recommendations before they reach the executive report | Slack interactive approval block; approve/edit/reject |
| **Human Decision Required** | Budget changes, campaign launches/pauses, strategic shifts, client-facing communication | Separate, higher-visibility approval; never auto-executed regardless of AI confidence |

No response within 24h triggers a reminder; 48h escalates to a backup approver. Rejections are logged with a reason field — this feedback becomes the input for Section 20's continuous-improvement loop.

---

## 17. Error Handling & Observability

| Failure | Handling |
|---|---|
| API fails | Retry with exponential backoff (n8n native); skip source and mark run partial rather than aborting entirely |
| Data incomplete | Soft/hard-fail branch per Section 10 |
| LLM call fails | Retry once; if still failing, route to manual analyst queue |
| AI output fails schema validation | Retry once; then manual review, never silently passed through |
| Dashboard/report can't update | Retry; alert if the write to the data store fails, since Looker Studio depends on it |
| Human doesn't approve | Reminder → escalation (Section 16); item remains "pending" in the audit log, not silently dropped |

Every run gets a `run_id`; a dedicated n8n **Error Trigger workflow** subscribes to failures across all other workflows and logs + alerts centrally, so failure handling isn't duplicated node-by-node.

---

## 18. Security & Data Governance

- Read-only OAuth scopes for Google Ads/GA4/Meta — this pipeline never needs write access to ad accounts
- Credentials live in n8n's credential store, not in workflow JSON or plaintext
- **No user-level PII should ever enter this pipeline** — only campaign-level aggregates. This is a deliberate design constraint, not an afterthought: nothing in the KPI framework requires user-level data
- LLM exposure is limited to aggregated, validated metrics and context flags — never raw exports
- Retention: granular daily data ~13 months, aggregated monthly indefinitely — proportional to an MVP, not enterprise infrastructure
- Full auditability comes from the `run_id`-linked schema in Section 7, not a separate governance system

---

## 19. Critical Gaps

**Critical:**
- No defined *source-of-truth* rule for conflicting attribution across platforms (Meta's own attribution vs. GA4's data-driven attribution vs. Google Ads' reported conversions will disagree — the system must document which one "wins" per metric or it will quietly erode trust)
- No sandbox/test-data strategy before connecting to live ad accounts

**Important:**
- No versioning of AI prompts — prompt changes alter report content and should be tracked like code
- No defined ownership of the KPI target/config table (who updates it, how often)

**Nice-to-have:**
- Multi-language reporting if stakeholders aren't all English-speaking
- An upgrade path toward a real anomaly-detection model (vs. threshold/z-score) for V3

---

## 20. Assumptions to Challenge

- **Is "AI agent" necessary?** No — this is a scheduled pipeline with one bounded reasoning step, not an autonomous agent. Naming it precisely is more credible than inflating it.
- **Is n8n the right orchestrator?** Yes for a no-code MVP — native OAuth connectors, visual explainability, built-in error workflows. Its weak points: long-running loops, complex branching state, and workflow JSON is painful to diff in git. Mitigation: export workflow JSON into a git repo anyway, to demonstrate version control in the portfolio.
- **Should n8n generate the report?** No — n8n feeds the data store; Looker Studio renders it. n8n may generate the plain-text Slack digest, not the dashboard.
- **Should anomaly detection happen before the LLM?** Yes, and it already does — this is the core guardrail against hallucinated causes and wasted LLM calls on noise.
- **Should recommendations come from raw data or validated insights?** Validated insights only. The LLM never sees unvalidated data.
- **Is multi-agent architecture unnecessary complexity here?** Yes, for the MVP. Revisit only if a single prompt starts genuinely underperforming on a fixed task list — an empirical decision, not a default.
- **What's the minimum viable version?** Two data sources, a handful of primary KPIs, Sheets as the datastore, simple threshold-based (no z-score) change detection, one LLM call, Slack approval, weekly cadence.
- **What makes this impressive for a portfolio?** Demonstrated judgment about *not* over-engineering, a real audit trail, honest failure-mode handling — not the number of integrations.

---

## 21. MVP vs. Version 2 vs. Advanced Version

**MVP:** Google Ads + GA4 only. 3–5 primary KPIs. Google Sheets as the datastore. Simple % + absolute + volume-floor threshold detection (no z-score). One LLM call producing structured JSON. Slack approval. Looker Studio reading from Sheets. Weekly cadence.

**Version 2:** Add Meta Ads. Add rolling-baseline + z-score detection once enough history exists. Migrate the datastore to Postgres/Supabase for reliability at scale. Add stakeholder-specific report variants (exec vs. analyst) generated automatically rather than manually split. Add prompt versioning.

**Advanced (V3):** Genuinely agentic capability only where justified — e.g., an LLM step that can request a specific follow-up query (e.g., "pull device-level breakdown for this campaign") when a signal is ambiguous, rather than only reasoning over what it was handed. Add a real anomaly-detection model. Add closed-loop feedback where rejected recommendations retrain prompt guidance over time.

---

## 22. Testing Plan

| # | Input | Expected Behavior | Expected Output | Failure Condition |
|---|---|---|---|---|
| 1 | Normal campaign performance | No material change detected | Standard KPI report only | AI called unnecessarily |
| 2 | Major CPA increase | Material change flagged, context attached | AI hypothesis + recommendation, human review | Cause invented without evidence |
| 3 | Major ROAS decrease | Same as above, cross-checked against related metrics | Coherent multi-metric explanation | Single-metric tunnel vision |
| 4 | Conversion tracking failure (sudden zeroes) | Data quality hard/soft fail | Alert, AI not run on affected campaign | AI analyzes broken data as if real |
| 5 | Missing platform data | Soft/hard fail per severity | Partial run, caveat flagged downstream | Silent gap in report |
| 6 | API failure | Retry w/ backoff, then skip + flag | Partial run, alert sent | Entire run aborts unnecessarily |
| 7 | Campaign launch | Context flag attached, change explained by launch | AI cites launch as likely cause | Launch ignored as context |
| 8 | Campaign pause | Same as above for a drop | Correct low-confidence-if-context-missing behavior | False anomaly on expected drop |
| 9 | Low-volume campaign | Below volume floor | Marked "directional only," excluded from AI | Noisy metric wrongly treated as material |
| 10 | Large seasonal fluctuation | Seasonality calendar flag attached | AI cites seasonality as likely cause | Seasonality misread as a real problem |
| 11 | Conflicting signals across platforms | Attribution source-of-truth rule applied | Report states which source governs and why | Contradictory numbers shown without explanation |
| 12 | AI recommendation unsupported by data | Schema/evidence validation catches it | Rejected, routed to manual review | Unsupported recommendation reaches a human as if valid |

---

## 23. Portfolio Presentation

Structure the case study as: Problem → Architecture (with the ASCII/n8n diagram) → Data flow → AI reasoning layer → **one real, walked-through example** (an actual anomaly, its evidence, the AI's hypothesis, the Slack approval, the final executive report) → business impact (even if framed as a credible counterfactual, e.g., "this would have surfaced the CPA spike two days before it was caught manually") → lessons learned → future improvements. Lead with a screenshot of the Slack approval and the executive report — that's the moment that shows human-in-the-loop judgment, which is the most credible part of the project.

---

## 24. Recommended Next Steps

1. Build the MVP scope (Section 21) end-to-end with two real, historical datasets — not synthetic happy-path data.
2. Document one case where the data-quality gate caught a real problem, not just successful runs.
3. Version-control the n8n workflow JSON and the LLM prompts in git with a changelog.
4. Write a short decision log explaining *why* each stage is deterministic vs. AI vs. human — this document is most of that log already.
5. Add the source-of-truth attribution rule (Section 19) before connecting Meta Ads in V2.

---

## Final Challenge: What Would a Hiring Manager Actually Look For?

Not the number of connected APIs. What separates a real AI Marketing Operations portfolio piece from "I connected ChatGPT to some marketing APIs":

1. **Restraint** — explicitly choosing not to build a multi-agent system, and explaining why.
2. **A real audit trail** — every number traceable to raw data, calculation, AI output, and human decision via `run_id`.
3. **Evidence of failure handling**, not just the happy path — a documented case where data quality caught something.
4. **Precision about what "AI" is doing** — a bounded reasoning step over validated data, not an autonomous agent.
5. **Human judgment preserved as the actual decision-maker**, with the system positioned as augmentation.

**Five modifications that would make this substantially more credible:**
- Run it against real historical data and show the actual output, not a mockup.
- Include a rejected-recommendation example alongside the approved ones.
- Put the n8n export and prompts in a public git repo with commit history.
- Publish the decision log (Section 20 of this doc, essentially) as its own artifact.
- Frame business impact in concrete, even if hypothetical, terms — time saved, or an issue caught earlier than manual review would have.
