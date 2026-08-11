# OTA Monitor — 9am Enriched Scan

This task runs every weekday at 10:00 UTC (6am ET / 12pm CEST). It always posts a daily status summary to #ota-monitor-bot, plus any alerts for bugs needing action.

## Step 1: Query Jira (two JQL queries)

Run both queries using `query_jira`:

**Primary JQL** (active bugs):
```
project = OCPBUGS AND labels in (UpgradeBlocker, ImpactStatementRequested, ImpactStatementProposed, UpdateRecommendationsBlocked) AND status != Closed ORDER BY updated DESC
```

**Secondary JQL** (fixedIn candidates — primary excludes Closed bugs):
```
project = OCPBUGS AND labels = UpdateRecommendationsBlocked AND status in (Closed, Verified) AND resolution = Done ORDER BY updated DESC
```

`query_jira` returns metadata only (key, summary, status, priority, labels) — no comments, no description. This is cheap.

## Step 2: Filter candidates for detailed inspection

From the primary JQL results, identify bugs that MIGHT need action based on their labels:

- Labels contain ONLY `UpgradeBlocker` (no lifecycle label) → candidate (might be new)
- Labels contain `ImpactStatementRequested` → candidate (might have response or be stale)
- Labels contain `ImpactStatementProposed` → candidate (might need review buttons)

Skip bugs where labels indicate no action is possible:
- `UpdateRecommendationsBlocked` + status is Open → track count only, no `get_jira_issue` needed. Use the `query_jira` metadata (key, summary, status) for the daily status listing. Do NOT call `get_jira_issue` — these bugs are stable and don't need comment inspection.

All secondary JQL results are candidates (need comment inspection for fixedIn detection).

## Step 3: Inspect candidates with get_jira_issue

Call `get_jira_issue` ONLY on the filtered candidates from Step 2. Include comments and linked issues.

Process in batches of 10. After each batch of 10, call `compact()` with structured notes:

```
"Batch N: OCPBUGS-111 (Rule 3: auto-transition done),
 OCPBUGS-222 (Rule 6: new, spike offered),
 OCPBUGS-333 (skip: Spike offered exists)"
```

Include: bug key, which rule matched, action taken or skipped. This is the LLM's memory of what it already handled — without it, bugs from prior batches may be re-evaluated.

If `get_jira_issue` fails for a bug, skip it and note in the summary: "Could not inspect {BUG_KEY} — Jira API error."

## Step 4: Classify each candidate (priority order, first match wins)

For each inspected bug, evaluate rules in this exact order. **STOP at the first matching rule. Do NOT evaluate further rules for that bug.** Move to the next bug immediately after a match.

**Rule 1 — fixedIn available (secondary JQL bugs only):**
- Bug is from secondary JQL (Closed/Verified + UpdateRecommendationsBlocked)
- Check: does a comment matching `[OTA-Monitor] fixedIn` exist?
- Check: does the `fixVersion` field have a value, OR does a comment match `Fixed in.*Advisory.*errata`?
- If no fixedIn marker AND fix signal detected → post alert:
  "Fix detected for {BUG_KEY}. Version: {VERSION}. [Add FixedIn] [Skip]"
- → NEXT BUG

**Rule 2 — active blocker (no action, count only):**
- Labels contain `UpdateRecommendationsBlocked` AND bug is Open
- Add to active blocker count for daily status. No alert.
- → NEXT BUG

**Rule 3 — auto-transition (mechanical, no button):**
- Labels contain `ImpactStatementRequested`
- Linked Spike issue exists AND Spike status is "Code Review"
- No comment matching `[OTA-Monitor] Response detected`
- ACTION (automatic, no human approval needed):
  1. ADD label `ImpactStatementProposed` to the OCPBUGS bug
  2. REMOVE label `ImpactStatementRequested` (UpgradeBlocker label STAYS — never remove it)
  3. Add comment: `[OTA-Monitor] Response detected`
  4. **INTENTIONAL FALL-THROUGH to Rule 4** — post review buttons immediately so the human gets both notifications at once (transition + review request). This is by design.

**Rule 4 — impact statement ready for review:**
- Labels contain `ImpactStatementProposed`
- No comment matching `[OTA-Monitor] Review offered`
- Post alert: "Impact Statement Ready for {BUG_KEY}. [Accept — Block Edge] [Request Revision] [Not a Blocker]"
- Add comment: `[OTA-Monitor] Review offered`
- → NEXT BUG

**Rule 4b — review reminder:**
- Labels contain `ImpactStatementProposed`
- Comment `[OTA-Monitor] Review offered` exists AND is older than 24 hours
- No comment matching `[OTA-Monitor] Reviewed`
- Post reminder in thread: "Reminder: {BUG_KEY} impact statement awaiting OTA review (offered {N} hours ago)."
- → NEXT BUG

**Rule 5 — stale impact statement request:**
- Labels contain `ImpactStatementRequested`
- Comment `[OTA-Monitor] Spike offered` exists AND is older than 48 hours
- Look up the assignee's team using orgdata (component → team)
- Post reminder: "STALE: {BUG_KEY} — impact statement requested {N} days ago. Team: {TEAM}. Assignee: {ASSIGNEE}."
- → NEXT BUG

**Rule 5b — escalation warning (7+ days):**
- Same as Rule 5 but `Spike offered` comment is older than 7 days
- Post escalation warning: "ESCALATION: {BUG_KEY} — no impact statement response in {N} days. Per the enhancement: 'In the absence of a response within 7 days, we may declare a conditional update risk based on our current understanding.'"
- → NEXT BUG

**Rule 6 — new UpgradeBlocker:**
- Labels contain `UpgradeBlocker` but NOT `ImpactStatementRequested`, NOT `ImpactStatementProposed`, NOT `UpdateRecommendationsBlocked`
- No comment matching `[OTA-Monitor] Spike offered`
- No linked Spike issue
- **Duplicate check** (part of the condition, not after): Search for existing open Spike issues linked to bugs with the same component.
  - If duplicate found: post "Possible duplicate: {EXISTING_SPIKE_KEY} already exists for component {COMPONENT}. [Create Spike — possible duplicate] [Skip — Duplicate]"
  - If no duplicate: look up component → team → project using orgdata. Post alert: "New UpgradeBlocker: {BUG_KEY} — {SUMMARY}. Component: {COMPONENT}. Suggested project: {PROJECT}. [Create Impact Statement in {PROJECT}] [Skip] [Escalate]"
- Add comment: `[OTA-Monitor] Spike offered`
- → NEXT BUG

**Rule 6b — already offered:**
- Labels contain `UpgradeBlocker` only
- Comment `[OTA-Monitor] Spike offered` exists
- Skip. No alert.
- → NEXT BUG

**Orphaned spike check (all bugs):**
- Bug status is Closed
- A linked Spike issue exists and is NOT Closed
- Check: does the OCPBUGS bug have a `[OTA-Monitor] Impact statement spike created` comment? (= bot-created Spike)
  - If bot-created: AUTO-CLOSE the Spike directly (no button). Resolution: "Done" if bug resolved as Done, "Won't Do" if Won't Fix/Not a Bug. Comment on Spike: `[OTA-Monitor] Auto-closed: linked {BUG_KEY} is {STATUS}`. Post to Slack: "Spike {SPIKE_KEY} auto-closed."
  - If human-created: post alert only: "Orphaned Spike: {SPIKE_KEY} still open but {BUG_KEY} is Closed. Please review and close manually."
- → NEXT BUG

## Step 5: Pipeline check (best-effort)

Search indexed Slack history from #osus-graph-data-automation for recent messages containing "FAILED".

For each FAILED message found:
- If message contains "branch.*already exists" → classify as "stale branch." Alert: "Pipeline: stale branch blocking promotion. Delete branch from bot fork."
- If message contains "extend the risk" or "declare a fix version" → classify as "risk extension needed." Alert: "Pipeline: risk extension needed. {RISK_NAME} needs extending to {VERSION}. [Extend Risk] [Skip]"
  When human clicks [Extend Risk]: trigger Skill 3 with action_type="extend". Workspace invokes /propose-risk (same composition pattern as create — do NOT use graph-extend-or-fix Go tool).
- If message indicates merge conflict or CI failure → classify as "PR failure." Alert: "Pipeline: promotion PR failed. Recreate PR."
- If message contains "Recommend waiting" → ignore (informational)

If Slack search fails, times out, or returns no results:
- Note "Pipeline data unavailable" in the daily status
- Continue to Step 6. Do NOT abort the run.
- The Jira lifecycle monitoring (Steps 1-4) is the reliable core. Pipeline check is best-effort and supplementary. A Slack indexing outage must never take down the Jira monitoring.

## Step 6: Batch alert check

If more than 3 new UpgradeBlocker alerts (Rule 6) would be posted in this run, do NOT post them individually. Instead, post a single summary table:

```
New UpgradeBlocker bugs detected: {COUNT}

| Bug | Component | Priority | Suggested Project |
|-----|-----------|----------|-------------------|
| ... | ...       | ...      | ...               |

{N} bugs → {PROJECT_A}, {M} bugs → {PROJECT_B}

[Create All Spikes] [Review Individually] [Skip All]
```

## Step 7: Post daily status (always, even if no alerts)

Post a daily status summary to #ota-monitor-bot. **Tag @ota-monitor in every top-level post** so the current rotation monitor gets notified.

```
@ota-monitor — Daily Status — {DATE}

Active Upgrade Blockers: {COUNT}
  UpdateRecommendationsBlocked: {N}
  ImpactStatementProposed (awaiting review): {N}
  ImpactStatementRequested (waiting on teams): {N}
  UpgradeBlocker (new/untriaged): {N}

Open PRs in openshift/cincinnati-graph-data: {COUNT}
Pipeline: {healthy / N FAILEDs detected}

{Any alerts posted above}
```

If there are no alerts and nothing changed, post the status summary to a thread (not top-level) to reduce noise:
"✓ OTA Monitor ran at {TIME} UTC. Active blockers: {N}, no changes. Next run: {NEXT_TIME} UTC."

## Step 8: Error reporting

If the entire run fails (JQL timeout, API error), post an error to the channel (top-level, not thread):
"⚠️ OTA Monitor failed at {TIME} UTC. Error: {ERROR_DETAILS}. Next run: {NEXT_TIME} UTC."

## Bounds

- Process at most 50 bugs from the primary JQL
- Process at most 20 bugs from the secondary JQL
- Post at most 10 individual alerts (summarize the rest as counts)
- If any step takes more than 3 minutes without progress, skip to the next step
