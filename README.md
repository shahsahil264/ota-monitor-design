# OTA Monitor — Agentic System

An agentic system that automates the weekly OTA Monitor RIT rotation role using [Chai Bot](https://redhat-chai-bot.github.io/guide.html) as the primary platform.

**Design doc:** [shahsahil264.github.io/ota-monitor-design](https://shahsahil264.github.io/ota-monitor-design/)

## What It Does

The OTA Monitor watches the Cincinnati update pipeline and drives UpgradeBlocker bugs through a 4-stage Jira lifecycle. Today this is ~1-2 hours/day of manual work. This system reduces it to **~5 minutes/day of button clicks**.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     CHAI BOT PERSONA (ota_monitor)                │
│                                                                  │
│  SCHEDULED                          BUTTON-DRIVEN                │
│  ┌────────────────┐                 ┌──────────────────────┐     │
│  │ 10:00 UTC      │── alerts ──►   │ Gate 1: Create Spike │     │
│  │ 22:00 UTC      │  + buttons     │ Gate 2: Accept+PR    │     │
│  │ Fri: Handover  │                │ Gate 3: Add FixedIn  │     │
│  └───────┬────────┘                │ Gate 4: Extend Risk  │     │
│          │                         │ Auto: Close Spike    │     │
│          │                         └──────────┬───────────┘     │
│  ┌───────▼────────────────────────────────────▼───────────┐     │
│  │                    TOOLS                                │     │
│  │  Jira (read/write/create)  │  GitHub (fork-based PRs)  │     │
│  │  Cyborg/OrgData (primary)  │  Workspace (/propose-risk) │     │
│  │  Slack (indexed search)    │  web_fetch (general_dev)  │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

## Component → Project Routing

**Cyborg is the primary source** — official, team-maintained, pivots when teams update preferences.

```
1. Override config (4 entries — ALL temporary, pending Cyborg MRs)
2. Cyborg jira-OCPBUGS-* lookup + filter meta-projects
3. Parent component fallback
4. Ask OTA Monitor manually
5. Human confirms EVERY time
```

Overrides pending elimination:
- **Cloud Compute / Azure** → OCPCLOUD — [MR !1144](https://gitlab.cee.redhat.com/hybrid-platforms/org/-/merge_requests/1144) pending
- **Monitoring** → MON — [MR !1153](https://gitlab.cee.redhat.com/hybrid-platforms/org/-/merge_requests/1153) pending
- **Logging** → LOG — MR !1153 pending
- **Distributed Tracing** → TRACING — MR !1153 pending

Once both MRs merge → **zero overrides**. Fully Cyborg-driven.

## State Machine

```
UpgradeBlocker ──► ISRequested ──► ISProposed ──► UpdateRecBlocked ──► fixedIn
                                                                        (done)
UpgradeBlocker stays forever. Lifecycle labels swap (ADD first, REMOVE second).
```

## Three Human Gates

| Gate | What You're Deciding |
|------|---------------------|
| **1. Create Spike** | \"Is this the right team project?\" |
| **2. Accept — Block Edge** | \"Is this really a blocker?\" (auto-triggers PR) |
| **3. Add FixedIn** | \"Is this the right fix version?\" (bot verifies graph-data YAML first) |

Everything else is automatic.

## Schedule

| Time | What | Posts? |
|------|------|--------|
| **10:00 UTC** (weekdays) | Daily status + Jira triage + pipeline check (two-phase: semantic search + keyword grep) | Always |
| **22:00 UTC** (weekdays) | Same checks | Only if action needed |
| **Fri 22:00 UTC** | Weekly handover (HTML from 9 sources) | Always |

12h apart to cover EU + US timezones. ~8 LLM calls/week. All posts tag `@ota-monitor`. Standard `general_dev` workspace (Go + Python 3 included).

## Where AI Earns Its Keep

| Area | AI? | Why |
|------|-----|-----|
| JQL → Slack | No | Chai Bot solves infra, not intelligence. |
| Component routing | No | Cyborg config (deterministic). |
| YAML generation | **Yes** | /propose-risk: from regex, matchingRules. |
| PromQL patterns | **Yes** | Suggest from existing blocked-edges. Human reviews. |
| Handover synthesis | **Yes** | 9 data sources → structured HTML. |

## Rollout

**Live from day 1.** Deploy direct to #ota-monitor-bot. All features available. Human controls pace.

| Day | What |
|-----|------|
| **Day 1** | Verify alerts match triage dashboard |
| **Day 2** | Start clicking [Create Spike] |
| **Day 3** | Start clicking [Accept—Block Edge] |
| **Day 4** | [FixedIn], [Extend Risk] when they come up |
| **Day 5** | Friday handover auto-generates. Done. |

## Channels

- **#ota-monitor-bot** — all bot output
- **#osus-graph-data-automation** — bot reads (indexed), never posts
- **#forum-ocp-updates** — indexed for context, bot never posts

## Feedback

Bot logs [Skip] clicks, manual overrides, and missed detections to [OTA-2104](https://redhat.atlassian.net/browse/OTA-2104). Weekly handover includes bot performance metrics from this ticket.

## Repository Contents

```
├── index.html                         # Design doc (GitHub Pages)
├── PROJECT-CONTEXT.md                 # Living status doc — architecture, every fix, every open issue
├── onboarding.md                      # #chai-users onboarding message
├── plan.md                            # Implementation plan summary
└── prompts/
    ├── 01_role.md                     # Persona identity + rules
    ├── monitor-enriched.md            # 10:00 UTC task (10 detection rules)
    ├── monitor-brief.md               # 22:00 UTC task (silent if nothing)
    ├── weekly-handover.md             # Friday handover (9 sources, reads OTA-2104)
    └── ota-component-mapping.yaml     # 4 temp overrides (zero after MRs merge)
```

## Status

**This section is a point-in-time snapshot as of early rollout. For current, actively-maintained status — every fix, every open issue, and full root-cause detail — see [PROJECT-CONTEXT.md](PROJECT-CONTEXT.md), which is the living source of truth for this project.**

- [x] Design reviewed 4x by RH Agentic SDLC persona
- [x] Gap analysis against 12 weeks of real RIT status docs
- [x] Trevor King (OTA SME) reviewed and approved
- [x] Cyborg routing verified — 4 MRs submitted to eliminate all overrides
- [x] 5 prompt files written, reviewed, posted to #chai-users
- [x] Onboarding posted in #chai-users
- [x] #ota-monitor-bot channel created
- [x] Jira permissions verified (all 6 target projects)
- [x] Feedback ticket created ([OTA-2104](https://redhat.atlassian.net/browse/OTA-2104))
- [x] Workspace confirmed: standard general_dev (no custom env needed)
- [x] Chai Bot team deploys persona (live Aug 13, 2026)
- [x] Snowflake credentials fixed (config override was clobbering inherited creds)
- [x] @ota-monitor subteam mention working (<!subteam^STE7S7ZU2|@ota-monitor> syntax)
- [x] Prompt file PRs deployed: #384, #417, #519, #538 (routing, Spike format, matchingRules mapping, assignee collection)
- [x] Edge case audit: 6 fixes (re-alerting, phantom FAILEDs, manual Spike stale, sync lag, clone orphans, missing PRs)
- [x] **Label-update bug — RESOLVED (PR #522).** Bot couldn't update labels on any bug it didn't create itself — blocked every UpgradeBlocker detection in production. Root-caused to a hardcoded parameter in shared Jira authorization code (not a config issue, despite initial appearances); fixed by the platform team with a per-project allowlist (`allow_full_edit: [OCPBUGS, OTA]`), a tighter design than what we'd originally proposed.
- [x] **Spike assignee collection — RESOLVED, then hardened (PR #538 → #635).** The bot can't see Jira assignees (PII stripped by design), so it now asks the human. First version asked for a literal email address, which violated the platform's own PII-solicitation policy and got blocked by the response evaluator in production; corrected to ask for a Slack @-mention instead, resolved via `resolve_slack_user`.
- [ ] **Pipeline "data unavailable" in scheduled scans — still open, unresolved since launch.** The channel is confirmed public and indexed, and returns real results when queried interactively — but scheduled-task runs still report "data unavailable" every time. This is NOT fixed by the two-phase semantic+keyword pipeline check design (that logic works correctly when it can reach the data at all); the actual failure is upstream of it, in how the scheduled-context search differs from interactive search. Needs platform-side investigation.
- [ ] Public Spike visibility at Gate 2 acceptance — filed as a feature request (needs a new `set_security_level` Jira tool); no ETA.
- [ ] Cyborg MRs merge (!1144 + !1153) → zero component-routing overrides
- [ ] Teach remaining Verified Knowledge lessons
"