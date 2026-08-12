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
│  │  Slack (indexed search)    │  web_fetch                │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

## Component → Project Routing

**Cyborg is the primary source** — official, team-maintained, pivots when teams update preferences.

```
1. Override config (2 entries only)          ← Cloud Compute/Azure, Monitoring
2. Cyborg jira-OCPBUGS-* lookup + filter    ← primary, 9/11 clean
3. Parent component fallback                ← safe, same team
4. Ask OTA Monitor manually                 ← last resort
5. Human confirms EVERY time               ← always
```

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

12h apart to cover EU + US timezones. ~8 LLM calls/week. All posts tag `@ota-monitor`.

## Where AI Earns Its Keep

| Area | AI? | Why |
|------|-----|-----|
| JQL → Slack | No | Chai Bot solves infra, not intelligence. |
| Component routing | No | Cyborg config (deterministic). |
| YAML generation | **Yes** | /propose-risk: from regex, matchingRules. |
| PromQL patterns | **Yes** | Suggest from existing blocked-edges. Human reviews. |
| Handover synthesis | **Yes** | 9 data sources → structured HTML. |

## Rollout

**Live from day 1.** All features available. Human controls pace by which buttons they click.

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

## Repository Contents

```
├── index.html                         # Design doc (GitHub Pages)
├── onboarding.md                      # Ready-to-post #chai-users message
├── plan.md                            # Implementation plan summary
└── prompts/
    ├── 01_role.md                     # Persona identity + rules
    ├── monitor-enriched.md            # 10:00 UTC task (10 detection rules)
    ├── monitor-brief.md               # 22:00 UTC task (silent if nothing)
    ├── weekly-handover.md             # Friday handover (9 data sources)
    └── ota-component-mapping.yaml     # 2-entry override (Cyborg is primary)
```

## Status

- [x] Design reviewed 4x by RH Agentic SDLC persona
- [x] Gap analysis against 12 weeks of real RIT status docs
- [x] Trevor King (OTA SME) reviewed and approved
- [x] Cyborg/OrgData verified via 3 personas — overrides minimized to 2
- [x] Azure Cyborg MR submitted (eliminates 1 override)
- [x] 4 prompt files + override config written and reviewed
- [x] #ota-monitor-bot channel created
- [x] Jira create permissions verified (all 6 target projects)
- [ ] Post onboarding in #chai-users
- [ ] Teach 8 Verified Knowledge lessons
- [ ] Create OTA-MONITOR-FEEDBACK Jira ticket
- [ ] Deploy and go live
"