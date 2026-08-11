# OTA Monitor — Agentic System

An agentic system that automates the weekly OTA Monitor RIT rotation role using [Chai Bot](https://redhat-chai-bot.github.io/guide.html) as the primary platform.

**Design doc:** [shahsahil264.github.io/ota-monitor-design](https://shahsahil264.github.io/ota-monitor-design/)

## What It Does

The OTA Monitor watches the Cincinnati update pipeline and drives UpgradeBlocker bugs through a 4-stage Jira lifecycle. Today this is ~1-2 hours/day of manual work. This system reduces it to **~5 minutes/day of button clicks**.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        CHAI BOT PERSONA                          │
│                        (ota_monitor)                             │
│                                                                  │
│  SCHEDULED TASKS                    BUTTON-DRIVEN                │
│  ┌────────────────┐                 ┌──────────────────────┐     │
│  │ 9am: Enriched  │──── alerts ───►│ Gate 1: Create Spike │     │
│  │ 3pm: Brief     │    + buttons   │ Gate 2: Accept+PR    │     │
│  │ Fri: Handover  │                │ Gate 3: Add FixedIn  │     │
│  └───────┬────────┘                │ Gate 4: Extend Risk  │     │
│          │                         │ Auto: Close Spike    │     │
│          │                         └──────────┬───────────┘     │
│          │                                    │                  │
│  ┌───────▼────────────────────────────────────▼───────────┐     │
│  │                    TOOLS                                │     │
│  │  Jira (read/write/create)  │  GitHub (fork-based PRs)  │     │
│  │  Cyborg/OrgData            │  Workspace (/propose-risk) │     │
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

### Data Flow

```
#osus-graph-data-automation ──► Bot reads (indexed) for pipeline FAILEDs
                                Never posts here

Jira OCPBUGS ──────────────────► Bot queries via JQL (2 queries per run)
  Labels = state machine          Primary: active UpgradeBlocker bugs
  Bot comments = audit trail      Secondary: Closed bugs for fixedIn detection
  Linked issues = relationships

#ota-monitor-bot ◄──────────── ALL bot output goes here
                                Alerts, status, handover, button workflows
                                Heartbeat, errors, escalations
```

## State Machine

UpgradeBlocker bugs move through 4 lifecycle stages via Jira labels:

```
UpgradeBlocker ──► ImpactStatement ──► ImpactStatement ──► UpdateRecommendations ──► fixedIn
(suspect)          Requested           Proposed             Blocked                   (done)
                   (waiting on team)   (OTA review)         (active blocker)
     │                                      │
     │                                      └──► [Not a Blocker] ──► labels removed
     │
     └── UpgradeBlocker label STAYS forever (additive)
         Only lifecycle labels swap (ADD next first, REMOVE previous — crash-safe)
```

**Classification priority** (first match wins): UpdateRecommendationsBlocked > ImpactStatementProposed > ImpactStatementRequested > UpgradeBlocker only

**Idempotency** (3-layer, defense in depth):
1. **Linked issues** (primary) — is a Spike already linked?
2. **Bot comments** (fast path) — does `[OTA-Monitor] Spike offered` exist?
3. **Labels** (tertiary) — is the expected label present?

## Four Skills

### Skill 1: Jira Monitor + Pipeline + Daily Status
- **Schedule:** 9am (enriched, always posts) + 3pm (brief, silent if nothing changed)
- **How it works:**
  1. `query_jira` (metadata only — cheap) → filter candidates by labels
  2. `get_jira_issue` only on bugs that might need action (batches of 10, `compact()` between)
  3. Classify each bug using 10 priority-ordered rules (STOP at first match)
  4. Pipeline check: search indexed Slack for FAILEDs (best-effort, Jira unaffected if it fails)
  5. Post daily status + any alerts with action buttons
- **Stateless:** each run reconstructs state from Jira. No cross-run memory. No database.

### Skill 2: Lifecycle Driver
- **Trigger:** Slack button click (NOT scheduled)
- **Gate 1:** [Create Spike] → OrgData lookup, suggest project, human confirms, create Spike (approval-gated)
- **Gate 2:** [Accept — Block Edge] → update labels, auto-trigger workspace PR (collapsed — one click does everything)
- **Gate 3:** [Add FixedIn] → workspace adds fixedIn to YAML, opens PR
- **Gate 4:** [Extend Risk] → workspace extends risk to new version via /propose-risk
- **Auto:** orphaned Spike closure (bot-created only), label auto-transitions, corrections via conversation

### Skill 3: PR Generator
- **Trigger:** Gate 2 or Gate 3 button click
- **Multi-turn execution:**
  - Turn 1: Human clicks → bot updates labels → starts workspace → turn ends
  - Turn 2: Workspace completes → bot shows diff → opens PR (approval-gated) → turn ends
  - Turn 3: PR merges (event or scan) → post-merge cleanup → turn ends
- **Workspace:** clone upstream (not fork), invoke `/propose-risk`, validate with `validate-blocked-edges.py`, push to fork, PR from fork to upstream
- **Branch naming:** `blocked-edges/OCPBUGS-XXXXX-{action}` (deterministic, prevents orphan branches on crash)
- **Post-merge:** create/extend → comment + close Spike | add_fixed_in → remove label | correct → comment only

### Skill 4: Weekly Handover
- **Schedule:** Friday 4pm ET
- **Two phases:** gather data from 9 sources (compact every 3) → synthesize HTML artifact
- **Sources:** JQL (active + resolved), GitHub API (PRs), indexed Slack (pipeline), bot comments (metrics), feedback ticket (signals), latency tracking
- **Output:** HTML via `send_html_to_thread` + summary to channel. Cross-check: output count == JQL count. Never empty.

## Three Human Gates

| Gate | Buttons | What You're Deciding |
|------|---------|---------------------|
| **1** | [Create Spike] / [Skip] / [Escalate] | "Is this the right team project?" |
| **2** | [Accept — Block Edge] / [Revise] / [Not a Blocker] | "Is this really a blocker?" (auto-triggers PR) |
| **3** | [Add FixedIn] / [Skip] | "Is this the right fix version?" |

Everything else is automatic: label transitions, orphaned Spike closure, risk extension, post-merge cleanup, stale reminders (48h/7d via OrgData).

## Key Design Decisions

| # | Decision | Why |
|---|----------|-----|
| 1 | 3 gates, not 5 | Approval fatigue is the #1 threat to agentic monitoring |
| 2 | UpgradeBlocker stays forever | Additive per enhancement doc. Removing breaks dashboard queries |
| 3 | ADD label first, REMOVE second | Crash-safe. Two labels → priority ordering handles it |
| 4 | 3-layer idempotency | Linked issues survive comment deletion and bulk operations |
| 5 | Stateless runs | Jira IS the database. No tracking tickets, no cross-run memory |
| 6 | Two JQL queries | Primary excludes Closed. Without secondary, fixedIn silently fails |
| 7 | /propose-risk for all YAML | Composition over duplication. Single source of truth for format |
| 8 | Always Spike URL | Never OCPBUGS URL in blocked-edge. Handles private bugs automatically |
| 9 | Stale-button check | Re-read bug state on every click. Prevents acting on stale alerts |
| 10 | Close Spike after merge | Spike's purpose fulfilled. Leaving it open clutters team boards |
| 11 | No guardian loop | Polling every 10 min is too expensive. Use event notifications or 9am/3pm scan |
| 12 | Human primary, bot safety net | Bot catches what you miss, ~8 LLM calls/week |

## Schedule

| Time | What | Posts? |
|------|------|--------|
| 9am ET (weekdays) | Daily status + Jira triage + pipeline check | Always |
| 3pm ET (weekdays) | Same checks | Only if action needed |
| Fri 4pm ET | Weekly handover (HTML from 9 sources) | Always |

~8 LLM calls/week. Weekdays only. Weekend labels caught Monday 9am.

## Channels

- **#ota-monitor-bot** — all bot output (alerts, status, handover, button workflows)
- **#osus-graph-data-automation** — bot reads (indexed) for pipeline FAILED detection, never posts
- **#forum-ocp-updates** — indexed for OTA team context, bot never posts

## Repository Contents

```
├── index.html                  # Design doc (GitHub Pages)
├── onboarding.md               # Ready-to-post #chai-users message
├── plan.md                     # Implementation plan summary
└── prompts/                    # Chai Bot scheduled task prompts
    ├── 01_role.md              # Persona identity, state machine, idempotency, buttons, template
    ├── monitor-enriched.md     # 9am task: 3-pass filtering, 10 detection rules, pipeline check
    ├── monitor-brief.md        # 3pm task: same rules, silent if nothing changed
    └── weekly-handover.md      # Friday: 9 data sources, two-phase gather+synthesize, HTML
```

## Risks (all addressed)

| Risk | Mitigation |
|------|-----------|
| Gate fatigue | 5→3 gates, auto-transitions |
| Label non-exclusivity | Priority ordering, first match wins |
| Slack fragility | Pipeline check is best-effort, Jira lifecycle unaffected |
| Button re-posting | \"offered\" markers prevent duplicate alerts |
| fixedIn JQL exclusion | Secondary JQL for Closed/Verified bugs |
| Crash safety | ADD-then-REMOVE label order |
| Early false positives | Week 1-2: human verifies all, pause if <90% precision |

## Status

- [x] Design reviewed 3x by RH Agentic SDLC persona
- [x] Gap analysis against 12 weeks of real RIT status docs (9 rotations, 8 monitors)
- [x] 4 prompt files written and reviewed
- [x] Channels created (#ota-monitor-bot, #test-ota-monitor-bot)
- [x] Onboarding message drafted
- [ ] Post onboarding in #chai-users
- [ ] Teach 8 Verified Knowledge lessons
- [ ] Create OTA-MONITOR-FEEDBACK Jira ticket
- [ ] Gradual rollout (weeks 1-2: read-only → 3-4: Spikes → 5-6: PRs → 7+: full)
"