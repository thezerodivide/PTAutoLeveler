# PTAutoLeveler Spec

Originally drafted Sep 28, 2026 · @Shane. Rewritten 2026-10-04 to match [DL-001](docs/decision_log.md) (v1.0 is solo only; acceptance criteria AC-1 to AC-18).

**How to read this document.** Corrections are visible: `[CORRECTED 2026-10-04, DL-001: ...]` marks text that replaces earlier text, `[CLARIFIED 2026-10-04, DL-001: ...]` marks a refinement, and `[DEFERRED]` marks content kept but not part of v1.0. The decision log's Active overrides index lists every supersession. Developer-stated facts are labeled with their evidence tier; nothing here has been verified by a script or a log unless it says so.

## Overview

PTAutoLeveler is a MacroQuest Lua orchestrator for Project Triune (RoF2 emu) that walks a character through a user-configured, ordered list of leveling/AA phases. It handles phase selection, AA XP %, travel and DZ management, and hands combat, AA spending, item evolution and death recovery to existing scripts.

**v1.0 is solo only.** [CORRECTED 2026-10-04, DL-001: the earlier text said v1.0 ships solo and duo together with no public beta. Duo is future functionality. The first release is for the developer alone; other players come after the tool is stable enough for outside testing and after a UI review pass.]

~~Feasibility: high. Complexity: ~8/10 with duo mode. Estimated size: 2,500–3,500 lines including UI. Estimated delivery: ~2–2.5 weeks.~~ [CORRECTED 2026-10-04, DL-001: estimates removed; they are not requirements and were written for a solo-plus-duo scope.]

**In scope (v1.0, solo)**

- Create (in the UI) and validate an ordered progression plan.
- Resolve the current phase from level and total assigned AAs; resume from anywhere.
- Set AA XP % per phase.
- Detect, identify, preserve or leave the active DZ.
- Travel: Bazaar → direct waypoint to the leveling zone.
- Create the phase's Respawning DZ via the Priest of Triune, and enter it.
- Start and monitor TAC and PTAAPlanner; start PTItemEvolver when the plan turns it on; yield to PTDeathRecovery.
- Detect stalls and recover or pause.
- A functional status window and plan creation in the UI.
- A diagnostic log.

**Out of scope**

- Choosing, optimizing or generating routes, zones or PTAutoRoute paths.
- Deciding AA purchases, combat, item evolution logic, death recovery.
- Hard-coding any example route. Example plans are illustrative only.
- Duo (future functionality, see "Future functionality: duo").
- Non-Respawning DZs. [CLARIFIED 2026-10-04, DL-001: lockout of 14 hours and mobs that do not respawn; the developer uses Respawning only.]
- PTAutoRoute second legs (every leveling zone must have a Bazaar waypoint inside it), open-world farming (every phase farms inside a DZ), and more than one character.

## Confirmed facts and their evidence

Nothing below has been checked by a script or a log. All are the developer's own in-game observation or statement (2026-10-04) unless noted; a screenshot is noted where the developer supplied one. These are inputs to the design, not guarantees.

- The Priest of Triune creates the DZ (hail, then `/say Respawning` or clicking the link). The DZ is entered through the Priest (`/say ready` or the link) or through the "Travel to Expedition" button on the Waypoint map in the Bazaar.
- Lockouts are per zone and per mode. Respawning is 30 minutes; Non-Respawning is 14 hours. Several can exist at once. They show in the Expedition Information window under "Outstanding Expedition Timers" (screenshot).
- The "Bazaar and Back" AA, `/alt activate 331`, transports the character to the Bazaar; used while already in the Bazaar it returns the character to where it was. Its refresh reads `0:02:00` (screenshot).
- The Priest is reachable from every waypoint landing point via `/nav`.
- The server owner was told of the design on 2026-09-29 and acknowledged it ("If you don't do it someone else will lol", Discord screenshot). [CORRECTED 2026-10-04, DL-001: the earlier text said "Server staff support full automation". The chat shows an acknowledgment, not an explicit approval.]

## Locked design decisions

These were settled in the feasibility discussion (not recorded at the time). Each is a hard rule unless marked.

| # | Topic | Decision |
| --- | --- | --- |
| 1 | Phase ordering | Strict. The current phase is the first phase whose own requirements are not met. No later phase runs while an earlier one is unmet. |
| 2 | Completion rule | Level ≥ target level AND assigned AAs ≥ target AAs. Omitted targets are ignored. |
| 3 | Below minimum level | User config error. Halt with a message naming the phase and reason. |
| 4 | Stall handling | See Stall detection. |
| 5 | Retries | Configurable retry counter; exhausted → PAUSED\_ERROR with reason. |
| 6 | Same zone/DZ | Compared by zone shortname + DZ name, both stored per phase. |
| 7 | DZ rebuilds | Never. Create only when no DZ is active; leave only when the active DZ is wrong for the current phase. A created DZ starts a lockout on creating another for the same zone and mode. [CORRECTED 2026-10-04, DL-001: was "Respawning DZ creation incurs a 30-minute lockout"; the lockout is per zone and per mode.] |
| 8 | No-progress remedy | Return to Bazaar and re-evaluate. Never pause-and-rebuild. |
| 9 | Bounce limit | Separate per-phase counter for stall-driven Bazaar returns; exhausted → PAUSED\_ERROR. Bad TAC setup is a user problem, not engineered around. |
| 10 | DZ creation retries | Re-check for an active DZ before every attempt; detect the lockout and pause. |
| 11 | Tunables | Initial timer values are implementation detail, tuned from logs and live use. |
| 12 | Travel (v1.0) | Direct Bazaar waypoint only. No PTAutoRoute support in v1.0. A zone is eligible only if the Bazaar waypoint map has a waypoint inside it. [CLARIFIED 2026-10-04, DL-001: eligibility is the reason for the rule; it avoids building routes to a DZ-creation point.] |
| 13 | DZ required | Every phase farms inside a DZ. No open-world farming. TAC never starts outside the verified DZ instance. |
| 14 | v1.0 scope | ~~Solo and duo (one killer + one leveler). Solo is an internal milestone only; no public beta.~~ [CORRECTED 2026-10-04, DL-001: v1.0 is solo only.] |
| 15–19 | Duo roles, DZ ownership, parking, grouping, deaths | [DEFERRED] Future functionality. Text preserved under "Future functionality: duo". |
| 20 | Self-explaining UI | [DEFERRED] The UI must be functional for the developer in v1.0 and receives a review pass before outside testers. Pre-flight check, one-click example plans, inline validation and tooltips are not v1.0 requirements. |
| 21 | DZ mode | New. Respawning only. |

## Phase model

A plan is an ordered list of phases. Each phase has these fields:

| Field | Required | Notes |
| --- | --- | --- |
| name | Yes | Display label. |
| minLevel | No | Level the character must be at or above to start this phase. |
| targetLevel | One of target fields | Completion when level ≥ this. |
| targetAssignedAA | One of target fields | Completion when total assigned AAs ≥ this. Assigned, not banked. |
| aaXpPct | Yes | 0–100, set on phase start. |
| zoneShortName | Yes | Leveling zone. It must have a Bazaar waypoint inside it. |
| dzName | Yes | Required DZ identity (Respawning). Every phase farms in a DZ. |
| waypoint | Yes | Bazaar waypoint; must land directly in the leveling zone. |
| autoRoute | No | Reserved for a later version; not supported in v1.0. |

The fields list the zones eligible to use; they are not a route description.

**Phase resolution (on start, resume, and after every phase completion)**

1. Walk phases in order.
2. The first phase whose completion rule is not met is the current phase.
3. If the character's level is below that phase's minLevel → config error, halt.
4. If every phase is complete → done.

**Worked example.** A level 52 character with 1,000 assigned AAs, using the example plan, resolves to the Umbral Plains 50→50 / 2,400 AA phase and farms at 100% AA XP. This is intended.

**Same-instance transitions.** If the next phase has the same zoneShortName and dzName, update AA XP % and keep farming. No travel, no DZ change.

## Config validation

The plan is validated on load, before any action. Any failure halts the script and the UI shows the phase and reason.

- [ ] No phases defined.
- [ ] A phase has neither targetLevel nor targetAssignedAA.
- [ ] targetLevel below minLevel.
- [ ] aaXpPct outside 0–100.
- [ ] targetAssignedAA set with aaXpPct = 0 (the phase can never complete).
- [ ] Missing zoneShortName, dzName or waypoint.
- [ ] autoRoute is set (not supported in v1.0).
- [ ] targetAssignedAA set but PTAAPlanner is not available.
- [ ] Character level below the resolved phase's minLevel (checked at resolution time).

## State machine

One loop drives everything: every entry to farming passes through Evaluate, so resume, phase change and stall recovery share one path.

[CORRECTED 2026-10-04, DL-001: the state machine diagram that was embedded here is missing from this document. The earlier text named Evaluate, Check DZ, Travel, Create/Enter, FARM and PAUSED\_ERROR and said "8 states + 2 global". The list of states is an open design item, to be drawn at the design stage and agreed with the developer.]

A same-zone/same-DZ phase change passes straight through the DZ and travel steps as no-ops.

**Per-state contract (AC-17)**

- On entry, re-read real game state. Never trust remembered state.
- Every state has a timeout.
- Every state counts retries; exhausted → PAUSED\_ERROR with the state name and reason.
- Log every transition with timestamp, from/to state and reason.

## DZ rules

DZ state is independent of physical zone: returning to the Bazaar never implies the DZ was left.

| Situation | Action |
| --- | --- |
| No active DZ | Create the phase's Respawning DZ via the Priest of Triune, then enter it through the Priest. |
| Active DZ matches the phase | Preserve and re-enter it. Never recreate. The mechanism is open (AC-7). |
| Active DZ does not match | Leave it, confirm it's gone, then create the correct one. |
| Leave fails | Retry up to the limit, then PAUSED\_ERROR. |
| Creation attempt about to run | Re-check for an active DZ first; a prior "failed" attempt may have succeeded. |
| Lockout for that zone and mode | PAUSED\_ERROR with the lockout reason. Never retry creation into a lockout. [CLARIFIED 2026-10-04, DL-001: per zone and per mode.] |
| Entered instance doesn't match phase | Treat as wrong DZ; return to Evaluate. |
| DZ state can't be read reliably | PAUSED\_ERROR. Never guess. |

**Candidate mechanisms (to verify on the server).** From the spec's first draft: `DynamicZone` TLO for existence and name; `/dzquit` to leave; Priest of Triune dialog for create and enter. Developer-stated (see Confirmed facts, not yet verified by a script): `/say Respawning`, `/say ready`, `/nav` to the Priest, `/alt activate 331` to return to the Bazaar, and the Expedition Information window's timer list for lockouts.

## Stall detection (solo)

[CORRECTED 2026-10-04, DL-001: replaces the earlier stall tree. The progress rules, the split rule and the PTAAPlanner check changed. Check 5 (stuck → camp point) is left out on purpose and revisited when TAC's role is discussed.]

A stall is no meaningful progress while farming (AC-11).

- **0% AA XP (level XP only):** no level XP gain within N minutes. A new level resets the counter.
- **100% AA XP:** no AA XP gain within N minutes, or a PTAAPlanner error. An AA XP gain resets the counter. A PTAAPlanner error is an immediate stall.
- **Split:** a stall in either measurement counts as a stall. The developer may revisit this once testing occurs.
- **Suspension:** the stall timer is suspended while the character is dead or PTDeathRecovery holds control, and resumes when control returns.
- The log states which measurement was read and why a stall was concluded.

**On a stall (AC-12)**, check in order and act on the first that applies:

1. Wrong zone or DZ instance → go to Evaluate.
2. TAC not running → start it, count a retry.
3. TAC running but paused → resume it, count a retry.
4. Otherwise, still no progress after a flat X-minute timer → treat as a TAC setup issue. Return to Bazaar (using the "Bazaar and Back" AA, never while already in the Bazaar), re-evaluate, count a bounce. Never rebuild a DZ.
5. Retries or per-phase bounces exhausted → PAUSED\_ERROR, e.g. "No progress in [zone] after K attempts. Check TAC camp/pull settings."

~~Check 5 of the first draft (stuck: position unchanged, nav failing → return to camp point, restart TAC)~~ and ~~check 7 (banked AAs rising but assigned AAs flat → check PTAAPlanner)~~ are not in v1.0. The PTAAPlanner error is now an immediate stall.

**Deliberately not handled:** mobs present but not engaged, or attacking without response. These resolve through death and PTDeathRecovery. The bounce loop from bad TAC setup is accepted; the bounce counter bounds it.

## Supporting script contract

Only one script drives the character at a time. PT scripts share a small contract, ideally over MQ Lua actors.

- **Status:** `idle / running / busy / paused / error / done`
- **Commands:** `start / pause / resume / stop`
- **Ownership:** a script claims and releases control; others wait while it's held.

| Script | Role | Required | Integration |
| --- | --- | --- | --- |
| TAC | Combat and farming | Yes | Not PT-owned. Detect via Lua script status plus progress metric. Started only inside the confirmed DZ instance (AC-9). |
| PTDeathRecovery | Death recovery | Yes | Claims control on death; PTAutoLeveler waits, then re-evaluates (AC-13). |
| PTAAPlanner | AA spending | AA phases | Status + "plan exhausted" signal (AC-14). |
| PTItemEvolver | Item evolution | Optional | Started only if the plan turns it on; status only; a failure warns and farming continues (AC-15). |
| PTAutoRoute | Second travel leg | Per phase | Not used in v1.0. |
| MQ2Nav | Navigation | Yes | Direct use for moving to the Priest. |

## UI

[CORRECTED 2026-10-04, DL-001: duo items and the self-explaining UI pass are deferred. The UI must be functional for the developer in v1.0 and receives a review pass before outside testers.]

A plan must be creatable in the UI (AC-1). A separate editor window is not preferred but acceptable if necessary; whether it is needed is a design item.

**Status window (AC-16)**

- Start / Pause / Resume / Stop
- Current phase (name and index of total)
- Current state and last transition reason
- Current zone and active DZ
- Level vs target level
- Assigned AAs vs target AAs; banked AAs
- Current AA XP %
- Helper status: TAC, PTAAPlanner, PTItemEvolver, PTDeathRecovery
- Retry and bounce counters
- Error banner with the halt reason (config error or PAUSED\_ERROR)

**Plan creation**

- Ordered phase list with the fields from the Phase model.
- Validation against the Config validation rules (AC-1).

[DEFERRED] Saving and loading named plans, add/remove/reorder/duplicate, pre-flight check on Start, actionable-error polish, one-click example plans (solo and duo), inline validation as the user types, hover tooltips, and safe defaults for every setting. Revisit trigger: the review pass before outside testers.

## Acceptance criteria (v1.0, solo)

Approved by the developer item by item, DL-001. Each says what is checkable locally by a test and what only live.

- **AC-1:** a plan can be created in the UI and loaded. It is checked against the Config validation rules before any action; an invalid plan halts with the phase and reason. Start resolves the current phase from level and total assigned AAs, with strict ordering, shows it in the status window, and takes no in-game action. *Local: validator, resolver. Live: real level and AA readings, the UI.*
- **AC-2:** the script reads the active DZ state from the game (none, matching, not matching), logs the decision, and takes no action. *Local: decision logic. Live: the reading.*
- **AC-3:** a wrong DZ is left and confirmed gone. If leaving fails after the retry limit, it pauses with a named reason. It never creates a DZ here. *Local: decision and retry logic. Live: the leave and its confirmation.*
- **AC-4:** with no DZ active, the script reaches the phase's zone through the Bazaar waypoint map and confirms the zone. If not in the Bazaar, it first returns using the "Bazaar and Back" AA (`/alt activate 331`), after reading the zone, and never activates it while already in the Bazaar. It pauses with a named reason after the retry limit. *Local: the zone-guard logic. Live: travel.*
- **AC-5:** in the zone with no DZ active, the script goes to the Priest (`/nav`) and creates a Respawning DZ. It re-checks for an active DZ before every attempt and confirms the DZ matches the zone and mode afterward. On a lockout for that zone and mode it pauses and never retries creation. It pauses with a named reason after the retry limit. *Local: re-check, retry, pause. Live: the Priest interaction, the lockout indication.*
- **AC-6:** it enters the created DZ through the Priest and confirms the instance matches. A mismatch means wrong DZ: return to evaluation, do not farm. It pauses with a named reason after the retry limit. *Local: match decision. Live: entry.*
- **AC-7:** with an active matching DZ and the character not in it, the script gets back in and confirms the instance. It never creates a DZ in this case. It pauses with a named reason after the retry limit. The mechanism is not part of the criterion. *Local: decision logic. Live: re-entry.*
- **AC-8:** when a phase starts, it sets the AA XP % and confirms it. For a same-zone-and-DZ phase it changes only that value. It pauses with a named reason after the retry limit. *Local: decision and retry. Live: setting and reading the value.*
- **AC-9:** TAC starts only after the game confirms the instance matches the phase, and is confirmed running. It pauses with a named reason after the retry limit. *Local: the start rule. Live: TAC's state.*
- **AC-10:** when a phase's completion rule is met (level ≥ target AND assigned AAs ≥ target; omitted targets ignored), the script re-resolves, proceeds to the next phase (AA XP % change only if zone and DZ match), and stops with "plan done" when every phase is complete. *Local: check, re-resolution, done decision. Live: real readings.*
- **AC-11:** stall detection as described under Stall detection. *Local: rules, resets, immediate stall, suspension, timer. Live: real XP values and helper states.*
- **AC-12:** the stall remedies as described under Stall detection. *Local: order of checks, counters, pause rules. Live: TAC's state, the actions.*
- **AC-13:** on death, the script yields to PTDeathRecovery, takes no action, suspends the stall timer, and re-evaluates from the start when control is released. It pauses with a named reason if control is not released within a time limit. *Local: yield, re-evaluate, limit. Live: death detection, the real return.*
- **AC-14:** in a phase with an assigned-AA target, it starts PTAAPlanner and confirms it running (pause after the retry limit). An error is an immediate stall. A plan exhausted with the target unmet pauses with a configuration error. *Local: start, retry, pause. Live: PTAAPlanner's real status.*
- **AC-15:** PTItemEvolver starts only if the plan turns it on. A failure to start gives a warning and farming continues. Its status never counts toward stall rules. *Local: on/off, warn-and-continue. Live: its real status.*
- **AC-16:** the controls and status window listed under UI. *Local: field selection, error-to-banner mapping (if the UI logic is separate from drawing). Live: appearance and behavior in ImGui.*
- **AC-17:** the per-state contract (re-read, timeout, retry count, named pause). *Local: counters, clock. Live: real readings.*
- **AC-18:** `PTAL_<server>_<character>.log` under `macroquest/logs/PTAutoLeveler/`, the build version on every line, and every transition, command, decision reading, retry, stall, bounce, pause and halt logged with its reason, including where evidence ends. *Local: content and format. Live: whether a real log explains a real failure.*

## Open unknowns and tunables

Each unknown is resolved by an in-game spike during the build step that needs it. [CORRECTED 2026-10-04, DL-001: rows for the duo handshakes were moved to the deferred duo section; step numbers in the original refer to the first draft's build order, which is itself reconsidered at the design stage.]

| Unknown | Notes |
| --- | --- |
| Detecting and identifying the active DZ | Candidate: `DynamicZone` TLO. |
| Reliably leaving a DZ | Candidate: `/dzquit`. |
| Returning to the Bazaar | Developer-stated: `/alt activate 331`. Whether it works on cooldown is unknown. |
| Priest of Triune dialog sequence | Developer-stated: hail, `/say Respawning`, `/say ready`. |
| Re-entering an existing correct DZ | Two options: copy PTDeathRecovery's method, or trigger it with `/echo You died.`. |
| Lockout refusal message text | Not shown in the developer's screenshots. |
| AA window controls for reading/setting AA XP % | |
| TAC readiness signal | |
| Handshakes: PTDeathRecovery, PTAAPlanner, PTItemEvolver | |
| TAC's role and the stuck/camp-point check | Revisit when TAC's role is discussed. |
| What Pause, Resume and Stop do; what happens at plan completion | |
| Whether plan creation needs a separate editor window | |
| The state list | The diagram is missing (see State machine). |

**Tunables** (initial values are implementation detail, tuned from logs):

- N — no-progress window
- X — flat TAC-setup timer before bouncing to Bazaar
- Retry limit (global, optional per-state override)
- K — per-phase bounce limit
- Per-state timeouts
- Time limit for PTDeathRecovery to release control

## Stewardship

PTAutoLeveler should be a good citizen of the game world at scale. [CORRECTED 2026-10-04, DL-001: the earlier text said "Server staff support full automation"; see Confirmed facts for what the Discord chat supports.]

**Already built into the design**

- DZ-only farming: never competes with players for open-world camps.
- No DZ rebuilds: no creation spam, no lockout churn.
- Pause on failure: broken setups stop quietly instead of thrashing.

**Release principles** (applying when the tool goes to outside testers)

- **Conservative defaults.** Ship slower retries, real bounce limits and pause-over-recover.
- **Rate-limited interactions.** Small delays and backoff on Priest of Triune, waypoint and AA window actions.
- **Clear docs on what it does not do.** Instance-only, no open-world farming, no DZ rebuilds, no route or AA decisions.
- **Useful logs.** Every transition, retry, bounce and halt reason.
- **Helper version checks.** [DEFERRED] Verify compatible versions of TAC and PTAAPlanner on start; pause with a clear message on mismatch. Revisit trigger: the review pass before outside testers.

## Build order

[CORRECTED 2026-10-04, DL-001: the first draft's 12 steps assumed duo. Steps 8–10 are deferred. The order below is the first draft's, kept as a starting point; the design stage reconsiders it against AC-1 to AC-18 and agrees it with the developer.]

Each step is testable in game on its own; steps 1–2 resolve most unknowns cheaply.

1. Plan loader, validator, phase resolver and status UI. Read-only.
2. DZ detection, identification and leaving.
3. Travel: return to Bazaar and direct waypoint to the leveling zone.
4. Priest of Triune create/enter, pre-create DZ re-check and lockout detection.
5. AA XP % setter.
6. Helper contract, FARM loop and stall tree.
7. PTDeathRecovery handoff. **Solo complete: v1.0 candidate.**
8. ~~Leveler ↔ killer messaging~~ [DEFERRED]
9. ~~Killer support mode~~ [DEFERRED]
10. ~~Duo deaths and duo stall adjustments~~ [DEFERRED]
11. UI review pass, example plans, inline validation, tooltips (before outside testers). [DEFERRED to the review pass]
12. Private testing with trusted testers, docs. [DEFERRED]

**After v1.0:** duo; PTAutoRoute second-leg travel.

## Future functionality: duo [DEFERRED]

Not part of v1.0 (DL-001: "Duo is future functionality"). The first draft's design is kept here so it is not lost; it is not approved and was written before the rest of this spec changed. It must be re-baselined against the spec when duo is taken up. Revisit trigger: v1.0 is stable and outside testing has begun.

Duo was one killer plus one leveler. Solo was to be duo with both roles on one character.

| Role | Duo |
| --- | --- |
| Leveler | Runs the plan, travels, creates the DZ, verifies the group, parks at the DZ entrance, never fights. |
| Killer | Support mode: no plan of its own, follows the leveler's commands, runs TAC. |

**Duo flow (first draft)**

1. Leveler resolves the phase, sets AA XP % and handles its own DZ state.
2. Leveler travels via Bazaar waypoint and creates or re-enters the DZ.
3. Leveler confirms the killer is in its group (grouped by the user before Start).
4. Killer joins the DZ from the Bazaar waypoint map.
5. Leveler verifies the killer is grouped and in the DZ instance, then tells it to start TAC.
6. Leveler stays at the DZ entrance. XP and loot are zone-wide.

**Keeping the killer in sync (first draft).** The leveler is the only source of truth. It announces a target state on a heartbeat (phase, zone, DZ name, sequence number); the killer acts only on the newest and reconciles (wrong DZ → leave; not in the leveler's DZ → return to Bazaar and join via waypoint map; not grouped → report; in place → run TAC). A restarted killer asks the leveler for the target; a restarted leveler re-evaluates and re-announces; a DZ-changing transition tells the killer to stop TAC and return to the Bazaar. A broken group → PAUSED\_ERROR.

**Deaths (first draft).** No global pause. A dead killer: the leveler waits at the entrance. A dead leveler: the killer keeps farming. The stall timer suspends while either is in PTDeathRecovery. Progress is measured on the leveler; a killer that stops answering the heartbeat counts as a retry, then PAUSED\_ERROR.

**Helper scripts by role (first draft)**

| Script | Duo leveler | Duo killer |
| --- | --- | --- |
| TAC | Never | Runs |
| AA XP % | Set | Untouched |
| PTAAPlanner | Runs | Not needed |
| PTDeathRecovery | Runs | Runs |
| PTItemEvolver | Configurable | Configurable |

Duo unknowns from the first draft: cross-client messaging (MQ Lua actors), the killer joining the DZ from the waypoint map, detecting a broken group, and whether PTItemEvolver progress is XP- or kill-driven.
