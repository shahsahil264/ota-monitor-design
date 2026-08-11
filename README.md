# OTA Monitor — Agentic System

An agentic system that automates the weekly OTA Monitor RIT rotation role using [Chai Bot](https://redhat-chai-bot.github.io/guide.html) as the primary platform.

**Design doc:** [shahsahil264.github.io/ota-monitor-design](https://shahsahil264.github.io/ota-monitor-design/)

## What It Does

The OTA Monitor watches the Cincinnati update pipeline and drives UpgradeBlocker bugs through a 4-stage Jira lifecycle. Today this is ~1-2 hours/day of manual work — running Go commands, creating YAML files, tracking Jira labels, writing handover docs.

This system reduces it to **~5 minutes/day of button clicks**. Chai Bot handles detection, classification, Spike creation, blocked-edge PR generation, post-merge cleanup, and weekly handover. The human approves 3 decisions.

## How It Works

```
Chai Bot (does the work)          You (approve or reject)
─────────────────────────         ─────────────────────────
9am/3pm: scans Jira + pipeline    Read daily status (~30 sec)
Detects new UpgradeBlocker bugs   Click [Create Spike] if needed
Creates Spike cards (Jira)        Review impact statement answers
Auto-transitions lifecycle labels Click [Accept — Block Edge]
Generates blocked-edge YAML       Review PR diff
Opens PRs from fork to upstream   Click [Add FixedIn] when fix ships
Closes Spikes after PR merge
Generates weekly handover (HTML)
```

## Three Human Gates

| Gate | What You're Deciding |
|------|---------------------|
| **1. Create Spike** | "Is this the right team project?" |
| **2. Accept — Block Edge** | "Is this really a blocker?" (auto-triggers PR) |
| **3. Add FixedIn** | "Is this the right fix version?" |

Everything else is automatic.

## Repository Contents

```
├── index.html                  # Design doc (GitHub Pages)
├── onboarding.md               # Ready-to-post #chai-users message
├── plan.md                     # Implementation plan summary
└── prompts/                    # Chai Bot scheduled task prompts
    ├── 01_role.md              # Persona identity + rules
    ├── monitor-enriched.md     # 9am task (10 detection rules)
    ├── monitor-brief.md        # 3pm task (silent if nothing)
    └── weekly-handover.md      # Friday handover (9 data sources)
```

## Key Design Decisions

- **Stateless** — each run reconstructs state from Jira (labels + bot comments + linked issues). No database.
- **3-layer idempotency** — linked issues (primary) + bot comments (fast path) + labels (tertiary)
- **Crash-safe labels** — ADD next label first, REMOVE previous second
- **Composition** — delegates YAML generation to `/propose-risk` in the graph-data repo
- **UpgradeBlocker stays forever** — additive label per the [enhancement doc](https://github.com/openshift/enhancements/tree/master/enhancements/update/update-blocker-lifecycle)
- **Human primary, bot safety net** — bot catches what you miss, doesn't replace you

## Schedule

| Time | What | Posts? |
|------|------|--------|
| 9am ET (weekdays) | Daily status + Jira triage + pipeline check | Always |
| 3pm ET (weekdays) | Same checks | Only if action needed |
| Fri 4pm ET | Weekly handover (HTML report) | Always |

~8 LLM calls/week. Weekdays only.

## Channels

- **#ota-monitor-bot** — all bot output (alerts, status, handover, button workflows)
- **#osus-graph-data-automation** — bot reads (indexed) for pipeline FAILED detection, never posts

## Status

- [x] Design reviewed 3x by RH Agentic SDLC persona
- [x] Gap analysis against 12 weeks of real RIT status docs
- [x] 4 prompt files written and reviewed
- [x] #ota-monitor-bot + #test-ota-monitor-bot channels created
- [x] Onboarding message drafted
- [ ] Post onboarding in #chai-users
- [ ] Teach 8 Verified Knowledge lessons
- [ ] Create OTA-MONITOR-FEEDBACK Jira ticket
- [ ] Gradual rollout (weeks 1-2: read-only → weeks 3-4: Spikes → weeks 5-6: PRs → week 7+: full)
"