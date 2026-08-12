# Onboarding Request for #chai-users

Post this message in #chai-users. Attach the 4 prompt files from `prompts/` in the thread.

---

🚀 **New Persona Onboarding Request: OTA Monitor**

**Persona:** ota_monitor
**Display name:** OTA Monitor
**Description:** Cincinnati pipeline health and UpgradeBlocker lifecycle automation

Design doc: https://shahsahil264.github.io/ota-monitor-design/
Repo: https://github.com/shahsahil264/ota-monitor-design

I'll post the 4 instruction files (01_role.md, monitor-enriched.md, monitor-brief.md, weekly-handover.md) in the thread.

---

**Channels**

Bot output channel:
- #ota-monitor-bot — https://redhat.enterprise.slack.com/archives/C0BQ59S21H6

Knowledge-source channels (indexed, bot does NOT post here):
- #osus-graph-data-automation — Cincinnati pipeline messages
- #forum-ocp-updates — OTA team discussions

---

**Knowledge Sources**

- Slack history: #osus-graph-data-automation + #forum-ocp-updates (standard sync)
- Knowledge profile: ocp_standard
- GitHub: openshift/cincinnati-graph-data (branch: master), openshift/cluster-version-operator
- Jira: OCPBUGS, OTA projects
- Web: https://github.com/openshift/enhancements/tree/master/enhancements/update/update-blocker-lifecycle

---

**Tools**

- Jira — expanded scope: label updates on existing OCPBUGS issues (only_self_created override needed). If override isn't possible, fallback: bot suggests label change, human applies manually.
- GitHub: fork-based PR workflow for openshift/cincinnati-graph-data
- Remote Workspace: custom env ota_dev (Go + Python 3), coordinator Jira callbacks preferred
- Cyborg/orgdata: component → team → project routing (primary source for Spike targeting)

Jira create access verified: ship-help-jira@redhat.com has Spike + create permissions in MON, CORENET, MCO, WINC, SPLAT, OCPEDGE.

---

**Jira Operations**

- Read: OCPBUGS, OTA, MON, CORENET, MCO, WINC, SPLAT, OCPEDGE
- Create: Spike issues in component team projects — approval-gated
- Update labels: On existing OCPBUGS bugs (full-set replace) — approval-gated
- Comment: [OTA-Monitor] prefixed audit trail — direct, no gate
- Link: \"is related to\" between Spike and OCPBUGS bug — direct
- Transition: Status changes on bot-created Spikes — direct

---

**Scheduled Tasks** (3 tasks, weekdays only, all post to #ota-monitor-bot)

All top-level posts @mention @ota-monitor so the current rotation monitor gets notified. Times are 12h apart to cover EU + US timezones.

1. monitor-enriched — `0 10 * * 1-5` (10:00 UTC). Always posts.
2. monitor-brief — `0 22 * * 1-5` (22:00 UTC). Posts only if action needed.
3. weekly-handover — `0 22 * * 5` (Fri 22:00 UTC). Always posts.

---

**Configuration**

- DM Access: Restricted to #forum-ocp-updates members
- VK Review: In-channel (#ota-monitor-bot), shared with other personas
- OTEL: `otel: {public: true}`, `otel_link: true` on all tasks

---

**Questions for the team:**

1. Are #osus-graph-data-automation and #forum-ocp-updates already indexed?
2. Can only_self_created guard on priv_jira_update_issue be overridden for this persona? Use case: approval-gated label updates on existing OCPBUGS bugs for the UpgradeBlocker lifecycle.
3. Is there a TTL on action buttons?
4. Are button callback turns strictly serialized per thread?
5. Can GitHub PR activity notifications be enabled for bot-opened PRs in openshift/cincinnati-graph-data?

---

I'll post the 4 instruction files in the thread after this message.
"