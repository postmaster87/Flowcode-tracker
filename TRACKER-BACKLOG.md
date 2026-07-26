# FlowCode Tracker — Backlog

Edits queued after the 2026-07-25 range session. Newest intent at top.

**Status 2026-07-26:** both items below are now BUILT, along with the full
"Range Nine" spec (`range-nine-spec_1.md`). Kept here for the rationale.
Open question still outstanding: item 1's (a)/(b) choice was resolved as
**(b)** — see note at the end of that section.

---

## 1. Add Node 3 (EMOTION / arousal) tracking — Matt's request — ✅ BUILT

**Source:** `flowcode-mental-game-plan.md` §3, Node 3 — EMOTION *("energy in motion" — Fudoshin)*.

> **Drill:** rate your arousal 1–10 before each shot. Above a 6, add one extra slow exhale before you step in. You're learning to *feel* your state and adjust it on command.

**What Matt asked for, in his words:** know from 1–10 his arousal *before* each shot, and whether he was able to calm himself down *before running his stack*.

### Two new fields per shot

1. **Arousal (1–10)** — captured **before** the shot, i.e. before the stack is run. This is a pre-shot reading, not a retrospective one; the UI should make that ordering obvious (arousal selector sits *above* / earlier than the stack + Mushin controls).
2. **Calmed down before the stack? (yes/no)** — did he successfully down-regulate before running the routine.

### Key design decision to confirm with Matt

The plan's rule is **>6 triggers the extra exhale**. So "did you calm down" is only strictly meaningful when arousal is 7+. Options:

- **(a)** Always ask — simplest, but adds a tap to every shot including calm ones, which cuts against range usability
- **(b)** Only show the calm question when arousal ≥ 7 — matches the plan's rule exactly and keeps low-arousal shots to a single extra tap. There's precedent in the code: the Fairway selector already appears conditionally, only in the pressure block (`index.html`, `updateLogBtn`)

**Recommend (b).** Ask Matt before building.

**Resolved 2026-07-26 → (b).** The Range Nine spec's companion principle
("the app must be invisible at the range… when choosing between a richer
interface and a quieter one, choose quieter") settles it: the calm-down
question only appears at arousal 7+, which is exactly where the plan's
extra-exhale rule applies. Below 7 there was nothing to regulate, so asking
would be pure friction. Flag if you want it on every shot instead.

### Implementation notes

- Shot object is currently `{ t, stack, mushin, note, block, fairway }` (`index.html`, `logBtn.onclick`). Add `arousal` (number 1–10) and `calmed` (bool, or `null` when not asked).
- Gate the Log button the same way `fairway` is gated in `updateLogBtn()` — arousal always required, `calmed` required only when arousal ≥ 7 under option (b).
- Old shots won't have these fields. Every stat reader must tolerate `undefined` — don't let historical shots break the averages.
- **Stats worth surfacing:** average arousal per session; and the real Node 3 metric — **% of high-arousal (7+) shots he successfully calmed**. That second number is the one that shows the skill developing.
- Consider whether arousal correlates with Mushin in the session summary; that's the feedback loop the drill is training.
- Range-usability constraint: this is used one-handed between shots. A 1–10 selector must be big-target and single-tap — not a slider, not a dropdown.

---

## 2. Block logic doesn't match the plan — ✅ ADDRESSED

`flowcode-mental-game-plan.md` §"Range session" specifies **Block 1: Routine reps 10–12 balls**, then **Block 2: Pressure sim 6–8 balls**. The app doesn't implement that shape:

- **Pressure never ends.** `currentBlock()` is a binary threshold — once `shots.length` passes `shotsPerBlock`, *every* remaining shot is pressure. There's no finite pressure block and no "block complete" state.
- **Counter overflows.** In pressure the display is `shots.length - shotsPerBlock + 1` with no cap, so shot 21 with a threshold of 10 renders **`11 / 10`**, then `12 / 10`. Likely to show up in any ~25–30 ball session.
- **One setting drives both blocks**, so the plan's asymmetry (10–12 then 6–8) can't be expressed.

**Likely fix:** separate `routineShots` / `pressureShots` settings and a real end-of-block state. Confirm with Matt whether endless pressure actually bothered him in practice before building — it may be fine to just stop when done.

**Done 2026-07-26.** Range Nine solves this properly: the pressure block is
a nine-hole card with a real end state, so it can't run forever. Classic mode
was left behaviourally unchanged per the spec, with one exception — the
counter no longer renders an impossible denominator (`11 / 10`); classic
pressure now reads `Shot 11`, since that block genuinely has no fixed length.
