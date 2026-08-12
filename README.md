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
| **3. Add FixedIn** | \"Is this the right fix version?\" |

Everything else is automatic.

## Schedule

| Time | What | Posts? |
|------|------|--------|
| **10:00 UTC** (weekdays) | Daily status + Jira triage + pipeline check | Always |
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
- [ ] Cyborg MRs merge (!1144 + !1153) → zero overrides
- [ ] Chai Bot team deploys persona
- [ ] Teach 8 Verified Knowledge lessons
- [ ] Go live
"