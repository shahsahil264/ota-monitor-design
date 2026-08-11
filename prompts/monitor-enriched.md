# OTA Monitor — 9am Enriched Scan

This task runs every weekday at 9am ET. It always posts a daily status summary to #ota-monitor-bot, plus any alerts for bugs needing action.

## Step 1: Query Jira (two JQL queries)

Run both queries using `query_jira`:

**Primary JQL** (active bugs):
```
project = OCPBUGS AND labels in (UpgradeBlocker, ImpactStatementRequested, ImpactStatementProposed, UpdateRecommendationsBlocked) AND status != Closed ORDER BY updated DESC
```

**Secondary JQL** (fixedIn candidates):
```
project = OCPBUGS AND labels = UpdateRecommendationsBlocked AND status in (Closed, Verified) AND resolution = Done ORDER BY updated DESC
```

`query_jira` returns metadata only (key, summary, status, priority, labels) — no comments, no description. This is cheap.

## Step 2: Filter candidates for detailed inspection

From primary JQL results:
- Labels contain ONLY `UpgradeBlocker` (no lifecycle label) → candidate
- Labels contain `ImpactStatementRequested` → candidate
- Labels contain `ImpactStatementProposed` → candidate
- `UpdateRecommendationsBlocked` + Open → count only, no `get_jira_issue` needed. Use `query_jira` metadata for the daily status listing.

All secondary JQL results are candidates.

## Step 3: Inspect candidates with get_jira_issue

Call `get_jira_issue` ONLY on filtered candidates. Include comments and linked issues.

Process in batches of 10. After each batch, call `compact()` with structured notes:
```
"Batch N: OCPBUGS-111 (Rule 3: auto-transition done),
 OCPBUGS-222 (Rule 6: new, spike offered),
 OCPBUGS-333 (skip: Spike offered exists)"
```

If `get_jira_issue` fails for a bug, skip it and note: "Could not inspect {BUG_KEY} — Jira API error."

## Step 4: Classify each candidate (priority order, first match wins)

For each bug, evaluate rules in order. **STOP at the first matching rule. Do NOT evaluate further rules for that bug.**

**Rule 1 — fixedIn available (secondary JQL only):**
- No `[OTA-Monitor] fixedIn` comment
- fixVersion field has value OR comment matches "Fixed in.*Advisory.*errata"
- → Post: "Fix detected for {BUG_KEY}. Version: {VERSION}. [Add FixedIn] [Skip]"
- → NEXT BUG

**Rule 2 — active blocker (count only):**
- UpdateRecommendationsBlocked + Open → count for status. No alert.
- → NEXT BUG

**Rule 3 — auto-transition (no button):**
- ImpactStatementRequested + linked Spike in Code Review + no `[OTA-Monitor] Response detected`
- ACTION: ADD ImpactStatementProposed, REMOVE ImpactStatementRequested (UpgradeBlocker STAYS)
- Comment: `[OTA-Monitor] Response detected`
- **INTENTIONAL FALL-THROUGH to Rule 4** — post review buttons immediately.

**Rule 4 — impact statement ready:**
- ImpactStatementProposed + no `[OTA-Monitor] Review offered`
- → Post: "Impact Statement Ready for {BUG_KEY}. [Accept — Block Edge] [Request Revision] [Not a Blocker]"
- Comment: `[OTA-Monitor] Review offered`
- → NEXT BUG

**Rule 4b — review reminder:**
- ImpactStatementProposed + Review offered >24h + no Reviewed
- → Reminder in thread
- → NEXT BUG

**Rule 5 — stale request (48h):**
- ImpactStatementRequested + Spike offered >48h
- → Reminder with team tag via orgdata
- → NEXT BUG

**Rule 5b — escalation (7d):**
- Same but >7 days
- → Escalation warning
- → NEXT BUG

**Rule 6 — new UpgradeBlocker:**
- UpgradeBlocker only + no Spike offered + no linked Spike
- **Duplicate check** (part of condition): search existing open Spikes for same component
  - Duplicate found → "Possible duplicate: {KEY}. [Create — duplicate] [Skip — Duplicate]"
  - No duplicate → OrgData lookup → "New UpgradeBlocker: {BUG_KEY}. Suggested: {PROJECT}. [Create Spike in {PROJECT}] [Skip] [Escalate]"
- Comment: `[OTA-Monitor] Spike offered`
- → NEXT BUG

**Rule 6b — already offered:**
- UpgradeBlocker + Spike offered exists → skip
- → NEXT BUG

**Orphaned spike check:**
- Bug Closed + linked Spike still open
- Bot-created → auto-close. Human-created → alert only.
- → NEXT BUG

## Step 5: Pipeline check (best-effort)

Search indexed #osus-graph-data-automation for "FAILED".
- "branch.*already exists" → stale branch alert
- "extend the risk" → "[Extend Risk] [Skip]" (triggers Skill 3 extend via /propose-risk)
- Merge conflict/CI → PR failure alert
- "Recommend waiting" → ignore

If Slack search fails: note "Pipeline data unavailable", continue. Do NOT abort.

## Step 6: Batch check

>3 new UpgradeBlocker alerts → summary table with [Create All Spikes] [Review Individually] [Skip All]

## Step 7: Daily status (always post)

```
OTA Monitor Daily Status — {DATE}
Active Upgrade Blockers: {COUNT}
  UpdateRecommendationsBlocked: {N}
  ImpactStatementProposed: {N}
  ImpactStatementRequested: {N}
  UpgradeBlocker (new): {N}
Open PRs: {COUNT}
Pipeline: {healthy / N FAILEDs}
```

Nothing changed → thread heartbeat: "✓ Ran at {TIME}. Active: {N}, no changes."

## Step 8: Error reporting

Run failed → channel top-level: "⚠️ OTA Monitor failed at {TIME}. Error: {DETAILS}."

## Bounds

- Max 50 primary, 20 secondary bugs
- Max 10 individual alerts
- 3 min timeout per step
