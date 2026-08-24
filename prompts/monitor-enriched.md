# OTA Monitor — 10:00 UTC Enriched Scan

This task runs every weekday at 10:00 UTC (6am ET / 12pm CEST). It always posts a daily status summary to #ota-monitor-bot, plus any alerts for bugs needing action.

## Step 1: Query Jira (two JQL queries)

Run both queries using `query_jira`:

**Primary JQL** (active bugs):
```
project = OCPBUGS AND labels in (UpgradeBlocker, ImpactStatementRequested, ImpactStatementProposed, UpdateRecommendationsBlocked) AND status != Closed ORDER BY updated DESC
```

**Secondary JQL** (fixedIn candidates — primary excludes Closed bugs):
```
project = OCPBUGS AND labels = UpdateRecommendationsBlocked AND status in (Closed, Verified) AND resolution = Done AND updated >= "2026-08-13" ORDER BY updated DESC
```

The `updated >= "2026-08-13"` filter excludes bugs closed before the bot started monitoring. Without this, every pre-existing Closed+UpdateRecommendationsBlocked bug appears as a fixedIn candidate every scan (they lack `[OTA-Monitor] fixedIn` comments because the bot didn't exist yet). Bugs closed before Aug 13 2026 were handled manually — the bot should not re-alert on them.

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

If `get_jira_issue` fails for a bug, skip it and note in the summary: "Could not inspect <BUG_KEY> — Jira API error."

## Step 4: Classify each candidate (priority order, first match wins)

For each inspected bug, evaluate rules in this exact order. **STOP at the first matching rule. Do NOT evaluate further rules for that bug.** Move to the next bug immediately after a match.

**Rule 1 — fixedIn available (secondary JQL bugs only):**
- Bug is from secondary JQL (Closed/Verified + UpdateRecommendationsBlocked)
- Check A: does a comment matching `[OTA-Monitor] fixedIn` exist? If yes → skip (already handled by bot)
- Check B: does the `fixVersion` field have a value, OR does a comment match `Fixed in.*Advisory.*errata`? If no fix signal → skip (nothing to act on)
- Check C: search openshift/cincinnati-graph-data for blocked-edge YAML files that reference this bug's Spike URL or risk name. If a matching file already has a `fixedIn` field set → skip (already handled manually before bot monitoring). This check prevents false alerts for pre-bot bugs that were resolved outside the bot's workflow.
  - **If multiple files share the same risk name across consecutive versions** (e.g., `4.20.23-RiskName.yaml` and `4.20.24-RiskName.yaml`): identify which file represents the LATEST affected version for that risk. Only the LATEST version's file is expected to have `fixedIn` set — earlier versions' files have no `fixedIn` field because the latest version's fixedIn already steers upgrades past all earlier blocked versions. Do NOT flag an earlier version's file as missing fixedIn; the absence of fixedIn on earlier versions is the expected state, not a gap. Only alert if the LATEST affected version's file lacks `fixedIn` despite a fix signal being detected.
- If fix signal detected (Check B) AND no existing fixedIn in YAML (Check C) AND no bot marker (Check A) → post alert:
  "Fix detected for <BUG_KEY>. Version: <VERSION>. [Add FixedIn] [Skip]"
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
- Post alert: "Impact Statement Ready for <BUG_KEY>. [Accept — Block Edge] [Request Revision] [Not a Blocker]"
- Add comment: `[OTA-Monitor] Review offered`
- → NEXT BUG

**Rule 4b — review reminder:**
- Labels contain `ImpactStatementProposed`
- Comment `[OTA-Monitor] Review offered` exists AND is older than 24 hours
- No comment matching `[OTA-Monitor] Reviewed`
- Post reminder in thread: "Reminder: <BUG_KEY> impact statement awaiting OTA review (offered <N> hours ago)."
- → NEXT BUG

**Rule 5 — stale impact statement request:**
- Labels contain `ImpactStatementRequested`
- Determine reference timestamp for staleness:
  - If comment `[OTA-Monitor] Spike offered` exists → use that comment's timestamp
  - Else if a linked Spike issue exists → use the Spike's creation date (Spike was created manually outside bot workflow)
  - Else → skip this rule (no way to determine when the request started)
- Reference timestamp is older than 48 hours
- Look up the assignee's team using orgdata (component → team)
- Post reminder: "STALE: <BUG_KEY> — impact statement requested <N> days ago. Team: <TEAM>. Assignee: <ASSIGNEE>."
- → NEXT BUG

**Rule 5b — escalation warning (7+ days):**
- Same as Rule 5 but reference timestamp is older than 7 days
- Post escalation warning: "ESCALATION: <BUG_KEY> — no impact statement response in <N> days. Per the enhancement: 'In the absence of a response within 7 days, we may declare a conditional update risk based on our current understanding.'"
- → NEXT BUG

**Rule 6 — new UpgradeBlocker:**
- Labels contain `UpgradeBlocker` but NOT `ImpactStatementRequested`, NOT `ImpactStatementProposed`, NOT `UpdateRecommendationsBlocked`
- No comment matching `[OTA-Monitor] Spike offered`
- No linked Spike issue (check for ANY linked Spike — open OR closed. A closed Spike means this bug was already triaged, possibly via [Not a Blocker]. Do NOT re-alert.)
- No comment matching `[OTA-Monitor][Feedback]` (if a Feedback comment exists, this bug was already triaged — either skipped or determined not a blocker. Do NOT re-alert.)
- **Duplicate check** (part of the condition, not after): Search for existing open Spike issues linked to bugs with the same component.
  - If duplicate found: post "Possible duplicate: <EXISTING_SPIKE_KEY> already exists for component <COMPONENT>. [Create Spike — possible duplicate] [Skip — Duplicate]"
  - If no duplicate: look up component → team → project using orgdata. Post alert: "New UpgradeBlocker: <BUG_KEY> — <SUMMARY>. Component: <COMPONENT>. Suggested project: <PROJECT>. [Create Impact Statement in <PROJECT>] [Skip] [Escalate]"
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
- **Clone check**: Before auto-closing, check if the Spike is also linked to any OTHER open OCPBUGS bugs (clones or related bugs). If yes → do NOT auto-close. Alert instead: "Spike <SPIKE_KEY> linked to closed <BUG_KEY> but also linked to open <OTHER_BUG_KEY>. Review before closing."
- If no other open bugs are linked to the Spike:
  - Check: does the OCPBUGS bug have a `[OTA-Monitor] Impact statement spike created` comment? (= bot-created Spike)
    - If bot-created: AUTO-CLOSE the Spike directly (no button). Resolution: "Done" if bug resolved as Done, "Won't Do" if Won't Fix/Not a Bug. Comment on Spike: `[OTA-Monitor] Auto-closed: linked <BUG_KEY> is <STATUS>`. Post to Slack: "Spike <SPIKE_KEY> auto-closed."
    - If human-created: post alert only: "Orphaned Spike: <SPIKE_KEY> still open but <BUG_KEY> is Closed. Please review and close manually."
- → NEXT BUG

## Step 5: Pipeline check (best-effort, two-phase)

**Important**: The indexed Slack search uses SEMANTIC similarity, not keyword matching. A search for "FAILED" will return messages that are semantically related to failure — not just messages containing the literal word "FAILED." This means you cannot rely on the search query alone to find FAILED messages. Use a two-phase approach:

### Phase 1 — Confirm channel is searchable

Search #osus-graph-data-automation for any recent messages (broad query like "recent pipeline activity" or "promotion status"). This confirms the channel is indexed and accessible. Record the number of messages returned.

- If Phase 1 returns results → channel is confirmed searchable. Proceed to Phase 2.
- If Phase 1 returns nothing → report: "🔄 Pipeline: ⚠️ data unavailable (channel not indexed or search failed)"

### Phase 2 — Keyword scan for FAILEDs

Use `result_grep` or `grep_text` on the messages returned in Phase 1 to find lines containing the literal substring "FAILED". This is a text match, not a semantic search — it will only match messages that actually contain the word.

**Before alerting on any FAILED message**, check if the associated PR (if mentioned) is already merged. Most FAILEDs are phantom failures from already-merged PRs — the pipeline retried after the PR landed. If the PR is merged, skip the alert entirely.

For each FAILED message found via grep:
- **First**: extract the PR number from the message. Check if that PR is merged in openshift/cincinnati-graph-data. If merged → skip (phantom failure, already resolved).
- If message contains "branch.*already exists" → classify as "possible stale branch." **NEVER recommend or perform branch deletion from the bot fork as the default action.** Most "branch already exists" FAILEDs are phantom failures caused by timing (a prior attempt already succeeded, or a retry raced against an in-flight promotion) — verify the branch's associated PR state first. Alert: "Pipeline: branch already exists blocking promotion — needs investigation before any action. Do NOT delete branches from the bot fork without confirming this isn't a phantom failure. [Investigate] [Skip]"
- If message contains "extend the risk" or "declare a fix version" → classify as "risk extension needed." Alert: "Pipeline: risk extension needed. <RISK_NAME> needs extending to <VERSION>. [Extend Risk] [Skip]"
  When human clicks [Extend Risk]: trigger Skill 3 with action_type="extend". Always invoke /propose-risk for extensions (same composition pattern as create); the graph-extend-or-fix Go tool is deprecated and must not be used.
- If message indicates merge conflict or CI failure → classify as "PR failure." Alert: "Pipeline: promotion PR failed. Recreate PR."
- If message contains "Recommend waiting" → ignore (informational)

### Classification

| Phase 1 | Phase 2 | Report |
|---------|---------|--------|
| Results returned | 0 FAILEDs in grep | 🔄 Pipeline: Healthy (searched <N> messages, 0 FAILEDs in last 2 days) |
| Results returned | N FAILEDs in grep | Process each per rules above, report count |
| No results | — | 🔄 Pipeline: ⚠️ data unavailable (channel not indexed or search failed) |

Continue to Step 6 regardless of pipeline check outcome. Do NOT abort the run.
The Jira lifecycle monitoring (Steps 1-4) is the reliable core. Pipeline check is best-effort and supplementary. A Slack indexing outage must never take down the Jira monitoring.

## Step 6: Batch alert check

If more than 3 new UpgradeBlocker alerts (Rule 6) would be posted in this run, do NOT post them individually. Instead, post a single summary table:

```
New UpgradeBlocker bugs detected: <COUNT>

| Bug | Component | Priority | Suggested Project |
|-----|-----------|----------|-------------------|
| ... | ...       | ...      | ...               |

<N> bugs → <PROJECT_A>, <M> bugs → <PROJECT_B>

[Create All Spikes] [Review Individually] [Skip All]
```

## Step 6b: Gather PR data (filtered)

Search GitHub for open PRs in openshift/cincinnati-graph-data. **Filter to OTA-relevant PRs only:**

**Include:**
- PRs touching `blocked-edges/` or `channels/` directories
- PRs authored by `openshift-ota-bot` (promotion PRs)
- PRs with OTA-relevant titles containing: `blocked-edges`, `promote`, `risk`, `minor_min`, `product-life-cycle`

**Exclude:**
- PRs by `dependabot` (dependency bumps)
- PRs that don't match any of the above criteria

**Split into two categories:**
- **Bug-linked PRs**: PRs whose title references a specific OCPBUGS bug (e.g., "blocked-edges: OCPBUGS-100182"). These go inline with their bug in the status, NOT in the PR section.
- **Other OTA PRs**: Everything else that passed the filter (minor_min raises, product-life-cycle, promotions). These go in a separate "Other OTA PRs" section.

## Step 7: Post daily status (always, even if no alerts)

Post a daily status summary to #ota-monitor-bot. **Tag the OTA Monitor rotation in every top-level post** using `<!subteam^STE7S7ZU2|@ota-monitor>` so the current rotation monitor gets notified.

### Format rules

- **Group bugs by lifecycle stage**, ordered by urgency (most actionable first)
- **Omit empty stages** — do NOT show stages with 0 bugs
- **Collapse when uniform** — if all bugs are in the same stage, use a single line: "Active Upgrade Blockers: <N> (all <STAGE>)" instead of a 4-line breakdown with three "0" lines
- **Bug-linked PRs inline** — each UpdateRecommendationsBlocked bug shows its associated blocked-edge PR and merge status on the same line (clickable link)
- **Spike status inline** — each ImpactStatementRequested/Proposed bug shows its linked Spike and Spike status
- **Separate PR section** — only for OTA-relevant PRs NOT tied to a specific bug
- **All links clickable** — bug keys link to Jira, PR numbers link to GitHub
- **Use emoji prefixes** for visual scanning and dividers between sections
- **Cross-reference alerts** — if an alert was posted above, reference it with ➡️

### Template: quiet day (all bugs in same stage)

```
<!subteam^STE7S7ZU2|@ota-monitor> — Daily Status — <DATE> <TIME> UTC

✅ No new alerts. No action needed.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 Active Upgrade Blockers: <N> (all UpdateRecommendationsBlocked)

✅ Blocked, waiting for fix:
• <BUG_KEY> — <SUMMARY>
  <COMPONENT> · <PRIORITY> · <STATUS> — PR #<XXXX> ✅ merged
• <BUG_KEY> — <SUMMARY>
  <COMPONENT> · <PRIORITY> · <STATUS> — PR #<XXXX> 🔄 open
• <BUG_KEY> — <SUMMARY>
  <COMPONENT> · <PRIORITY> · <STATUS> — ⚠️ No blocked-edge PR found

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📦 Other OTA PRs: <N>
• #<XXXX> — <TITLE> <STATUS_EMOJI> <STATUS>

🔄 Pipeline: Healthy (searched <N> messages, 0 FAILEDs in last 2 days)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Next scan: <NEXT_DATE> <NEXT_TIME> UTC
```

### Template: busy day (bugs across multiple stages)

```
<!subteam^STE7S7ZU2|@ota-monitor> — Daily Status — <DATE> <TIME> UTC

🔴 <N> items need attention — see alerts above

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 Active Upgrade Blockers: <TOTAL>

🔴 New/Untriaged (<N>):
• <BUG_KEY> — <SUMMARY>
  <COMPONENT> · <PRIORITY> · <STATUS>
  ➡️ [Create Spike] alert posted above

⏳ Waiting on Teams (<N>):
• <BUG_KEY> — <SUMMARY>
  <COMPONENT> · <PRIORITY> · <STATUS> — Spike: <SPIKE_KEY> (<SPIKE_STATUS>, <N> days)

📋 Awaiting OTA Review (<N>):
• <BUG_KEY> — <SUMMARY>
  <COMPONENT> · <PRIORITY> · <STATUS> — Spike: <SPIKE_KEY> (<SPIKE_STATUS>)
  ➡️ [Accept — Block Edge] alert posted above

✅ Blocked, waiting for fix (<N>):
• <BUG_KEY> — <SUMMARY>
  <COMPONENT> · <PRIORITY> — PR #<XXXX> ✅ merged
• <BUG_KEY> — <SUMMARY>
  <COMPONENT> · <PRIORITY> — PR #<XXXX> 🔄 open
• <BUG_KEY> — <SUMMARY>
  <COMPONENT> · <PRIORITY> — ⚠️ No blocked-edge PR found

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📦 Other OTA PRs: <N>
• #<XXXX> — <TITLE> <STATUS_EMOJI> <STATUS>

🔄 Pipeline: <PIPELINE_STATUS>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Next scan: <NEXT_DATE> <NEXT_TIME> UTC
```

### Stage emoji mapping

| Stage | Emoji | Label |
|-------|-------|-------|
| New/Untriaged | 🔴 | UpgradeBlocker only (no lifecycle label) |
| Waiting on Teams | ⏳ | ImpactStatementRequested |
| Awaiting OTA Review | 📋 | ImpactStatementProposed |
| Blocked, waiting for fix | ✅ | UpdateRecommendationsBlocked |

### Pipeline status format

| Condition | Output |
|-----------|--------|
| Search returned results, 0 FAILEDs | 🔄 Pipeline: Healthy (searched <N> messages, 0 FAILEDs in last 2 days) |
| Search returned results, N FAILEDs | 🔄 Pipeline: ⚠️ <N> FAILED(s) detected — see alerts above |
| Search failed or channel not indexed | 🔄 Pipeline: ⚠️ data unavailable (channel not indexed or search failed) |

### Thread heartbeat (no alerts, nothing changed)

If there are no alerts and nothing changed, post to a thread (not top-level):
"✓ OTA Monitor ran at <TIME> UTC. Active blockers: <N>, no changes. Next run: <NEXT_TIME> UTC."

## Step 8: Record run metrics

After posting the daily status, record what this run did by commenting on OTA-2104 (the OTA-MONITOR-FEEDBACK ticket). This is how the weekly handover collects bot performance data — JQL cannot search comment body text, so each run self-reports.

Call `comment_on_jira_issue` with:
- `issue_key`: "OTA-2104"
- `comment_text`: single-line run summary (format below)

**Format** (single line, machine-parseable):
```
[OTA-Monitor][Run] <DATE> <TIME> UTC | alerts:<N> spikes:<N> transitions:<N> reviews:<N> skips:<N> fixedIn:<N> bugs_processed:<N>
```

Count each action taken during THIS run:
- `alerts`: total alerts posted (new UpgradeBlocker, fix detected, pipeline FAILED, etc.)
- `spikes`: spike creation buttons posted (Rule 6)
- `transitions`: auto-transitions executed (Rule 3)
- `reviews`: review buttons posted (Rule 4)
- `skips`: bugs skipped by any rule (already handled, no action needed)
- `fixedIn`: fixedIn alerts posted (Rule 1)
- `bugs_processed`: total bugs inspected via `get_jira_issue`

**Error isolation**: If `comment_on_jira_issue` fails, log the error but do NOT abort the run. Run metrics are nice-to-have, not mission-critical. The daily status and alerts are the critical outputs.

## Step 9: Error reporting

If the entire run fails (JQL timeout, API error), post an error to the channel (top-level, not thread):
"⚠️ OTA Monitor failed at <TIME> UTC. Error: <ERROR_DETAILS>. Next run: <NEXT_TIME> UTC."

## Bounds

- Process at most 50 bugs from the primary JQL
- Process at most 20 bugs from the secondary JQL
- Post at most 10 individual alerts (summarize the rest as counts)
- If any step takes more than 3 minutes without progress, skip to the next step
