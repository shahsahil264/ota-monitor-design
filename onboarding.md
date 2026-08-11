# Onboarding Request for #chai-users

Post this message in #chai-users. Attach the 4 prompt files from `prompts/` in the thread.

---

🚀 **New Persona Onboarding Request: OTA Monitor**

**Persona:** ota_monitor
**Display name:** OTA Monitor
**Description:** Cincinnati pipeline health and UpgradeBlocker lifecycle automation

I will deliver 4 instruction files in the thread: 01_role.md, monitor-enriched.md, monitor-brief.md, weekly-handover.md.

---

**Channels**

Bot output channel:
- #ota-monitor-bot — https://redhat.enterprise.slack.com/archives/C0BQ59S21H6

Knowledge-source channels (indexed, bot does NOT post here):
- #osus-graph-data-automation — Cincinnati pipeline messages
- #forum-ocp-updates — OTA team discussions

Testing channel:
- #test-ota-monitor-bot — https://redhat.enterprise.slack.com/archives/C0BNVHYMEH5

---

**Knowledge Sources**

- Slack history: #osus-graph-data-automation + #forum-ocp-updates (standard sync)
- Knowledge profile: ocp_standard
- GitHub: openshift/cincinnati-graph-data (branch: master), openshift/cluster-version-operator
- Jira: OCPBUGS, OTA projects
- Web: https://github.com/openshift/enhancements/tree/master/enhancements/update/update-blocker-lifecycle

---

**Tools**

- Jira — expanded scope: label updates on existing OCPBUGS issues (only_self_created override needed)
- GitHub: fork-based PR workflow for openshift/cincinnati-graph-data
- Remote Workspace: custom env ota_dev (Go + Python 3), coordinator Jira callbacks
- Cyborg/orgdata: component → team → project mapping

---

**Scheduled Tasks** (3 tasks, weekdays only, all post to #ota-monitor-bot)

1. monitor-enriched — `0 13 * * 1-5` (9am ET). Always posts.
2. monitor-brief — `0 19 * * 1-5` (3pm ET). Posts only if action needed.
3. weekly-handover — `0 20 * * 5` (Fri 4pm ET). Always posts.

---

**Configuration**

- DM Access: Restricted to #forum-ocp-updates members
- VK Review: In-channel (#ota-monitor-bot), shared with other personas
- OTEL: `otel: {public: true}`, `otel_link: true` on all tasks

---

**Questions for the team:**

1. Are #osus-graph-data-automation and #forum-ocp-updates already indexed?
2. Does ship-help-jira@redhat.com have create permissions in MON, CORENET, MCO, WINC, SPLAT, OCPEDGE? Does Spike issue type exist in all?
3. Can only_self_created guard on priv_jira_update_issue be overridden?
4. Is GitHub PR activity notification available for bot-opened PRs?
5. Button TTL? Callback serialization per thread?
6. Context/timeout limits for 10-15 get_jira_issue calls after pre-filtering?
7. Can you share a redacted scheduled task prompt example?
8. Is mid-turn compact() recommended for batch Jira processing?
9. Scope precedent — any comparable personas?
10. Can the RIT manual Google Doc be shared with chai-bot@redhat.com?

---

I'll post the 4 instruction files in the thread after this message.
