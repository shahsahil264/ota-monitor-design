# OTA Monitor — Complete Project Context

This document captures the full history, architecture, design decisions, findings, and learnings from building the OTA Monitor Chai Bot persona. Written as a handoff/context document for anyone (human or AI) picking up this project.

---

## 1. What This Project Is

**The problem:** The OTA Monitor is a weekly rotating role on PIXAA's Rotational Interrupt Team (RIT) at Red Hat. The person holding this role is responsible for watching OpenShift's Cincinnati update service for "UpgradeBlocker" bugs — bugs serious enough that a specific upgrade path (e.g., 4.20.x → 4.21.0) needs to be blocked until fixed. Doing this manually takes ~1-2 hours/day: watching Jira, creating impact statement requests, reviewing responses, hand-writing blocked-edge YAML configs, opening PRs, tracking pipeline health, and writing a weekly handover doc.

**The solution:** We built `ota_monitor`, a persona on **Chai Bot** (Red Hat's internal agentic Slack platform, GA, 100+ personas). It automates detection, drafts Jira tickets and PRs, and asks a human to approve before anything consequential happens. Human-in-the-loop by design — the bot proposes, a human always approves creation/PR actions.

**Platform:** Chai Bot. Not Claude Code, not a custom bot — a managed persona system where behavior is defined entirely through markdown instruction files that get compiled into an LLM's system prompt for scheduled tasks and conversational interactions.

---

## 2. The State Machine (core domain model)

UpgradeBlocker bugs move through 4 Jira label stages:

```
UpgradeBlocker + ImpactStatementRequested
  → UpgradeBlocker + ImpactStatementProposed
    → UpgradeBlocker + UpdateRecommendationsBlocked
      → (fixedIn added when fix ships)
```

**Two label categories:**
- **`UpgradeBlocker`** and **`Upgrades`** — ADDITIVE. Stay on the bug forever. Never removed, even after resolution.
- **`ImpactStatementRequested` / `ImpactStatementProposed` / `UpdateRecommendationsBlocked`** — MUTUALLY EXCLUSIVE. When transitioning, ADD the next label FIRST, then REMOVE the previous one. This order is crash-safe (if the process dies between the two operations, the bug never vanishes from JQL entirely).

**Classification priority (first match wins) when a bug has multiple labels:**
1. UpdateRecommendationsBlocked (most advanced)
2. ImpactStatementProposed
3. ImpactStatementRequested
4. UpgradeBlocker only (least advanced)

---

## 3. Idempotency (critical design pattern)

**3-layer idempotency**, checked in this priority order before taking ANY action:
1. **Linked issues** (primary, most robust) — is a Spike already linked to this bug?
2. **Bot comments** (fast path) — does an `[OTA-Monitor] <action>` comment exist?
3. **Labels** (tertiary) — is the expected label already present?

Only act if ALL THREE indicate the action hasn't been taken. Linked-issue checks survive comment deletion; comments are the cheap fast-path check.

**Comment format** (the bot's memory — no external database, Jira IS the state):
```
[OTA-Monitor] Spike offered
[OTA-Monitor] Impact statement spike created: <SPIKE_KEY>
[OTA-Monitor] Response detected
[OTA-Monitor] Review offered
[OTA-Monitor] Blocked: <risk_name> — <PR_URL>
[OTA-Monitor] fixedIn: <version> — <PR_URL>
[OTA-Monitor][Feedback] SKIP — <BUG_KEY>
```

Every Jira issue key referenced in a comment should be a full markdown link (`[KEY](url)`), not bare text — bare text doesn't render clickable in Jira.

**Stateless design:** each scheduled run is fully independent. No cross-run memory beyond what's readable from Jira (labels, comments, linked issues). This means the system self-heals — if a run crashes mid-action, the next run detects the actual state and continues correctly.

---

## 4. The 3 (really 4) Human Approval Gates

| Gate | Trigger | Buttons |
|------|---------|---------|
| 1 — Create Spike | New UpgradeBlocker detected | `[Create Impact Statement in {PROJECT}]` `[Skip]` `[Escalate]` |
| 2 — Accept/Reject | Impact statement ready | `[Accept — Block Edge]` `[Request Revision]` `[Not a Blocker]` |
| 3 — Add FixedIn | Fix detected (via Rule 1) | `[Add FixedIn]` `[Skip]` |
| 4 — Extend Risk | Pipeline FAILED needs risk extension | `[Extend Risk]` `[Skip]` |

Originally designed with 5 gates, collapsed to 3+1 to reduce approval fatigue (the #1 threat to adoption identified during design).

**Auto-actions (no human gate needed):**
- Auto-transition when a Spike moves to "Code Review" (Rule 3) — mechanical, not a judgment call
- Orphaned Spike auto-close when the bot itself created the Spike and the linked bug closes
- Stale/escalation reminders (informational, no action required)

---

## 5. Detection Rules (priority order, first match wins, STOP at first match)

Defined in `monitor-enriched.md` Step 4:

- **Rule 1** — fixedIn available (secondary JQL bugs only). Three checks: A) bot comment marker, B) fix signal (fixVersion or advisory comment), C) does the actual graph-data YAML already have fixedIn set (prevents false positives on pre-bot-era bugs). **Critical sub-rule:** when multiple files share a risk name across consecutive versions, only the LATEST affected version's file should have `fixedIn` — earlier versions correctly have none. Don't flag earlier versions as "missing" fixedIn.
- **Rule 2** — active blocker, count only, no alert (UpdateRecommendationsBlocked + Open)
- **Rule 3** — auto-transition (mechanical). Spike reaches Code Review → ADD ImpactStatementProposed, REMOVE ImpactStatementRequested → **intentional fall-through to Rule 4** so the human gets both notifications (transition + review request) in one alert.
- **Rule 4** — impact statement ready for review → post Gate 2 buttons
- **Rule 4b** — review reminder if `[OTA-Monitor] Review offered` is >24h old
- **Rule 5** — stale impact statement request (>48h). Timestamp source: `[OTA-Monitor] Spike offered` comment, OR fallback to the linked Spike's creation date if the Spike was created manually outside the bot's workflow (this fallback was a fix — originally Rule 5 silently never fired for manually-created Spikes).
- **Rule 5b** — escalation warning (>7 days), cites the enhancement doc directly
- **Rule 6** — new UpgradeBlocker. Checks: no `[OTA-Monitor] Spike offered` comment, no linked Spike (ANY status — open OR closed; a closed Spike means already triaged, e.g. via [Not a Blocker], and must not re-alert), no `[OTA-Monitor][Feedback]` comment. Includes a duplicate-Spike check for the same component before alerting.
- **Rule 6b** — already offered, skip silently
- **Orphaned Spike check** — bug Closed + linked Spike still open. If bot-created (has the marker comment), auto-close. If human-created, alert only. **Clone check added:** before auto-closing, verify no OTHER open bug is also linked to the same Spike (a clone bug for a later version might still need it).

---

## 6. Pipeline Check (two-phase, Step 5)

**Root cause of an early bug:** indexed Slack search uses SEMANTIC similarity, not keyword matching. Searching for "FAILED" returns semantically-related content, not literal matches — so it often returned nothing even when real FAILED messages existed, and the bot couldn't tell "search failed" from "search worked, found nothing."

**Fix — two-phase approach:**
- **Phase 1:** broad semantic query ("recent pipeline activity") to confirm the channel is indexed/accessible. Record message count.
- **Phase 2:** `result_grep`/`grep_text` on Phase 1's results for the literal substring "FAILED" — a text match, not semantic.
- Report `"Healthy (searched N messages, 0 FAILEDs)"` vs `"data unavailable (channel not indexed or search failed)"` — never the ambiguous old wording "No FAILED messages detected."

**Before alerting on any FAILED:** check if the associated PR is already merged — most FAILEDs are phantom failures from timing (pipeline retried after a promotion already succeeded).

**Known VK lesson violated and fixed:** the original Step 5 alert text for "branch already exists" said "Delete branch from bot fork" — directly contradicting an onboarding lesson: "NEVER delete branches from the openshift-ota-bot fork, most FAILEDs are phantom failures." Fixed to say "investigate first, never recommend deletion as default action."

---

## 7. Component → Project Routing

**Cyborg (Red Hat's org data registry) is primary.** Routing algorithm:
1. Check a small override config (originally 20+ entries, pruned to 2, currently near-zero as Cyborg MRs land) for verified wrong/missing Cyborg mappings
2. OrgData `jira-OCPBUGS-*` lookup by exact component name → filter out meta-projects (OCPBUGS/RFE/OCPSTRAT) → suggest if 1 candidate
3. Try parent component if sub-component isn't individually registered (safe — sub-components always belong to the parent's team)
4. Ask human if all else fails
5. **Human confirms EVERY time**, regardless of which step found the answer

Query tip: use exact Jira component names, not colloquial ones — e.g. "Auth" not "Authentication", "Management Console" not "Console". Case-insensitive fuzzy search handles most slug differences.

---

## 8. Skill 3 — Blocked-Edge PR Generation

**Composition, not duplication:** delegates YAML generation to `/propose-risk` (a Claude Code command already living in `openshift/cincinnati-graph-data/.claude/commands/`). The bot's job is orchestration (when/who/after), not YAML-writing logic.

**Real blocked-edge YAML structure** (verified against actual files in the repo):
```yaml
to: 4.20.24              # unquoted, single version
from: 4[.]20[.].*         # character-class regex (backslash-escape also technically valid, validator doesn't enforce style)
fixedIn: 4.20.25          # ONLY on the file representing the LAST affected version
url: https://redhat.atlassian.net/browse/CONSOLE-5337   # ALWAYS the Spike URL, NEVER the OCPBUGS URL
name: ControlPlaneStatusGreyIcon   # CamelCase, no version suffix — enforced by validator regex ^[A-Z][A-Za-z0-9_]*$
message: |
  Plain customer-facing description. NEVER reference the OCPBUGS URL here either — only the Spike.
matchingRules:
- type: Always   # or type: PromQL — if PromQL, the expression MUST evaluate to exactly 0 or 1; empty/multi-result = "Unknown" to the CVO
```

**File naming convention:** `blocked-edges/<version>-<RiskName>.yaml` — one file per affected z-stream version. Only the file for the LAST affected version gets `fixedIn`.

**What `hack/validate-blocked-edges.py` actually enforces (confirmed by reading the source):** risk name matches `^[A-Z][A-Za-z0-9_]*$`, and `to`/`from` are present. It does NOT enforce regex escaping style, does NOT require `matchingRules`. **Validation passing is necessary but not sufficient — a YAML can pass and still violate real production convention.**

**Pre-Flight Checklist (a fix we added):** before presenting generated YAML for human approval, the bot must grep 1-2 real recent files in `blocked-edges/` as ground truth and diff its output's structure against them — do not rely solely on memorized documentation, which is often incomplete.

**Branch naming:** `blocked-edges/OCPBUGS-XXXXX-{action}` (e.g., `-create`, `-fixedin`, `-extend`) — deterministic, prevents orphan branches on crash and collision between actions on the same bug.

**Clone via bot's own fork → PR to upstream**, never push directly to upstream. Clone UPSTREAM (not the fork) when generating — always based on latest master, fork is just the push target.

---

## 9. The Cincinnati Stabilization Pipeline (separate system, context only)

`openshift-ota-bot` is NOT a separate bot — it's a GitHub account used by `hack/stabilization-changes.py`, a ~900-line script in `cincinnati-graph-data` that runs on a polling loop, promoting versions through channels (candidate → fast → stable → eus) based on soak-time delays and errata publication. It posts FAILED/success/waiting messages to `#osus-graph-data-automation`, which is what our pipeline check watches.

The `autoExtend` YAML field is misleadingly named — it does NOT auto-extend anything. It's an opaque string appended to the FAILED message text for human context; the stabilization script never fetches or parses it. Not worth investing in for automation.

---

## 10. Deployment Architecture (Chai Bot specifics)

- Prompt files live in `openshift-eng/ship-help-bot` (private repo), under `ship_help_bot/shared/instructions/ota_monitor/01_role.md` and `ship_help_bot/shared/instructions/scheduled/ota_monitor_{enriched,brief,weekly_handover}.md`
- **Placeholder syntax constraint (hard-won lesson):** ship-help-bot's build pipeline runs Python `.format()` on instruction files, THEN validates via regex `\{([A-Za-z_][A-Za-z0-9_]*)\}` that no bare `{IDENTIFIER}` patterns remain. Double-brace escaping (`{{X}}`) does NOT work as an escape — `.format()` un-escapes it to `{X}` before the regex check runs, so it still fails. **The only fix: use angle brackets (`<IDENTIFIER>`) instead of curly braces for all LLM-facing placeholders.** This applies to ALL personas' instruction files, not just ours.
- Design repo (source of truth we control): `shahsahil264/ota-monitor-design` — prompts mirrored to `~/ota-monitor-prompts/` locally and pushed via PRs into ship-help-bot
- Persona name: `ota_monitor`, serves `#ota-monitor-bot` (C0BQ59S21H6)
- Two scheduled tasks: `ota_monitor_enriched` (10:00 UTC weekdays, always posts) and `ota_monitor_brief` (22:00 UTC weekdays, silent unless action needed), plus `ota_monitor_weekly_handover` (Friday 22:00 UTC)
- **On-demand triggering works**: `@chai-bot run the enriched scan now` or `re-scan now` in the persona's channel executes the same logic as the scheduled task immediately (`run_on_request: true` on the task config) — critical for demos, since waiting for the 12h cron cycle is impractical.
- **Snowflake sync lag (~3 hours):** Jira comment/description content the bot reads comes from a Dataverse database (`CLOUDRHAI_MARTS`) in Snowflake, not live Jira. A just-created or just-edited Spike may not be visible to the bot for up to ~3 hours. Workaround: paste the content directly into Slack conversationally instead of relying on the bot re-reading the Spike.
- **Approval self-approve inconsistency:** creating a Jira issue allows self-approval ("Peer review recommended but you may self-approve"). Closing/modifying a GitHub PR requires a DIFFERENT team member to approve — self-approval explicitly refused even for a demo PR with "do not merge" in the body. This is a known open friction point, tracked but unresolved as of this writing.
- **`update_only_self_created` permission gap:** label updates on Jira issues NOT created by the bot's own service account sometimes get blocked, despite an onboarding answer stating this restriction was overridden to `false`. Comments post fine regardless of creator; only label updates hit this. Root cause unconfirmed — tracked as an open platform issue.

---

## 11. Deployment Timeline (what shipped when)

- **Aug 13** — Initial persona onboarded, PR #384 (Batch A: daily status format overhaul, PR filtering v1, JQL date cutoff v1, run metrics logging, subteam mention syntax)
- **Aug 14-18** — Snowflake credentials issue found and fixed (root cause: a persona config override clobbered inherited Snowflake credentials); @ota-monitor mention fixed
- **Aug 18** — PR #384 deployed
- **Aug 18-19** — Check C YAML verification + 6 edge-case fixes (Rule 6 re-alert prevention, phantom-FAILED PR-merge check, Rule 5 manual-Spike fallback, orphaned-Spike clone check, missing-PR flag, known-edge-cases docs) — these were incorrectly marked "Resolved (deployed)" in the tracker but actually never made it into a real ship-help-bot PR (a real gap between "tracker status" and "actual deployed code" — always verify against the live diff, not the status label)
- **Aug 19** — PR #417 opened (two-phase pipeline check only, initially)
- **Aug 21-22** — Full end-to-end demo run (test bug OCPBUGS-112668) surfaced 13 real findings (see Section 12) — PR #417 expanded to consolidate Batches B+C+D
- **Aug 24-25** — Curly-brace placeholder CI failures found and root-caused (see Section 10); design repo fully converted to angle-bracket notation; PR #417 verified clean against design repo HEAD via direct diff comparison (not just trusting status messages)
- **Aug 25, ~9:22-9:38 PM** — PR #417 merged and deployed
- **Aug 26** — Scheduled 10:00 UTC run was MISSING entirely (no post at all) — confirmed via on-demand test that the prompt/persona itself was functionally intact post-deployment, isolating the issue to the scheduler layer, not a prompt regression. Reported to platform team, unresolved as of writing.
- **Aug 25-27** — Second full demo (OCPBUGS-113992, OLM component) run against the fully-deployed prompts — confirmed stage-grouped format, PR filtering, missing-PR flags all working correctly in production

---

## 12. Findings From Live Testing (the full audit trail)

These were discovered by actually running the bot through complete lifecycles and checking its output against real files/history, not just trusting its descriptions. **Recurring meta-lesson: always verify bot claims against actual state (real files, real diffs, real Jira data) — the bot repeatedly presented confident, specific claims that turned out to be incomplete or wrong until externally checked.**

1. **Status field doesn't change** — non-issue. The bot manages labels, not Jira Status (New/POST/MODIFIED/etc.), which reflects real engineering progress on a fix. Confirmed correct behavior.
2. **`update_only_self_created` blocks label updates** on bugs not created by the bot's service account — contradicts an onboarding answer claiming this was overridden. Real, unresolved, platform-level.
3. **Comments render as plain text, not clickable links** — fixed by instructing the bot to use full markdown links for Jira issue key references.
4. **Duplicate `[OTA-Monitor] Spike offered` comments** posted ~2h apart — likely caused by rapid on-demand re-scans hitting the Snowflake sync-lag window before the bot could see its own prior comment. Unresolved, platform-level.
5. **Bot extracted fabricated data from unanswered example placeholder text** — a high-severity finding. The impact statement template has `(example: ...)` text under each question; when a question was left blank, the bot used the EXAMPLE text as if it were a real answer, generating a blocked-edge YAML from fabricated data. Fixed: bot must now detect "only the example remains" and stop to ask, rather than guess.
6. **Missing required answer silently omitted** rather than flagged — same root cause/fix as #5.
7. **Message field leaked the private OCPBUGS URL** even though the `url` field correctly used the Spike URL — the "Spike URL only" rule was only being checked against one field, not the generated customer-facing text as a whole. Fixed by extending the rule's scope explicitly. The bot did NOT self-correct this on the first attempt after being told to fix other things — had to be told twice.
8. **Risk naming convention initially wrong** (kebab-case + version suffix instead of CamelCase, no version) — bot corrected once asked and cited the real convention, but didn't proactively verify against real files before presenting.
9. **Snowflake sync lag confirmed live** — matches documented known edge case, no new fix needed, just empirical confirmation.
10. **`matchingRules` field missing entirely** from first-generated YAML — a required-by-convention (not validator-enforced) field omitted; bot acknowledged as its own mistake and fixed once flagged.
11. **Regex escaping convention overstated as an absolute rule** — bot claimed one style was "the" standard when the validator doesn't enforce any style at all (confirmed by reading `hack/validate-blocked-edges.py` directly). Real files in the repo use both styles.
12. **NOT a bug:** revealing a known fix version upfront in an impact statement causes `fixedIn` to be set immediately at blocked-edge creation (correct, matches real production pattern) — but this means the same test bug can't also demo the separate "fix ships later" detection flow. A test-design lesson, not a defect.
13. **Self-approve inconsistency** between Jira issue creation (self-approve allowed) and GitHub PR state changes (different-teammate approval required, even for closing a demo PR) — real friction for solo testing, unresolved, platform-level.
14. **VK-lesson-taught-but-never-codified gap** — the fixedIn-on-latest-version rule was literally VK lesson #4 from onboarding (Aug 13), but Rule 1's Check C still flagged an earlier version's file as deficient in a REAL production bug (OCPBUGS-87100 / CONSOLE-5337, not a demo). Root cause, per the Chai team: VK lessons are technically available via a `check_verified_knowledge` tool during scan execution, but the LLM follows scheduled-task instructions step-by-step and won't spontaneously consult VK unless a step explicitly says to. **This led to a full audit of all 8 onboarding VK lessons against the prompt files — found 2 more of the same pattern** (see below) and confirms an architectural principle: VK only reliably affects conversational reasoning, not mechanical scan logic, unless explicitly written into the instructions.

**Full VK lesson audit results (all 8 original onboarding lessons):**
| # | Lesson | Status before audit |
|---|--------|---------------------|
| 1 | Check PR merge status before FAILED alerts | ✅ Already codified |
| 2 | CVO PromQL must return 0 or 1, empty = Unknown | ❌ Gap — never codified, fixed |
| 3 | `from` regex: unaffected→affected only, never affected→affected | ❌ Gap — never codified, fixed |
| 4 | fixedIn only on last affected patch | ❌ Gap — the one that started the audit, fixed |
| 5 | Always use Spike URL, never OCPBUGS | ✅ Already codified (and extended) |
| 6 | Label spelling: UpdateRecommendationsBlocked (plural) | ✅ Covered by construction |
| 7 | ADD label first, REMOVE second | ✅ Already codified |
| 8 | NEVER delete branches from bot fork | ❌ Gap — contradicted by the bot's own alert text, fixed |

**Recommendation for future work:** any new VK lesson taught to a persona with scheduled/automated tasks should get a corresponding, explicit instruction in the relevant prompt file — otherwise the lesson only helps when a human happens to prompt the bot conversationally into re-deriving it, not during actual automated execution.

---

## 13. Design Principles That Emerged

1. **Never trust a status label over the actual diff/state.** The tracker said "Resolved (deployed)" for changes that were never actually in a real PR. Multiple times, checking the live GitHub diff directly caught discrepancies between what was reported as pushed and what was actually there.
2. **VK/taught-knowledge ≠ applied logic.** If a rule matters during automated execution, write it into the instruction file directly. Don't rely on the model "remembering" a lesson unless the step explicitly says to check it.
3. **Validation passing ≠ correctness.** A schema/lint validator only enforces what it's coded to enforce. Real convention (file naming, required-but-unenforced fields like `matchingRules`) needs the bot to check against real examples, not just pass the validator.
4. **Human approval gates should be cheap to click, expensive to bypass.** The 3-gate design (down from 5) balances safety against approval fatigue — the biggest threat to actual adoption of an automation tool is people getting tired of clicking things.
5. **Fabrication risk is real and specific:** LLMs filling out structured templates will use unedited example/placeholder text as if it were real data unless explicitly told to distinguish "answered" from "still just the example." This is a general pattern worth checking for in any agent that reads human-filled forms.
6. **Semantic search ≠ keyword search.** Don't assume a "search for X" instruction reliably finds literal occurrences of X if the underlying search is embedding-based/semantic. Use a two-phase approach: broad semantic query to confirm access, then literal grep for the specific signal.
7. **Self-approve policies vary by action type and aren't always intuitive** (Jira create: self-approve OK; GitHub PR close: needs a different approver) — worth explicitly asking the platform team rather than assuming consistency.
8. **Compression via on-demand triggering is essential for demos of scheduled-task-driven bots** — waiting for real cron cycles (12h+) makes live demos impractical. If the platform supports on-demand invocation, use it.
9. **Deployments during a scheduled task's execution window can silently kill that run.** A deployment sent SIGTERM to a pod 24 seconds into the Aug 26 enriched scan; the pod died after its grace period and the replacement started too late to matter for that cycle — producing a complete, silent "missing run" with no error posted anywhere. Treat scheduled-task execution windows (plus a buffer) as deployment blackout periods, or the failure will look like a mysterious platform bug when it's actually just bad timing.
10. **A second independent occurrence turns "maybe it's my test setup" into a confirmed production pattern.** When only one person (during testing) hits an issue, it's reasonable to wonder if it's specific to the test bugs or environment. The moment a second, unrelated person hits the identical issue on a real bug they didn't design, that ambiguity is gone — escalate accordingly, and don't let a real, universal, production-blocking issue sit at the same priority as a one-off testing quirk.

---

## 14. Presentation / Demo Materials Built

- **Design doc (GitHub Pages):** https://shahsahil264.github.io/ota-monitor-design/ — Overview, Lifecycle (13-phase manual-vs-automated breakdown with Slack mockups), Architecture, Prompt Files tabs
- **Slide deck structure** (in progress, Google Slides): Title → What's an OpenShift Update → What's the OTA Monitor role → "A Day in the Life: The Manual Way" (timeline narrative, NOT a dry bullet list) → The Solution + Lifecycle diagram → "A Day in the Life: With the Bot" (mirrored timeline) → Live Demo (screenshot walkthrough) → Constraints (honest limitations) → Results → What's Next → Thank You
- **Demo test bugs used** (all cleaned up after each session): OCPBUGS-112668 (kubelet, first full demo), OCPBUGS-113992 (OLM, second demo post-deployment), OCPBUGS-114720 (RHCOS, screenshot session) — pattern: `[TEST-DEMO][DO NOT TRIAGE]` prefix, fictional scenario clearly marked, cleaned up (bug closed, Spike closed, PR closed unmerged, branch deleted) immediately after use
- **Key demo learnings:**
  - On-demand scan triggering (`@chai-bot run the enriched scan now` / `re-scan now`) makes live demos feasible
  - Real end-to-end interaction has genuine wait time (workspace spin-up for PR generation, Snowflake sync lag) — screenshot-based narration over slides works better than continuous screen recording for avoiding dead air
  - Show the manual way and bot way as two separate sequential blocks, not interleaved per-phase — avoids audience "screen-switching whiplash"
  - A "Day in the Life" narrative (showing fragmentation/context-switching cost) lands harder than a dry step-by-step list of what the role involves
  - Don't overstate the "manual" baseline — check what tooling already exists (e.g., Cyborg/OrgData for routing) before claiming something is 100% manual

---

## 14b. Findings From Real Production Use (post-launch, by other team members)

Once the persona was live and other RIT rotation members started using it directly (not just testing), new findings surfaced independently:

15. **`update_only_self_created` confirmed again, on a real bug, by a different team member.** A teammate hit the identical label-update block on OCPBUGS-114342 — had to manually add the lifecycle label and tell the bot before it could proceed. This confirms the issue isn't specific to test bugs or how they were created — it's universal, since NO real production bug is ever created by the bot's own service account. Escalated with this as fresh evidence; still unresolved as of writing.

16. **Component ownership and the bug's assignee's team can disagree — VK lesson, now codified.** On OCPBUGS-114342 (a MetalLB/FRR-K8s BGP issue), OrgData component lookup suggested OPNET/NE, but the actual assignee was on the Telco Platform Networking team (CNF project) — a specialist working on a shared/cross-cutting component. Took multiple corrections and the teammate proving it via the actual Cyborg org config before the bot routed correctly. Fix: added a new routing step that cross-checks the assignee's team (via Cyborg org config, `config/structures/ecosystem/teams/`) against component ownership, and — critically — surfaces BOTH to the human when they disagree rather than silently picking one. Neither signal is unconditionally more correct than the other.

17. **`link_jira_issues` tool behavior for `link_type="Blocks"` is inverted from its own documentation.** Verified empirically by a teammate: calling with `inward_issue_key=A, outward_issue_key=B` produces "B is blocked by A" per the tool's stated description, but actually produces the opposite relationship in practice. To make "A blocks B", you must pass `inward_issue_key=A (the blocker), outward_issue_key=B (the blocked one)` — backwards from what the tool description implies. **Does not require a prompt change for us** — our own design uses the "is related to" link type for Spike-to-bug linking, not "Blocks", so this bug doesn't affect our current Gate 1 flow. Captured as general tool-behavior knowledge in case "Blocks" is ever used elsewhere.

**Reinforced meta-lesson:** issues found during solo testing can look like they might be artifacts of the test setup. The moment a second, independent person hits the exact same issue on a real bug they didn't design or expect, it stops being a "maybe it's my test bug" question and becomes a confirmed, universal, production-blocking pattern — worth re-escalating with that framing rather than assuming it's already been deprioritized correctly.

## 15. Open Items / Not Yet Resolved (updated with platform team responses, Aug 31)

- **`update_only_self_created` blocking label updates — ROOT CAUSE CONFIRMED IN CODE, still needs a fix applied.** Platform team traced it exactly: `comment_on_jira_issue`, `transition_jira_issue`, `assign_jira_issue`, and `link_jira_issues` have NO ownership check and work on any issue. Only `priv_jira_update_issue` (→ `update_jira_issue_impl`) — the sole tool that can modify labels — has an `only_self_created` guard in `JiraConfig`, defaulting to `True`. It compares the issue creator's email against the bot's service account email and blocks the operation on mismatch. The Aug 13 onboarding override to `false` for this persona was never applied, was applied incorrectly, or was reverted — root cause unconfirmed on that specific point. Two fix options given: (a) actually set `update_only_self_created: false` in the persona's `JiraConfig` in `workspaces.yaml` (the fix that was supposed to happen at onboarding), or (b) expose the existing `add_issue_labels` function (`client.py:744`, atomic label-add, no ownership check, currently only wired to the `jira_solve` RWS plugin profile) as a general-purpose tool, architecturally cleaner since label-add is lower-risk than a full issue update. **Confirmed on 3 separate bugs across 2 different people as of Aug 31 — flagged urgent.** This blocks the core workflow on every single new UpgradeBlocker detection in production, since no real bug is ever created by the bot's own service account.

- **Scheduler missed run (Aug 26) — RESOLVED.** Root cause: a deployment (commit `c26d58ae` → `c26c6642`) landed while the 10:00 UTC enriched scan was mid-execution. The deployment sent SIGTERM 24 seconds into the run; the pod was killed after a 120-second grace period, and the replacement pod didn't start until 10:20 — past the point where anything would post for that cycle. Not a scheduler bug or a prompt regression — a deployment-timing collision. **Learning for future deployments: avoid deploying during scheduled task execution windows.** OTA Monitor's enriched scan runs ~10:00-10:10 UTC and brief scan ~22:00-22:10 UTC on weekdays — treat these (plus a buffer) as deployment blackout windows for this persona.

- **Pipeline check "data unavailable" despite confirmed-indexed channel — still open, no new progress.** Same 3 investigation angles as before stand: whether Phase 1's query references the channel by name vs. a stable ID, whether the persona's datastore config actually includes this channel, whether search filter behavior differs between scheduled and interactive execution contexts. Needs platform-side investigation into the persona's researcher behavior during an actual scheduled run — not something visible or debuggable from outside.

- **Duplicate idempotency-marker comments — explained, not a bug, operational mitigation only.** Confirmed: `[OTA-Monitor]` comments write via the Jira REST API in real-time, but `get_jira_issue` reads comments from the Snowflake `CLOUDRHAI_MARTS` database, which syncs periodically (not real-time). Two on-demand scans run close together can race this sync gap — the second scan genuinely cannot see a marker the first scan just wrote. At normal 12-hour scheduled-run cadence, the sync lag (empirically ~3 hours) is comfortably shorter than the gap, so this is a real risk specifically during rapid on-demand re-scans (e.g., demo/testing sessions), not during normal production operation. No platform fix expected — mitigate by avoiding back-to-back on-demand re-scans when testing.

- **Self-approve policy inconsistency (Jira vs. GitHub) — by design, logged as a feature request, not a bug.** Jira issue creation and GitHub PR/issue state changes go through separate approval policy configurations. Jira allows self-approval because the bot generates the content and the human reviews/confirms it inline. GitHub write operations default to a stricter policy because they affect shared, external repositories with broader blast radius. The policy today does not distinguish "close" (low-risk) from "merge" (high-risk) — logged as a reasonable ask for more granular GitHub operation policies, not an active bug fix.

- **VK-to-prompt verification as a standing process gap — recommendation given, not yet a formalized process.** No automated or process-level link exists between "a VK lesson gets approved" and "the corresponding scheduled-task prompt actually gets updated to reflect it." This has now caused 2 confirmed real gaps (fixedIn-on-latest-version, and implicitly the routing/assignee-team lesson would have had the same fate if not manually codified). Recommended to the Chai team as a process improvement; no confirmation yet that this will become a formal step for other personas.

- Google Doc integration proposal (reading/writing the team's weekly RIT status doc for narrative context beyond what Jira labels capture) — designed and proposed to the platform team (#chai-users and #wg-ge-agentic-sdlc), platform capability unconfirmed as of writing.

---

## 16. Key Files (design repo: shahsahil264/ota-monitor-design)

- `prompts/01_role.md` — persona identity, state machine, idempotency, comment format, action buttons, impact statement template, Spike creation details, component routing, Pre-Flight Checklist, known edge cases
- `prompts/monitor-enriched.md` — 10:00 UTC scan: JQL queries, filtering, 10 detection rules, two-phase pipeline check, PR filtering, daily status format (2 templates: quiet day / busy day), run metrics logging
- `prompts/monitor-brief.md` — 22:00 UTC scan, same rules as enriched, silent unless action needed
- `prompts/weekly-handover.md` — Friday handover, 9 data sources, HTML artifact, verification/cross-check logic
- `prompts/ota-component-mapping.yaml` — routing overrides (near-empty now, Cyborg-primary)
- `index.html` — GitHub Pages design doc
- `README.md` — status tracking
- `PROJECT-CONTEXT.md` — this file

---

## 17. Quick Reference: What the Bot Can and Cannot Do

**Fully automatic (no human click):**
- Detection and alerting
- Auto-transition on Spike → Code Review
- Orphaned Spike auto-close (bot-created only)
- Stale/escalation reminders
- Daily status, weekly handover generation
- Duplicate-Spike detection

**Requires human click (approval gate):**
- Creating a Spike
- Accepting/rejecting an impact statement
- Adding fixedIn
- Extending a risk
- Any PR open/close/merge

**Cannot do at all (out of scope by design):**
- CVE triage (needs symbol-level analysis, VEX judgment)
- Release lifecycle scripts (rare, high-impact, manual)
- PR /hold management (judgment call)
- Merging any PR (always human)
- General release-blocker bug triage outside the formal UpgradeBlocker label taxonomy (a real gap identified — the human role's actual scope is broader than what the bot's JQL currently tracks)
