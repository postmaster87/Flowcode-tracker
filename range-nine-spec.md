# Flow Tracker — "Range Nine" Game Mode Spec

Handoff doc for Claude Code. Target: `index.html` (single-file app, GitHub Pages, Firebase sync). Extend, don't rewrite — all existing storage, sync, and snapshot logic stays as-is. New fields must serialize through the existing snapshot JSON and survive the timestamp-based conflict resolution.

## Governing principle — honesty by design

This is a standing requirement for every feature in this spec and every future revision. **Adjustments are made deliberately in advance; never in the moment under pressure.** Settings changed between sessions are plans; anything changeable mid-pressure is the monkey negotiating, and the app must make that structurally impossible. Corollaries: recorded data is never revised (Stack/Mushin values are uneditable everywhere), mistakes are moved and kept visible rather than deleted, and raw values are preserved even when adjusted values are displayed. When a design decision is ambiguous, resolve it in favor of this principle.

**Companion principle — the app must be invisible at the range.** Every interaction competes with presence. Logging a shot should take seconds and zero navigation; anything that makes Matt fight the screen — scroll hunts, buried fields, extra taps — is mental noise injected between shots, which is precisely what the training exists to remove. When choosing between a richer interface and a quieter one, choose quieter. Friction anywhere in the log-shot path is a bug of the same severity as a data bug.

## Why (context from the 2026-07-25 session)

First live run of a "play nine at the range" pressure game exposed the gaps:

- No concept of a hole. Shots logged as a flat list; one shot got logged in the wrong block and reconstructing the card afterward took a manual walkthrough.
- No way to reassign a stray log to the right hole/block after the fact.
- Circle Drill timer counted ball-staging setup time (~1–2 min) as drill time, corrupting the drill stats.

## Feature 1 — Game modes

Add a `mode` concept to sessions. Two modes at launch:

- **Classic** (default): current behavior, unchanged. Routine block → Pressure block, shotsPerBlock from settings.
- **Range Nine**: routine block (configurable count, default 7) → hole-by-hole pressure block.

Mode is chosen at session start. Session records `mode` and, for Range Nine, a `course` object.

### Range Nine flow

- Ships with two built-in cards: **Veenker back nine** (data below) and **Veenker front nine** (to be pulled — see "Course data" below). Card structure is data-driven so more nines can be added later without code changes.
- **Tee selection:** every hole carries both **Gold and Blue** yardages. Session start asks for a default tee; every shot has a **one-tap Gold/Blue toggle** on the hole card, because tournament setups mix tees hole-by-hole and the Blues at Veenker are a different golf course on several holes. The shot record stores `tee` so the debrief can split results by tee. The toggle is a pre-shot declaration — it locks when the shot is logged.
- Pressure block shows a **hole card**, not a shot counter: hole number, name, par, yardage for the selected tee, planned clubs, and the strategy line ("the line"). This replaces the block banner's denominator, which was meaningless in the manual run.
- Par 4/5 = 2 logged shots (tee, then approach/second). Par 3 = 1 shot. App advances to the next hole automatically when the hole's shots are logged.
- Per shot, existing fields (Stack Y/N, Mushin 1–5, note) plus the fairway toggle relabeled **"Target found"** in this mode (applies to every shot, not just drives).
- **Hole result, computed:** both shots Stack Y → birdie look; one Y → par; zero Y → **bogey, Circle Drill owed**. Par 3: Y → par-or-better, N → bogey.
- Running debt indicator: circles owed vs. circles paid, always visible during the pressure block.
- End-of-session summary adds the nine-line card (hole, Stack marks, result) alongside the existing Stack % and Mushin average.

### Course data — Claude Code task

Pull the front nine (and Blue yardages for the back nine) from the course's own hole pages: `https://veenkergolf.com/hole-1` through `hole-9` (back nine at `hole-10` … `hole-18`). Each page has the hole name as an H2, a strategy paragraph (use it as the `line`, trimmed to one or two sentences), and a yardage table with Blue/Gold/White/Red rows plus Men's Par and HCP. Use **Gold and Blue, men's par/HCP** only.

Validation anchors already confirmed by hand: Hole 1 "The Meadow" — Blue 435 / Gold 419, par 4, HCP 7. Hole 2 "Plum Valley" — Blue 334 / Gold 283, par 4, HCP 9 (where a page lists two numbers per tee like 283/260, take the first). Hole 5 is flagged by the course among the Des Moines Register's 18 toughest. If a page can't be fetched, stub the hole and flag it rather than inventing numbers. Note: the per-hole pages' gold totals run slightly hot vs. the rated card (3,026 vs 2,998 on the back) — the pages are the accepted source; don't reconcile.

Suggested per-hole shape: `{"n":1,"name":"The Meadow","par":4,"hcp":7,"yds":{"gold":419,"blue":435},"shots":[...],"line":"..."}`. Planned clubs for the front nine: leave `shots` empty and let Matt fill them in-app the first time he plays the card (club choices are his, and Blue vs Gold changes them).

### Veenker back nine (Gold) — seed data

Gold yardages verified by hand below. **Claude Code: add `blue` yardages and `hcp` to each hole from the same course pages** (hole-10 … hole-18) and convert `yds` to the `{gold, blue}` shape.

```json
{
  "courseId": "veenker-back-gold",
  "name": "Veenker Back Nine — Gold",
  "par": 36,
  "holes": [
    {"n":10,"name":"Punch Bowl","par":5,"yds":473,"shots":["Driver","Long iron"],"line":"Drive left of center. Reachable — make the go/lay-up call behind the ball."},
    {"n":11,"name":"Davey Jones","par":3,"yds":134,"shots":["Short iron"],"line":"All carry, usually into wind. Plenty of club."},
    {"n":12,"name":"Cotton Woods","par":4,"yds":306,"shots":["Metal/hybrid","Wedge"],"line":"Aim last two trees left; everything kicks right. Right is jail."},
    {"n":13,"name":"Oak Ridge","par":3,"yds":144,"shots":["Short iron"],"line":"Must find the dance floor. Everything around it kicks."},
    {"n":14,"name":"Walnut Drive","par":4,"yds":397,"shots":["Driver","Short iron"],"line":"Aim the Y tree on the left of the dogleg."},
    {"n":15,"name":"Over There","par":4,"yds":386,"shots":["Metal/hybrid","Mid iron"],"line":"~220 around the dogleg, left of the overhanging tree. Plenty of club in."},
    {"n":16,"name":"Squaw Creek","par":5,"yds":485,"shots":["Driver","Long iron/metal"],"line":"HCP 2. Left side, accuracy over length. Go/lay-up decided behind the ball."},
    {"n":17,"name":"Little Boy","par":3,"yds":180,"shots":["Mid iron"],"line":"42-yard green, three clubs of difference. Pick a pin and commit."},
    {"n":18,"name":"Glory Hole","par":5,"yds":521,"shots":["Driver","Metal"],"line":"Down the middle. Don't be short."}
  ]
}
```

## Feature 2 — Reassign / fix a logged shot

From the session log (during or after a session): edit any shot's **block** and, in Range Nine mode, its **hole and shot slot**. This is the "assign to hole" fix — a mistap should be a ten-second correction, not a post-round forensic exercise.

**No delete button.** Instead: **"Flag as mis-entry."** A flagged shot is removed from all blocks, holes, and stats, and moves to a **Mis-entries** section pinned at the bottom of the session log, showing its original values and timestamp. Mis-entries are visible for the rest of the session — they have to be seen, not vanished. Deletion is only enabled **after the session ends**; during a live session the flag is the only option (and it's reversible — unflag restores the shot to wherever it was). Rationale: an accidental log is real data about a real moment, and making it disappear mid-session invites laundering a bad shot as a "mistake." Flagging keeps the correction honest by keeping it in view.

Guardrails: flagging and reassigning keep the original timestamp; a small "edited" marker on reassigned entries; **no editing of Stack/Mushin values anywhere** — the honesty of the two marks is the whole system. Fixing *where* a shot lives is fine; revising *how it went* is not.

## Feature 3 — Circle Drill setup-time adjustment

- After a drill is stopped, show an optional **"setup time to deduct"** field (seconds or m:ss). Stored as `setupSec` on the drill record; effective drill time = `durationSec - setupSec`.
- All drill stats (fastest, average, recent list) use effective time. Keep raw `durationSec` in the record for honesty.
- Nothing else gets added to drills. No restart counts, no Mushin ratings, no putting stats — the drill's existing time-only tracking is intentional and stays that way.

## Feature 4 — Staging state (small, optional)

A lightweight toggle on the pressure screen: **"Consequence staged"** (putter + balls on the green) / **"Packed up."** Packing up while circles are still owed prompts a confirm ("Debt outstanding — pack up anyway?"). This encodes the declaration ritual: staging keeps the debt visible; packing up is a commitment that the last bogey is behind you.

## Feature 5 — Presence timer scoping

The between-shot presence timer changes from global to block-scoped:

- **Routine block: timer off.** Shots can be logged back-to-back. Routine reps are about grooving the sequence; pacing there is self-managed.
- **Pressure block: 120 seconds by default.** No skip button, no early unlock, survives app backgrounding (existing timestamp-based implementation carries over). In Range Nine mode this runs between holes and between shots within a hole.
- **`pauseSec` stays in settings as a master override for the pressure timer.** Relabel it to make its new role clear (e.g. "Pressure pause — default 2:00"). Changing it is a deliberate act in settings before or between sessions, made when time-constrained — not an in-the-moment escape. It must not be adjustable from the timer screen itself, and there is still no skip: whatever the value is set to runs in full.

## Feature 6 — Bug fix: input fields scroll out of view (mobile)

On the Galaxy S26 (Chrome/Android), once the session's shot list grows, the entry fields (note box, etc.) sit below the visible viewport — every log requires a manual scroll down first. Fix: when any entry field receives focus, scroll it into the **visible** viewport, centered in the space that remains after the keyboard opens — not the full-page viewport. Use `scrollIntoView({block:"center"})` on focus plus the VisualViewport API (or `interactive-widget=resizes-content` in the viewport meta) so the on-screen keyboard is accounted for; test with the keyboard open, since Android Chrome overlays it rather than resizing by default. Better still, keep the active entry controls in a fixed/sticky panel so the shot list scrolls under them and logging never requires scrolling at all — implementer's call, but the acceptance test is: with a 20-shot list, tap to log a new shot and every field is visible and tappable without any manual scroll, keyboard open or closed.

## Feature 7 — Mantra update + creed sizing

The mantra is now, everywhere it appears: **"The Clouds Cannot Change The Sky."** Replace all instances of the old wording ("the cloud doesn't change the sky"). Bump the creed and mantra a few text sizes so they fill their space and read as the anchor of the screen, not a footnote — the mantra should be the largest text on whatever screen it closes.

## Feature 8 — Delete blank sessions

Demoing the app to people creates real sessions (four logged from yesterday alone). Add delete for **blank sessions only**: a session is blank iff it has zero shots and zero drills. Blank sessions get a delete control (with confirm) in the session list; any session containing even one shot or drill never shows it — those follow the mis-entry rules. This doesn't bend the honesty principle: a blank session contains no record of anything, so there's nothing to launder.

## Feature 9 — Demo mode

For showing the app to people without polluting real data. A **Demo** entry point (session-start screen, clearly labeled) that runs a scripted mini-session: routine block for **1 shot**, then pressure block for **3 shots** — full real UI each step: Stack Y/N, Mushin, note, target toggle, and the pressure pause (shortened to a few seconds in demo so the walkthrough doesn't stall). Persistent "DEMO" banner while active. **Nothing is written to storage or synced — ever.** Exits back to the normal start screen; on exit, nothing remains. Demo prevents both problems at once: no more blank sessions, and no more explaining the app while the honesty rules fight you.

## Non-goals

- No changes to colors or the 20-char autocorrect-off note field. The creed's wording changes only per Feature 7. The pressure-block timer keeps its no-skip, background-surviving behavior (see Feature 5 for the new scoping).
- No editing of Stack/Mushin values anywhere. No hard delete during a live session — mis-entry flag only.
- No putting stats beyond drill time and the setup deduction. Putting is trained separately and stays out of the tracker.
- No new backends. Everything rides the existing snapshot/Firestore path.

## Migration

Old sessions have no `mode`: treat as `"classic"`. Old drills have no `setupSec`: treat as 0. Old shots have no mis-entry flag or `tee`: treat as not flagged / tee unknown (display "—", exclude from tee-split stats). Snapshot version bump if the app versions its schema.
