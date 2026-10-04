# PTAutoLeveler Project Ledger

Maintained per [Development_Protocol.txt](Development_Protocol.txt) Section 11. This ledger tracks the project's *current state* in four categories. Read this first to orient; consult [decision_log.md](decision_log.md) for the rationale, history, and active overrides behind any entry only as needed — don't reconstruct current state by reading the full decision log from scratch.

When new evidence resolves an open question, update this ledger before building on that conclusion. Do not silently rewrite prior entries — if something here turns out wrong, record the correction visibly (for example `[CORRECTED YYYY-MM-DD: ...]`) and, if the correction is material, add a decision log entry explaining why. Never write the hash of HEAD here (it is stale one commit later).

## Up next

**Cold start: where things stand at the end of 2026-10-04 (replace this block as the state changes; read it first, then the rest of this section only as needed).**

- **Repository:** `origin` is `https://github.com/thezerodivide/PTAutoLeveler` (public). Everything is committed and pushed, including DL-001, the rewritten `SPEC.md` (approved by the developer), the deletion of `TEMPLATE_README.md`, and `.claude/settings.json` (a project allow rule for `git push origin main`). The review folder `PTAutoLeveler-Review` (beside the checkout, no `.git`) was created with `tools/refresh_review.py` from the pushed commit, with no candidates; refresh it after the next push (the manifest names the commit it holds). A push to the wrong remote happened once on 2026-10-04 and was reverted on `New-Project-Template`; run `git remote -v` before every push.
- **Build:** nothing is built. No test harness yet.
- **Decisions approved and recorded:** DL-001 (retrofit of the pre-method spec; v1.0 solo only; acceptance criteria AC-1 to AC-18) and its two addenda dated 2026-10-04 (the second records the seven changes approved after ChatGPT's first review: Start boundary, same-DZ phase change, PTAAPlanner error pause, labels, index, and open rows). Next decision number: DL-002.
- **In flight, NOT approved and NOT recorded:** ChatGPT's artifact review of the exact files after the approved edits (a handoff has to be relayed by the developer). The PTAAPlanner discussion the developer asked for ("We'll discuss current PTAAPlanner functionality separately"): its README is out of date, so the PTAAPlanner facts drawn from it are unestablished, and the dependency lines in CLAUDE.md and this ledger (ProjectTriuneMQ2AASpend) are unchanged until that discussion. Developer-confirmed: ChatGPT does not review process setup.
- **Still open, in the order they bear on the work:** artifact review of the spec by ChatGPT; the PTAAPlanner functionality discussion; agree revisit triggers for the deferred items (DL-001 Open; the helper-version-check and duo triggers in the spec are marked proposed); the design stage (smallest design meeting AC-1 to AC-18, with a recommendation); TAC's role and the spec's stuck/camp-point check; the test harness and `test\check.cmd`, test-first; spikes for the game readings.
- **Process, in one place:** one decision at a time, each with the HANDOFF header, an `Answering:` line when it answers a review, verified / reasoned / unknown labels, the worst case and the developer's risk call, and a closing routing line; the reviewer's replies are information only and the developer approves only in their own words; documentation-only commits and pushes need no approval after the safety scan; code commits need the developer's explicit permission. All of this, with the reasons, is in `CLAUDE.md`.

## Dependencies

- **TAC** — combat and farming; not the developer's. Read its source before design (Protocol §4).
- **PTDeathRecovery, PTAAPlanner, PTItemEvolver** — the developer's own scripts; contracts not yet read for this project.
- **ProjectTriuneMQ2AASpend** — PTAAPlanner's dependency.
- **MQ2Nav** — navigation to the Priest.
- **MacroQuest (emu RoF2)** — https://github.com/macroquest/macroquest/releases/tag/rel-emu-rof2, docs https://docs.macroquest.org/. Nothing is built, so nothing is blocked yet.

## Pending Live Verification

Empty: nothing is implemented.

## Resolved behavior

Behavior actually agreed upon, by the developer, in DL-001. `SPEC.md` was rewritten to match it on 2026-10-04 and approved for commit by the developer; carried-over parts of the pre-method text that DL-001 did not address (the Locked decisions not marked corrected, the Phase model, Config validation) have not been re-approved item by item.

- v1.0 is solo only; duo is future functionality; other players come after a review pass.
- Respawning DZs only.
- A zone is eligible only if the Bazaar waypoint map has a waypoint inside it.
- Acceptance criteria AC-1 to AC-18 (DL-001) are the approved definition of v1.0 behavior.

## Confirmed live/system facts

Facts established through testing, source inspection, logs, or documentation. Each states how it was established.

Developer-observed in the game (2026-10-04 statements; no log or script output; one observer):
- The Priest of Triune creates the DZ (hail, `/say Respawning`); the DZ is entered through the Priest (`/say ready`) or the "Travel to Expedition" button on the Waypoint map in the Bazaar.
- Lockouts are per zone and per mode: Respawning 30 minutes, Non-Respawning 14 hours; several can exist at once; shown in the Expedition Information window.
- "Bazaar and Back" AA, `/alt activate 331`: transports to the Bazaar, and used while already in the Bazaar returns to the previous location; refresh time reads `0:02:00` (screenshot).
- The Priest is reachable from every waypoint landing point via `/nav`.

Read in the repo: PTAutoRoute's log and config layout and test strategy; PTDeathRecovery's README says it activates expedition travel from the Waypoint Map.

Not established: see Open implementation details.

## Open implementation details

Questions intentionally unresolved — do not decide these unilaterally; surface them for discussion when they become relevant. Each with its revisit trigger where deferred.

- How the script reads the DZ state, the AA XP % controls, TAC's readiness and the helper scripts' status (spikes, build steps 2–6 of the old spec).
- The lockout refusal message text; the Priest dialog sequence.
- Whether `/alt activate 331` works on cooldown.
- Re-entry mechanism for AC-7 (copy PTDeathRecovery's method, or trigger it with `/echo You died.`) — decide at the design stage.
- TAC's role and the spec's stuck/camp-point check — trigger: the TAC-role discussion.
- What Pause, Resume and Stop do to the character and TAC; what happens at plan completion.
- The split-phase stall rule — revisit trigger: once testing occurs.
- Whether plan creation needs a separate editor window.
- The state list (the spec's state machine diagram is missing).
- Tunables N, X, K, retry limit, per-state timeouts, PTDeathRecovery time limit — first values are implementation choices.
- Added 2026-10-04 (DL-001 second addendum): PTItemEvolver on/off setting scope (plan-wide or per phase); whether PTAAPlanner stops on an AA-phase to level-only-phase move; how the script detects PTAAPlanner errors and plan exhaustion (version and source revision to identify, spike); the meaning of "every state has a timeout" for FARM and a user pause.
- Deferred with revisit triggers still to be agreed: saving and loading named plans; helper version checks on start; PTAutoRoute second legs; all duo behavior; the self-explaining UI pass; open-world farming.

## Out of scope

Explicitly decided not to build or investigate.

- Duo (future functionality, not deleted from the spec).
- Non-Respawning DZs.
- Choosing, optimizing or generating routes, zones or PTAutoRoute paths; PTAutoRoute second legs in v1.0.
- Deciding AA purchases, combat, item evolution logic, death recovery.
- Hard-coding any example route.
