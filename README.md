# AI Marketing Reporting Workflow

## What it is

A design for an n8n workflow that pulls ad and analytics data, finds the changes that actually matter, and drafts human-approved insights and an executive report.

## Problem it solves

Marketing teams spend hours each week exporting data from Google Ads, Meta and GA4, reconciling naming conventions, and writing the same "here's what happened and why" summary by hand. A KPI table alone also doesn't say what changed, why, or what to do next.

## How it works

1. **Ingest** – scheduled API pulls from Google Ads, Meta Ads and GA4, plus a small config of KPI targets, campaign metadata and known events (launches, pauses, budget changes).
2. **Standardize and validate** – a data quality gate catches broken tags, zero-spend days and naming mismatches before anything is analyzed.
3. **Calculate KPIs** – all math is deterministic; the LLM never does arithmetic.
4. **Detect material changes** – thresholds and minimum-volume rules separate real movement from noise.
5. **AI interpretation** – an LLM explains only the validated, material changes and drafts recommendations.
6. **Score and route** – recommendations are prioritized and sent to a human for approval.
7. **Report** – approved insights go into an executive report; every step is written to an audit log.

## Files

| File | What it is |
|---|---|
| `marketing-reporting-agent-architecture.md` | Full architecture: workflow stages, n8n implementation, data model, KPI framework, change detection, AI guardrails, human-in-the-loop design, testing plan, and MVP vs. V2 roadmap |
| `LICENSE` | MIT license |

## Status

**Design spec.** Not yet built or deployed.

*Designed by me; drafted with Claude/ChatGPT.*
