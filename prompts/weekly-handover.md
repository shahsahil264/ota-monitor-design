# OTA Monitor — Weekly Handover

This task runs every Friday at 4pm ET. It generates a structured handover document for the incoming OTA Monitor and always posts to #ota-monitor-bot.

## Execution Structure (Two Phases)

**PHASE 1 — Data gathering** (sources 1-9): For each source, call the tool, extract key data. Call `compact()` after every 3 sources to free context. Store results in compact notes with structured summaries.

**PHASE 2 — Synthesis**: Read compacted notes. Compose HTML artifact. Do NOT re-query sources.

## Data Gathering (Phase 1)

Every number MUST come from a tool query — never estimate or guess.

### Source 1: Active blockers by lifecycle stage
JQL: `project = OCPBUGS AND labels in (UpgradeBlocker, ImpactStatementRequested, ImpactStatementProposed, UpdateRecommendationsBlocked) AND status != Closed ORDER BY priority DESC, updated ASC`

### Source 2: Items waiting on component teams
Filter from Source 1: ImpactStatementRequested. Days waiting, team, reminder status.

### Source 3: Resolved this week
JQL: `project = OCPBUGS AND labels in (UpgradeBlocker) AND status = Closed AND resolved >= -7d`

### Source 4: Open PRs in openshift/cincinnati-graph-data
GitHub API. PR number, title, age, CI status, review status.

### Source 5: Merged PRs this week
GitHub API, "blocked-edges" in title, last 7 days.

### Source 6: Pipeline health
Indexed Slack #osus-graph-data-automation, last 7 days. FAILEDs count, with/without PRs.
Stale "Recommend waiting" messages >14 days.

### Source 7: Bot activity metrics
Scan OCPBUGS for [OTA-Monitor] comments, last 7 days. Count by type.

### Source 8: Feedback signals
OTA-MONITOR-FEEDBACK ticket comments, last 7 days. SKIPs, manual interventions.

### Source 9: Latency metric (approximate)
Latency = Spike offered timestamp − bug updated timestamp. Report avg/max.

## Output Format

HTML artifact via `send_html_to_thread` + text summary to channel.

Sections: Immediate Attention, Active Blockers by Stage, Waiting on Teams, Waiting on OTA, Resolved, PRs (Open + Merged), Pipeline Health, Bot Performance, Key Contacts, Notes for Incoming Monitor.

## Verification

Active Blockers count == Source 1 JQL count. Flag mismatches.

## Error Handling

- Query fails → [DATA UNAVAILABLE] marker. Never empty handover.

## Bounds

- >100 blockers → top 20 by priority
- >200 feedback comments → last 50
- Taking too long → post with [PARTIAL] markers
