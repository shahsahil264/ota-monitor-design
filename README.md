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

**Cyborg is the primary source** — official, team-maintained, pivots when teams update preferences ([per Trevor King's recommendation](https://redhat-internal.slack.com/archives/CEGKQ43CP)).

```
1. Override config (2 entries only — verified wrong/missing)   ← Cloud Compute/Azure, Monitoring
2. Cyborg jira-OCPBUGS-* lookup → filter meta-projects        ← primary, 9/11 components clean
3. Parent component fallback                                   ← safe, same team
4. Ask OTA Monitor manually                                   ← last resort
5. Human confirms EVERY time                                  ← always
```

Only 2 overrides needed:
- **Cloud Compute / Azure** → OCPCLOUD (not in Cyborg — MR to add pending)
- **Monitoring** → MON (ambiguous: MON and COO are both `types: main`)

See [`prompts/ota-component-mapping.yaml`](prompts/ota-component-mapping.yaml) for overrides and verified results.

## State Machine

```
UpgradeBlocker ──► ImpactStatement ──► ImpactStatement ──► UpdateRecommendations ──► fixedIn
(suspect)          Requested           Proposed             Blocked                   (done)

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
| JQL → Slack | No | Deterministic. Chai Bot solves infra (creds, hosting), not intelligence. |
| Component routing | No | Cyborg config (deterministic, team-maintained). |
| Label transitions | No | Rule-based. |
| YAML generation | **Yes** | /propose-risk: from regex, matchingRules, file structure. |
| PromQL patterns | **Yes** | Suggest from existing blocked-edges. Human always reviews. |
| Handover synthesis | **Yes** | 9 data sources → structured HTML. |
| Pipeline FAILED classification | **Yes** | Stale branch vs risk extension vs race condition. |

## Key Design Decisions

| # | Decision | Why |
|---|----------|-----|
| 1 | 3 gates, not 5 | Approval fatigue is the #1 threat |
| 2 | Cyborg-primary routing | Official data source. Teams maintain it. 2-entry override only. |
| 3 | UpgradeBlocker stays forever | Additive per enhancement doc |
| 4 | ADD label first, REMOVE second | Crash-safe label transitions |
| 5 | 3-layer idempotency | Linked issues survive deletion |
| 6 | Stateless runs | Jira IS the database |
| 7 | Two JQL queries | Secondary catches fixedIn on Closed bugs |
| 8 | /propose-risk for all YAML | Composition. Single source of truth. |
| 9 | Always Spike URL | Handles private bugs automatically |
| 10 | No guardian loop | Polling too expensive. Events or scan. |
| 11 | Chai Bot for infra | Solves creds, hosting, scheduling. AI for later stages. |
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
└── prompts/
    ├── 01_role.md                     # Persona identity + rules
    ├── monitor-enriched.md            # 10:00 UTC task (10 detection rules)
    ├── monitor-brief.md               # 22:00 UTC task (silent if nothing)
    ├── weekly-handover.md             # Friday handover (9 data sources)
    └── ota-component-mapping.yaml     # 2-entry override config (Cyborg is primary)
```

## Status

- [x] Design reviewed 3x by RH Agentic SDLC persona
- [x] Gap analysis against 12 weeks of real RIT status docs
- [x] Trevor King reviewed — Cyborg-primary routing, timezone, AI-value feedback incorporated
- [x] Cyborg/OrgData verified via 3 personas — routing tested, overrides minimized to 2
- [x] 4 prompt files + override config written and reviewed
- [x] Channels created (#ota-monitor-bot, #test-ota-monitor-bot)
- [x] Jira create permissions verified (all 6 target projects)
- [ ] Post onboarding in #chai-users
- [ ] Teach 8 Verified Knowledge lessons
- [ ] Create OTA-MONITOR-FEEDBACK Jira ticket
- [ ] Gradual rollout (weeks 1-2: read-only → 3-4: Spikes → 5-6: PRs → 7+: full)
"