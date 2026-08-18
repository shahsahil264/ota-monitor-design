# OTA Monitor — 22:00 UTC Brief Scan

This task runs every weekday at 22:00 UTC (6pm ET / 12am CEST). It posts ONLY if something needs action. If nothing changed since the 10:00 UTC run, it stays silent (heartbeat to thread only).

## Execution

Follow the SAME Steps 1-6b as monitor-enriched.md:
1. Run both JQL queries (primary + secondary) via query_jira
2. Filter candidates by labels
3. Inspect candidates with get_jira_issue (batches of 10, compact between)
4. Classify using the same 10 priority-ordered rules
5. Pipeline check (best-effort Slack search — report message count or "data unavailable")
6. Batch alert check (>3 new bugs → summary table)
6b. Gather PR data (same filter: OTA-relevant only, no dependabot)

## Differences from enriched run

- **NO daily status section.** Do not post the formatted status summary with stage breakdowns. If you find yourself composing a full daily status section, STOP — that belongs in the 10:00 UTC run, not here. Post only actionable alerts.
- **Silent if nothing needs action.** If all bugs have existing markers and no new alerts are generated, do NOT post to the channel top-level.
- **Heartbeat to thread only.** If nothing changed, post to a thread (not top-level): "✓ OTA Monitor ran at {TIME} UTC. No changes detected."
- **Same error reporting.** If the run fails, post error to channel top-level.
- **Same PR filtering.** Only OTA-relevant PRs (same criteria as enriched). No dependabot, no unrelated feature PRs.
- **Same pipeline reporting.** Report message count searched or "data unavailable" — never ambiguous "No FAILED messages detected."

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

## Run metrics

Same as monitor-enriched.md Step 8: after posting (or deciding to stay silent), call `comment_on_jira_issue` on OTA-2104 with this run's counts. This ensures the weekly handover captures all 10 runs (5 enriched + 5 brief).

Format: `[OTA-Monitor][Run] {DATE} {TIME} UTC | alerts:{N} spikes:{N} transitions:{N} reviews:{N} skips:{N} fixedIn:{N} bugs_processed:{N}`

**Error isolation**: If `comment_on_jira_issue` fails, log the error but do NOT abort the run. Run metrics are nice-to-have, not mission-critical.
