# OTA Monitor — 3pm Brief Scan

This task runs every weekday at 3pm ET. It posts ONLY if something needs action. If nothing changed since the 9am run, it stays silent (heartbeat to thread only).

## Execution

Follow the SAME Steps 1-6 as monitor-enriched.md:
1. Run both JQL queries (primary + secondary) via query_jira
2. Filter candidates by labels
3. Inspect candidates with get_jira_issue (batches of 10, compact between)
4. Classify using the same 10 priority-ordered rules
5. Pipeline check (best-effort Slack search)
6. Batch alert check (>3 new bugs → summary table)

## Differences from 9am enriched run

- **NO daily status section.** Do not post the "Active Upgrade Blockers: N" summary. If you find yourself composing a full daily status section, STOP — that belongs in the 9am run, not here. Post only actionable alerts.
- **Silent if nothing needs action.** If all bugs have existing markers and no new alerts are generated, do NOT post to the channel top-level.
- **Heartbeat to thread only.** If nothing changed, post to a thread (not top-level): "✓ OTA Monitor ran at {TIME} UTC. No changes detected."
- **Same error reporting.** If the run fails, post error to channel top-level.

## When to post

Post to the channel top-level ONLY if:
- A new UpgradeBlocker bug is detected (Rule 6)
- An impact statement is ready for review (Rule 4)
- A fixedIn signal is detected (Rule 1)
- A stale bug reminder is due (Rule 5)
- A pipeline FAILED needs attention
- An orphaned spike is detected
- The run itself failed (error)

If NONE of the above apply, post heartbeat to thread and stop.
