# OTA Monitor — Weekly Handover

This task runs every Friday at 22:00 UTC (6pm ET / 12am CEST). It generates a structured handover document for the incoming OTA Monitor and always posts to #ota-monitor-bot.

## Execution Structure (Two Phases)

**PHASE 1 — Data gathering** (sources 1-9): For each source, call the tool, extract key data. Call `compact()` after every 3 sources to free context. Store results in compact notes with structured summaries.

**PHASE 2 — Synthesis**: Read compacted notes. Compose HTML artifact. Do NOT re-query sources.

## Data Gathering (Phase 1)

Gather data from these 9 sources. Every number in the handover MUST come from a tool query — never estimate or guess.

### Source 1: Active blockers by lifecycle stage

Use `query_jira` with:
```
project = OCPBUGS AND labels in (UpgradeBlocker, ImpactStatementRequested, ImpactStatementProposed, UpdateRecommendationsBlocked) AND status != Closed ORDER BY priority DESC, updated ASC
```

For each bug, record: key, summary, labels (= lifecycle stage), component, priority, assignee, days since last update.

Group by stage:
- UpdateRecommendationsBlocked (active blocks)
- ImpactStatementProposed (waiting on OTA review)
- ImpactStatementRequested (waiting on component teams)
- UpgradeBlocker only (new/untriaged)

### Source 2: Items waiting on component teams

Filter from Source 1: bugs with ImpactStatementRequested label. For each:
- Days waiting (since [OTA-Monitor] Spike offered comment timestamp)
- Component team (from bug's component field)
- Whether a reminder has been posted (check for stale reminder comments)

### Source 3: Resolved this week

Use `query_jira` with:
```
project = OCPBUGS AND labels in (UpgradeBlocker) AND status = Closed AND resolved >= -7d ORDER BY resolved DESC
```

Record: key, summary, resolution, fix version.

### Source 4: Open PRs in openshift/cincinnati-graph-data (filtered)

Search GitHub for open PRs. **Filter to OTA-relevant PRs only:**
- INCLUDE: PRs touching `blocked-edges/` or `channels/`, PRs by `openshift-ota-bot`, PRs with titles containing `blocked-edges`, `promote`, `risk`, `minor_min`, `product-life-cycle`
- EXCLUDE: PRs by `dependabot`, PRs not matching any include criteria

Record: PR number, title, age (days since opened), CI status, review status (approved/changes requested/pending). Note which PRs are linked to specific UpgradeBlocker bugs (these appear inline in the blocker table, not repeated in the PR section).

### Source 5: Merged PRs this week

Search GitHub for PRs merged in the last 7 days in openshift/cincinnati-graph-data with "blocked-edges" in the title. Record: PR number, title, merge date.

### Source 6: Pipeline health

Search indexed Slack history from #osus-graph-data-automation for messages in the last 7 days. Count: total FAILEDs, FAILEDs with associated PRs, FAILEDs without PRs, Recommend waiting messages.

### Source 7: Bot activity metrics

**Primary method** — read run summaries from OTA-2104:
1. Call `get_jira_issue` on OTA-2104
2. Filter comments for lines matching `[OTA-Monitor][Run]` with timestamps from the last 7 days
3. Parse each line's key:value pairs and aggregate totals:
   - alerts (total alerts posted)
   - spikes (spike creation buttons posted)
   - transitions (auto-transitions)
   - reviews (review buttons posted)
   - skips (bugs skipped)
   - fixedIn (fixedIn updates)
   - bugs_processed (total bugs inspected)

Each `monitor-enriched` and `monitor-brief` run appends one `[OTA-Monitor][Run]` comment to OTA-2104 via `comment_on_jira_issue` at the end of its execution. With 10 runs per week (2/day × 5 days), expect ~10 run summary comments.

**Data lag caveat**: Comment data comes from the Dataverse CLOUDRHAI_MARTS database, which syncs periodically. Comments from the Friday 10:00 UTC run may not be synced by the Friday 22:00 UTC handover (only 12 hours apart). Accept this gap — report metrics as covering Mon 10:00 through Thu 22:00 (9 of 10 runs). The missing run is always the most recent and least likely to have significant new activity.

**Fallback** — if OTA-2104 has no `[OTA-Monitor][Run]` comments (bot just started, or ticket missing):
1. Use the bugs already fetched for Sources 1-3 (do NOT re-query)
2. For each bug, scan its comments (already available from `get_jira_issue` calls in Sources 2 and 9) for `[OTA-Monitor]` prefixes
3. Count each marker type and filter to comments from the last 7 days
4. This is approximate — it only covers bugs still in the active lifecycle, not bugs that were resolved and fell out of Source 1

Note: JQL does NOT support searching comment body text. The `comment ~ "[OTA-Monitor]"` query will not work. This is a known Jira API limitation. Always use the primary method (OTA-2104 run summaries) or the fallback (per-bug comment inspection).

### Source 8: Feedback signals

Use `get_jira_issue` on OTA-2104 (the OTA-MONITOR-FEEDBACK ticket). Read comments from the last 7 days. Count:
- SKIP signals (potential false positives)
- Manual interventions (things bot missed)
- Handover edits

If the feedback ticket doesn't exist yet, note "Feedback tracking not yet configured" and skip this section.

### Source 9: Latency metric (approximate)

For each bug where `[OTA-Monitor] Spike offered` exists, calculate approximate latency:
- Latency = `[OTA-Monitor] Spike offered` comment timestamp minus the bug's `updated` timestamp at the time the Spike was offered
- This is approximate — Jira doesn't expose per-label timestamps directly
- Report average and max latency across all bugs this week
- If latency is consistently hours (not minutes), the bot is a true safety net (human acted first). If latency is near zero, the bot is the primary detector — consider increasing scan frequency.
- If data is insufficient (fewer than 2 bugs), note "Insufficient data for latency tracking" and skip.

## Output Format

Render the handover as an HTML artifact using `send_html_to_thread` for proper table formatting. Also post a text summary to the channel.

### Text summary (channel top-level):

```
<!subteam^STE7S7ZU2|@ota-monitor> — OTA RIT Handover — Week of <MONDAY_DATE> to <FRIDAY_DATE>

Active blockers: <N> | Resolved this week: <N>
Open PRs: <N> | Merged PRs: <N>
Items needing immediate attention: <N>

Full report attached below ⬇️
```

### HTML artifact sections:

**Immediate Attention Required**
Items the incoming monitor should act on first thing Monday. For each:
- Bug/PR link
- What needs to happen
- Exact command or button to use

**Active Upgrade Blockers by Stage**
Table with: Bug, Summary, Stage, Component, Priority, Days in Stage
Sorted by days-in-stage descending (oldest first).

**Items Waiting on Component Teams**
Table with: Bug, Summary, Team, Days Waiting, Last Reminder
Highlight any >7 days (escalation territory).

**Items Waiting on OTA Review**
Table with: Bug, Summary, Spike Link, Days Waiting

**Resolved This Week**
Table with: Bug, Summary, Resolution, Fix Version

**Graph-Data PRs (OTA-relevant only)**
Bug-linked PRs appear inline with their bug in the Active Blockers table — do NOT repeat them here.
Open: Table with PR, Title, Age, CI, Review (only PRs not tied to a specific bug)
Merged: Table with PR, Title, Merged Date

**Pipeline Health**
- Total FAILEDs this week: <N>
- FAILEDs with PRs: <N> (handled)
- FAILEDs without PRs: <N> (gaps)
- Notable failures: brief summary if any
- Stale "Recommend waiting" messages: list any advisories that have been in "Recommend waiting" status for >14 days. These are unusual and worth the incoming monitor's attention.

**Bot Performance**
Table with:
| Metric | Count |
| Alerts posted | <N> |
| Spikes created | <N> |
| Review buttons posted | <N> |
| PRs linked | <N> |
| fixedIn updates | <N> |
| Auto-transitions | <N> |
| Skips (potential FPs) | <N> |
| Missed detections | <N> |
| Avg detection latency | <N>h |
| Max detection latency | <N>h |

**Key Contacts**
- OTA team: #forum-ocp-updates
- Pipeline: #osus-graph-data-automation
- Bot issues: #chai-users
- Escalation: tag OTA reviewers in PR, or ping <!subteam^STE7S7ZU2|@ota-monitor>

**Components That Required Manual Routing**
List any OCPBUGS components where Cyborg lookup failed and the OTA Monitor had to specify the project manually. If the same component appears 3+ weeks in a row, it needs an override entry in ota-component-mapping.yaml or a Cyborg MR to fix the mapping.

**Notes for Incoming Monitor**
- Any in-flight items that need context
- Any unusual patterns observed this week
- Any skill prompt adjustments recommended

## Verification

After generating the handover, verify:
- Count of bugs in "Active Blockers" sections == count from Source 1 JQL
- Count of "Resolved" == count from Source 3 JQL
- Count of "Open PRs" == count from Source 4 GitHub query

If any count doesn't match, flag it in the output: "⚠️ Count mismatch: handover shows <X> but query returned <Y>. Data may be incomplete."

## Error Handling

- If a JQL query fails: include the section header with "[DATA UNAVAILABLE — Jira API error]" instead of omitting the section entirely
- If GitHub query fails: same pattern
- If Slack search fails: note "Pipeline health data unavailable this week"
- NEVER produce an empty handover. Always post something, even if partial.

## Bounds

- If >100 active blockers (unusual): show top 20 by priority, note total count
- If feedback ticket has >200 comments: read only the last 50
- Max execution time: if gathering is taking very long, post what you have with [PARTIAL] markers
