# PTAutoLeveler Decision Log

Maintained per [Development_Protocol.txt](Development_Protocol.txt) Section 2. This log records *why* material decisions were made and how they evolved. It is distinct from [SPEC.md](../SPEC.md) (*what* the system must do) — do not merge the two, and do not reconstruct this log from memory; append to it as decisions are made. It is append-only: a correction is a dated addendum, never a rewrite.

Each entry is structured into four labeled blocks per §2: **Requirement** (behavior explicitly stated or approved), **Design choices** (the agreed shape of the solution), **Implementation choices** (details free to decide), and **Open** (unresolved questions). Approving a name or a number is not the same as approving a requirement. The blank shape is [templates/decision_log_entry.md](templates/decision_log_entry.md).

## Active overrides index

Entries below that supersede a specification item are indexed here so the supersession is visible without cross-referencing the whole log against the spec. Format: `**DL-nnn** SUPERSEDES SPEC.md <section or item>: <one line>`.

- **DL-001** SUPERSEDES SPEC.md Overview and Locked decision 14: v1.0 is solo only (the spec says solo and duo ship together).
- **DL-001** SUPERSEDES (defers) SPEC.md Locked decisions 15-19, the "Solo and duo modes" section, build steps 8-10, and the duo status-window and stall items: duo is future functionality, not deleted.
- **DL-001** SUPERSEDES (defers) SPEC.md "Self-explaining UI": a review pass before outside testers, not a v1.0 requirement.
- **DL-001** SUPERSEDES SPEC.md Stall detection check 7 and the split rule: a PTAAPlanner error is an immediate stall; in a split phase a stall in either measurement counts as a stall.
- **DL-001** SUPERSEDES SPEC.md lockout wording (Locked decision 7 and DZ rules): the lockout is per zone and per mode; v1.0 uses Respawning DZs only (30-minute lockout).
- **DL-001** SUPERSEDES SPEC.md Stewardship "Server staff support full automation": reword to what the Discord chat (2026-09-29) supports.
- **DL-001** SUPERSEDES SPEC.md delivery estimates (Overview): removed as non-requirements.
- **DL-001** SUPERSEDES SPEC.md Stall detection check 5 (stuck, camp point): left out of v1.0; revisited when TAC's role is discussed.
- **DL-001** SUPERSEDES (defers) SPEC.md Build order steps 11-12 (UI review pass, trusted testers).
- **DL-001 addendum 2026-10-04** SUPERSEDES SPEC.md Same-instance transitions wording ("update AA XP % ... No travel, no DZ change"): "only" excludes travel and DZ action, not the next phase's required helper actions.
- **DL-001 addendum 2026-10-04** SUPERSEDES the approved AC-12/AC-14 handling of a PTAAPlanner error: a detected error is handled first, with a pause naming PTAAPlanner, not the TAC-setup Bazaar bounce.

`SPEC.md` has not yet been rewritten to match these lines; until it is, this index is the authority where they differ.

---

### DL-001 — PTAutoLeveler v1.0 (solo): retrofit of the pre-method SPEC.md, with approved acceptance criteria

(For a later correction of this entry, do not edit it: append `**Addendum to DL-nnn, YYYY-MM-DD — <what changed and why>.**` at the end of the log.)

- **Status:** Approved 2026-10-04. **Approved by the developer in their own words:** 'Approved as written.' (on the full draft of this entry, shown in the conversation; each criterion's own approval is quoted below). Verification tier: none (documentation only).
- **Story:** PTAutoLeveler is a MacroQuest Lua orchestrator for Project Triune (RoF2 emu). It walks a character through an ordered, user-configured list of leveling and AA phases and hands combat, AA spending, item evolution and death recovery to existing scripts. `SPEC.md` (dated Sep 28, 2026) was written before this method, from a feasibility discussion not recorded anywhere, so this entry is a retrofit. Nothing in it was observed in a log. Developer observations are in-game only and noted as such.
- **Requirement** (each item approved by the developer):
  - **Scope.** "The initial release will be solo (one instance) only. Other players will be using it once we reach a stable enough state for outside testing, but initially it will be just me." "Duo is future functionality. v1.0 will include solo mode only." "The UI has to be functional for me, it will receive a review pass and updates before it would be made available for other players."
  - **Plan creation:** "There will need to be a way to create a plan in the UI." A separate editor is not preferred, but acceptable if necessary.
  - **DZ mode:** Respawning only. "We're not using Non-Respawning DZs due to both the length of the lockout, and that respawning is better for leveling as the mobs will respawn."
  - **Entering a created DZ:** the runner enters it through the Priest of Triune. PTDeathRecovery handles deaths and uses the "Travel to Expedition" button.
  - **Risk:** the developer judges the worst realistic outcome small ("a very small risk profile").
  - **Acceptance criteria**, each approved with "Approved as written" (AC-1 as "Agreed with AC-1 Draft 3", AC-2 as "Agreed with AC-2", AC-3 as "Agreed as written", and the full list as "The list is complete for v1.0 solo"):
    - **AC-1:** a plan can be created in the UI and loaded. It is checked against the Config validation rules before any action. An invalid plan halts with the phase and the reason. Start resolves the current phase from level and total assigned AAs, with strict ordering, shows it in the status window, and takes no in-game action.
    - **AC-2:** the script reads the active DZ state from the game (none, matching, not matching), logs the decision, and takes no action.
    - **AC-3:** a wrong DZ is left and confirmed gone. If leaving fails after the retry limit, it pauses with a named reason. It never creates a DZ here.
    - **AC-4:** with no DZ active, the script reaches the phase's zone through the Bazaar waypoint map and confirms the zone. If not in the Bazaar, it first returns using the "Bazaar and Back" AA (`/alt activate 331`), after reading the zone, and never activates it while already in the Bazaar. It pauses with a named reason after the retry limit.
    - **AC-5:** in the zone with no DZ active, the script goes to the Priest (`/nav`) and creates a Respawning DZ. It re-checks for an active DZ before every attempt, and confirms the DZ matches the zone and mode afterward. On a lockout for that zone and mode it pauses and never retries creation. It pauses with a named reason after the retry limit.
    - **AC-6:** it enters the created DZ through the Priest and confirms the instance matches. A mismatch means wrong DZ, so it returns to evaluation and does not farm. It pauses with a named reason after the retry limit.
    - **AC-7:** with an active matching DZ and the character not in it, the script gets back in and confirms the instance. It never creates a DZ in this case. It pauses with a named reason after the retry limit. The mechanism is not part of the criterion.
    - **AC-8:** when a phase starts, it sets the AA XP % and confirms it. For a same-zone-and-DZ phase it changes only that value. It pauses with a named reason after the retry limit.
    - **AC-9:** TAC is started only after the game confirms the instance matches, and is confirmed running. It pauses with a named reason after the retry limit.
    - **AC-10:** when a phase's completion rule is met (level at or above target AND assigned AAs at or above target; omitted targets ignored), the script re-resolves, proceeds to the next phase (AA XP % change only if zone and DZ match), and stops with "plan done" when every phase is complete.
    - **AC-11:** stall detection. 0% AA XP: no level XP gain within N minutes (a new level resets the counter). 100% AA XP: no AA XP gain within N minutes, or a PTAAPlanner error (an AA XP gain resets the counter; an error is an immediate stall). Split: a stall in either measurement counts as a stall. The timer is suspended while dead or PTDeathRecovery holds control. The log states the measurement used.
    - **AC-12:** on a stall, in order, the first that applies: wrong zone or DZ → evaluation; TAC not running → start (retry); TAC paused → resume (retry); otherwise after a flat X-minute timer → return to the Bazaar, re-evaluate, count a bounce (no DZ rebuild); retries or per-phase bounces exhausted → pause with a named reason. The spec's check 5 (stuck/camp point) is left out on purpose and revisited "when we talk about TAC's role".
    - **AC-13:** on death, the script yields to PTDeathRecovery, takes no action, suspends the stall timer, and re-evaluates from the start when control is released. It pauses with a named reason if control isn't released within a time limit.
    - **AC-14:** in a phase with an assigned-AA target, it starts PTAAPlanner and confirms it running (pause after the retry limit). An error is an immediate stall. A plan exhausted with the target unmet pauses with a configuration error.
    - **AC-15:** PTItemEvolver starts only if the plan turns it on. A failure to start gives a warning and farming continues. Its status never counts toward stall rules.
    - **AC-16:** Start, Pause, Resume and Stop controls, and a status window showing phase (name and index), state and last-transition reason, zone and active DZ, level vs target, assigned vs target AAs and banked AAs, AA XP %, helper status (TAC, PTAAPlanner, PTItemEvolver, PTDeathRecovery), retry and bounce counters, and an error banner with the halt reason.
    - **AC-17:** every state re-reads real game state on entry, has a timeout, counts retries, and pauses with a named error when either runs out.
    - **AC-18:** `PTAL_<server>_<character>.log` under `macroquest/logs/PTAutoLeveler/`, the build version on every line, and every transition, command, decision reading, retry, stall, bounce, pause and halt logged with its reason, including where evidence ends.
- **Design choices** (developer-stated or agreed):
  - **Eligible zones:** a zone is eligible only if it has a Bazaar waypoint inside it, which avoids building routes to a DZ-creation point. Per the developer, "waypoint inside the zone" and the spec's "lands directly in the leveling zone" mean the same.
  - **Log prefix:** `PTAL`. **Test harness:** written fresh for this project, test-first. "I'd rather just reuse the strategy" (only PTAutoRoute's testing strategy is borrowed).
  - **Re-entry (AC-7):** two options, not chosen: copy PTDeathRecovery's method, or trigger PTDeathRecovery with `/echo You died.`.
- **Implementation choices:** N, X, K, the retry limit, per-state timeouts and the PTDeathRecovery time limit are tunables. Their first values are implementation choices (Protocol §15), with no evidence yet. The log line format and the rotation size are implementation choices.
- **Open:**
  - **Game readings:** how the script reads DZ state, the AA XP % controls, TAC's state and the helpers' status. The spec lists these as spikes.
  - **Messages and sequences:** the lockout refusal message text, and the Priest dialog sequence.
  - **Cooldown:** whether `/alt activate 331` works when its refresh is not ready.
  - **Re-entry:** the choice between the two options in AC-7.
  - **TAC's role:** the spec's check 5 (stuck → camp point). Revisit trigger: the TAC-role discussion.
  - **Pause, Resume and Stop:** what each does to the character and to TAC.
  - **Plan completion:** what the script does with TAC and the character at completion.
  - **Split revisit:** the split stall rule ("We may revisit this later once testing occurs", trigger: once testing occurs).
  - **UI shape:** whether plan creation needs a separate editor window.
  - **State machine:** the spec's diagram is missing, so the list of states is Open.
  - **Deferred, each with a revisit trigger still to be agreed:** saving and loading named plans; helper version checks on start; PTAutoRoute second legs; all duo behavior; the self-explaining UI pass (review pass before outside testers); open-world farming.
- **Evidence:**
  - **Verified (read in the repo):** `SPEC.md` content, the PTAutoRoute layout and test strategy, PTDeathRecovery's README line about activating expedition travel, the repo list and visibility.
  - **Developer-observed in game, no log:** the 30-minute Respawning lockout, the 14-hour Non-Respawning lockout, per-zone-and-mode lockouts, the Priest creating the DZ, entry through the Priest (`/say ready`) or the "Travel to Expedition" button, the Bazaar waypoint map, and the Priest being reachable from every waypoint landing via `/nav`.
  - **Screenshots viewed:** the "Bazaar and Back" AA description (toggle behavior; refresh `0:02:00`), the Priest chat and the Expedition Information window, and the Discord chat.
  - **Discord chat (2026-09-29):** shows the server owner acknowledging the design, but not explicit approval of full automation.
  - **Reasoned, not verified:** the Mistmoore `56M` timer is the remainder of a 14-hour lockout.
  - **Unknown:** everything under Open.
- **Review chain:** developer-approved item by item in conversation (quotes above). ChatGPT has not seen this entry. The developer said it joins "if any spec changes are recommended"; the review-folder process in `CLAUDE.md` is not yet set up.
- **Depends on / Shares seams with:** none (first entry). AC-4, AC-11, AC-12 and AC-13 share the "Bazaar and Back" use and the stall timer. AC-13 and AC-7 share the PTDeathRecovery interface.
- **Not yet verified (do not describe as confirmed):** every game reading and mechanism in Open; `/alt activate 331` working on cooldown; the PTDeathRecovery `/echo You died.` trigger; any tunable's value; the lockout refusal text; the "Travel to Expedition" button's exact behavior; the staff statement as written in the spec.
- **Supersedes:** SPEC.md Overview and Locked decision 14 (v1.0 is solo only); Locked decisions 15-19, the Solo and duo modes section, build steps 8-10 and the duo status-window and stall items (deferred); Self-explaining UI (deferred); stall check 7 and the split rule; the lockout wording (per zone and mode, Respawning only); the "Server staff support full automation" sentence; the delivery estimates. All indexed in the Active overrides index above.

**Addendum to DL-001, 2026-10-04 — SPEC.md has now been rewritten to match this entry and committed with the developer's approval ("Go ahead and commit."). The sentence under the Active overrides index saying SPEC.md "has not yet been rewritten" described the state when DL-001 was written and is no longer current; the index lines remain accurate as the record of what was superseded. ChatGPT has not yet reviewed the rewrite.**

**Addendum to DL-001, 2026-10-04 — Spec read-back fixes applied, approved by the developer ("I approve the six fixes."). Added to SPEC.md: an illustrative example plan (developer-supplied; columns "Zone, Min Level, Target Level, XP to AA %, AA Spent Target", read as targetAssignedAA, "assigned, not banked", confirmed: "That is correct."); a note that the carried-over Locked decisions 1-6, 8-11 and 13 were not each re-approved; an Open unknown on whether dzName includes the "(Respawning)" suffix; build step 1 changed from "Read-only" to "no in-game action". No Requirement in DL-001 changed. ChatGPT has not yet reviewed the spec.**
 
**Addendum to DL-001, 2026-10-04 (second) — Changes agreed after the first review of the rewritten SPEC.md, approved by the developer: "I approve 1-7 as reached by consensus."** Source: ChatGPT's review of SPEC.md and DL-001 (hash-identified files, static review) and the exchange between ChatGPT and Claude, which reached technical consensus; the developer approved the consolidated proposal. This addendum does not edit the original entry. Requirement changes are items 1, 2 and 4; items 3 and 5 to 7 are labeling and recording.
1. **Start (Requirement clarification).** AC-1 and AC-2 are the evaluation stages of a single Start run and issue no game actions themselves. When evaluation succeeds and the plan has an incomplete phase, the same run proceeds to the applicable action stages according to the observed game state. An invalid plan halts; an already-complete plan follows AC-10's "plan done" outcome. Each evaluation stage can also be built and tested independently. No new control.
2. **Same-DZ phase change (Requirement clarification).** AC-8's "changes only that value" means no travel and no DZ action. Helper actions the next phase requires (AC-14, AC-15) still apply. Open: whether PTAAPlanner stops on a move from an AA phase to a level-only phase.
3. **PTItemEvolver setting (Open).** Whether the AC-15 on/off setting is plan-wide or per phase. The phase model and UI follow the agreed answer.
4. **PTAAPlanner error (Requirement change).** A detected PTAAPlanner error is an immediate stall. Its remedy takes precedence over AC-12's TAC recovery and Bazaar-bounce checks: pause with a reason identifying PTAAPlanner and including its reported error text when it is available. It does not trigger a restart or a Bazaar bounce. AC-13's death-recovery yield still governs while the character is dead or PTDeathRecovery holds control. The developer approved this remedy after being told its worst case (an unattended character left paused until noticed) and that the risk tolerance is theirs. Open: how the script reliably detects PTAAPlanner errors and plan exhaustion (identify the PTAAPlanner version and source revision being integrated; inspect its status and lifecycle paths; spike where needed).
5. **Labels.** The revisit triggers for helper version checks and for duo are proposed, not approved. The supporting-script contract is a proposed interface, not yet verified to exist in the helper scripts.
6. **Index.** The Active overrides index identifies superseded specification behavior; clarifications and newly identified open questions are recorded in DL-001 and its dated addenda. Index lines were added for stall check 5, build steps 11-12, the same-DZ clarification and the PTAAPlanner error remedy.
7. **AC-17 (Open).** What "every state has a timeout" means for FARM and for a user-initiated pause; the state list is open. The approved AC-17 wording is unchanged.

**Evidence note.** The reviewer and Claude read PTAAPlanner's README and grepped its source; the developer then said the README is out of date, so statements taken from the README (its stop-on-uncertain-purchase behavior, its retry-once behavior, and "no MQ2AASpend plugin is required") are not established for the current version. The recommendation to pause was made on that evidence and on the reviewer's own source reading; the developer approved it as stated above. PTAAPlanner's current functionality will be discussed separately. The dependency lines in CLAUDE.md and the ledger (ProjectTriuneMQ2AASpend as PTAAPlanner's dependency) are left unchanged until then.

**Addendum to DL-001, 2026-10-04 (third) — PTAAPlanner version scope and correction of the evidence note, approved by the developer: "I approve 1-4 and the separate correction."** Source: the developer's statements and the ChatGPT/Claude exchange, which reached technical consensus; the developer approved the consolidated proposal. This addendum does not edit earlier entries.
1. **Requirement:** PTAutoLeveler supports PTAAPlanner v1.0.0 and above only. The developer: "This project would only support PTAAPlanner versions 1.0 and above."
2. **Dependency fact** (the developer's statement, supported by Claude's reading of the PTAAPlanner README): PTAAPlanner v0.2 and later has no MQ2AASpend dependency. The developer: "PTAAPlanner versions 0.2 and onward do NOT have any dependency on MQ2AASpend." The MQ2AASpend repositories exist only for people who want an older PTAAPlanner.
3. **Correction of the evidence note in the second addendum.** That note said the developer called PTAAPlanner's README out of date, so statements taken from it were not established. That was Claude's misreading of the developer's earlier remark. The developer confirms the PTAAPlanner README is current. The note is withdrawn; the earlier entries are preserved as history. The statements taken from the README (a recoverable error is retried once, unsafe identity or confirmation errors stop Auto Spend, an uncertain purchase outcome is never treated as permission to spend again) are therefore README claims about v1.0.0, not yet verified behavior of the integrated script.
4. **Inspection baseline (reported evidence from Claude's inspection only; the reviewer could not retrieve it):** PTAAPlanner v1.0.0 (the `VERSION` constant in `aaplanner.lua` on main, and a release tag v1.0.0) at revision a01290f445a9bcf08297d3221222d444091ee7b5, committed 2026-09-29. This is the initial baseline; it does not establish compatibility with any later version.
5. **Open (not a requirement):** how PTAutoLeveler enables and confirms PTAAPlanner Auto Spend, and how the configured purchase safety checks interact with TAC's activity. The README describes a UI control for enabling Auto Spend; an external activation mechanism has not been established. Inspect the relevant source paths and use a spike where runtime behavior remains unresolved.
6. **Deferral clarified:** the deferred helper-version check would enforce the minimum PTAAPlanner version as a compatibility rule. The deferral does not defer or waive the supported-version requirement above. No automatic check is added now.
7. **Process documents corrected** (not reviewed by ChatGPT): the dependency lines in CLAUDE.md and the ledger no longer list ProjectTriuneMQ2AASpend as PTAAPlanner's dependency.
