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
│  │  Cyborg/OrgData (fallback) │  Workspace (/propose-risk) │     │
│  │  Slack (indexed search)    │  web_fetch                │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
         │                              │
         ▼                              ▼
┌─────────────────┐           ┌──────────────────────┐
│   Jira OCPBUGS  │           │ openshift/cincinnati- │
│                 │           │ graph-data            │
│ Labels = state  │           │                      │
│ Comments = log  │           │ blocked-edges/ YAML  │
│ Links = rel     │           │ /propose-risk cmd    │
└─────────────────┘           └──────────────────────┘
```

## Component → Project Routing

Deterministic config from 50+ historical Spikes, not AI guessing ([per Trevor's recommendation](https://github.com/openshift/enhancements/tree/master/enhancements/update/update-blocker-lifecycle)):

```
1. Curated config file (ota-component-mapping.yaml)    ← primary, data-driven
2. OrgData fuzzy lookup → filter out OCPBUGS/RFE       ← fallback for unknowns
3. Ask OTA Monitor manually                            ← last resort
4. Human confirms EVERY time                           ← always
```

Config includes Jira project + Slack channel per component. OrgData tested against 10 components — only 2/10 had clean mappings (OCPBUGS component names ≠ Cyborg names). Config is the reliable path.

See [`prompts/ota-component-mapping.yaml`](prompts/ota-component-mapping.yaml) for the full mapping.

## State Machine

```
UpgradeBlocker ──► ImpactStatement ──► ImpactStatement ──► UpdateRecommendations ──► fixedIn
(suspect)          Requested           Proposed             Blocked                   (done)
                   (waiting on team)   (OTA review)         (active blocker)

UpgradeBlocker label STAYS forever (additive per enhancement doc)
Only lifecycle labels swap (ADD next first, REMOVE previous — crash-safe)
```

**Classification priority** (first match wins): UpdateRecommendationsBlocked > ImpactStatementProposed > ImpactStatementRequested > UpgradeBlocker only

**Idempotency** (3-layer): linked issues (primary) > bot comments (fast path) > labels (tertiary)

## Four Skills

| Skill | Trigger | What It Does |
|-------|---------|-------------|
| **1. Monitor** | 10:00 + 22:00 UTC (weekdays) | Three-pass JQL filtering, 10 detection rules, pipeline FAILED check, daily status, heartbeat |
| **2. Lifecycle** | Button click | 4 gates + auto-actions. Spike creation, label transitions, orphan closure, risk extension |
| **3. PR Generator** | Gate 2/3/4 click | Multi-turn: workspace → /propose-risk → validate → PR from fork → post-merge cleanup |
| **4. Handover** | Friday 22:00 UTC | Two-phase: gather 9 sources → synthesize HTML artifact |

## Three Human Gates

| Gate | What You're Deciding |
|------|---------------------|
| **1. Create Spike** | "Is this the right team project?" |
| **2. Accept — Block Edge** | "Is this really a blocker?" (auto-triggers PR) |
| **3. Add FixedIn** | "Is this the right fix version?" |

Everything else is automatic.

## Schedule

| Time | What | Posts? |
|------|------|--------|
| **10:00 UTC** (weekdays) | Daily status + Jira triage + pipeline check | Always |
| **22:00 UTC** (weekdays) | Same checks | Only if action needed |
| **Fri 22:00 UTC** | Weekly handover (HTML from 9 sources) | Always |

12h apart to cover EU + US timezones. ~8 LLM calls/week. Weekdays only. All posts tag `@ota-monitor`.

## Key Design Decisions

| # | Decision | Why |
|---|----------|-----|
| 1 | 3 gates, not 5 | Approval fatigue is the #1 threat |
| 2 | Curated config for routing | OrgData tested — only 2/10 clean mappings. Deterministic config from real data. |
| 3 | UpgradeBlocker stays forever | Additive per enhancement doc |
| 4 | ADD label first, REMOVE second | Crash-safe label transitions |
| 5 | 3-layer idempotency | Linked issues survive deletion |
| 6 | Stateless runs | Jira IS the database |
| 7 | Two JQL queries | Secondary catches fixedIn on Closed bugs |
| 8 | /propose-risk for all YAML | Composition. Single source of truth. |
| 9 | Always Spike URL | Handles private bugs automatically |
| 10 | No guardian loop | Polling too expensive. Events or scan. |
| 11 | Chai Bot for infra | Solves creds, hosting, scheduling. AI earns its keep in later stages (YAML generation, PromQL, handover synthesis). |
| 12 | Human primary, bot safety net | ~8 LLM calls/week |

## Channels

- **#ota-monitor-bot** — all bot output (alerts, status, handover, button workflows)
- **#osus-graph-data-automation** — bot reads (indexed) for pipeline FAILED detection, never posts
- **#forum-ocp-updates** — indexed for OTA team context, bot never posts

## Repository Contents

```
├── index.html                         # Design doc (GitHub Pages)
├── onboarding.md                      # Ready-to-post #chai-users message
├── plan.md                            # Implementation plan summary
└── prompts/                           # Chai Bot prompt files
    ├── 01_role.md                     # Persona identity + rules
    ├── monitor-enriched.md            # 10:00 UTC task (10 detection rules)
    ├── monitor-brief.md               # 22:00 UTC task (silent if nothing)
    ├── weekly-handover.md             # Friday handover (9 data sources)
    └── ota-component-mapping.yaml     # Component → project + Slack routing
```

## Risks (all addressed)

| Risk | Mitigation |
|------|-----------|
| Gate fatigue | 5→3 gates, auto-transitions |
| Label conflicts | Priority ordering, first match wins |
| Slack fragility | Best-effort, Jira lifecycle unaffected |
| Button re-posting | \"offered\" markers |
| fixedIn JQL exclusion | Secondary JQL for Closed bugs |
| Crash safety | ADD-then-REMOVE label order |
| Early false positives | Week 1-2: human verifies all, pause if <90% |

## Status

- [x] Design reviewed 3x by RH Agentic SDLC persona
- [x] Gap analysis against 12 weeks of real RIT status docs (9 rotations, 8 monitors)
- [x] 4 prompt files + component mapping config written and reviewed
- [x] Channels created (#ota-monitor-bot, #test-ota-monitor-bot)
- [x] Onboarding message drafted
- [x] OrgData tested — routing validated, config built from 50+ historical Spikes
- [x] Trevor King reviewed — timezone, routing, and AI-value feedback incorporated
- [ ] Post onboarding in #chai-users
- [ ] Teach 8 Verified Knowledge lessons
- [ ] Create OTA-MONITOR-FEEDBACK Jira ticket
- [ ] Gradual rollout (weeks 1-2: read-only → 3-4: Spikes → 5-6: PRs → 7+: full)
"