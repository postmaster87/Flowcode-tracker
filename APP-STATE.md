# FlowCode Range Tracker — current state

Status summary for the Claude chat that authored the mental game plan and the Range Nine spec.
Written 2026-08-02. Reflects what is actually built and deployed, not what was planned.

**Live:** https://postmaster87.github.io/Flowcode-tracker/
**Repo:** `postmaster87/Flowcode-tracker` (public, GitHub Pages, single-file `index.html`)

---

## 1. Infrastructure — done and verified

- **Published** on GitHub Pages, installed to the S26 home screen.
- **Cloud sync working.** Firebase project `flowcode-tracker`, Google sign-in, Firestore
  own-document pattern (`users/{uid}`, auth-only, own-doc-only, deny-all fallback).
  Verified end to end: logged on laptop, signed in on the phone, phone pulled the laptop's data.
- **Sync model unchanged from the original design:** whole-state snapshot JSON, last-write-wins
  on `updatedAt`, plus three redundant localStorage keys. All new fields ride this path.
- **Auth gotcha worth remembering:** sign-in must use `signInWithPopup`. `signInWithRedirect`
  is broken here because the app is served from `github.io` while `authDomain` is
  `firebaseapp.com` — Chrome's third-party storage partitioning kills the redirect handoff.
  Popup works on both desktop and Android.

---

## 2. Three session modes

Mode is chosen at session start.

**Classic** — routine block, then an open-ended pressure block. Fairway hit/miss per pressure
shot; a miss opens the Circle Drill immediately.

**Range Nine** — routine block, then a hole-by-hole nine. Two built-in cards, both pulled from
the course's own hole pages (Gold + Blue, men's par/HCP): **Veenker Back Nine** and
**Veenker Front Nine**. Per-shot Gold/Blue tee toggle, declared before the shot and locked on log.

**Demo** — 1 routine shot then 3 pressure, full real UI, shortened pause, persistent DEMO banner.
**Writes nothing to storage or the cloud, ever** (verified: localStorage byte-identical before/after).
Exists so showing people the app stops creating real sessions.

---

## 3. The session plan is declared up front

This was a real bug fix, and it matters for the honesty principle. Originally the block counts
lived in Settings; changing them mid-session silently did nothing. Now **every session declares
its own plan at start, for both modes**, and the values are copied onto the session:

| Set at session start | Default |
|---|---|
| Routine shots before pressure | 10 classic / 7 nine |
| Pause between holes (classic: pressure pause) | 120s |
| Pause between shots on a hole (nine only) | 30s |
| Circle Drill — putts in the circle | 8 |
| Circle Drill — distance in feet | 3 |
| Course + default tee (nine only) | Veenker Back / Gold |

Settings is now **defaults-only** — it prefills the next session and cannot touch a running one.

---

## 4. The pause is now hard

Previously the countdown was **advisory only** — it displayed but never blocked logging. Now the
Log button is disabled and counts down until the pause has run in full. No skip, no early unlock,
survives backgrounding (timestamp-based).

- **Routine block: no timer.** Reps can be logged back to back.
- **Pressure block: enforced**, two-tier in Range Nine — short reset between shots on the same
  hole, full pause when stepping to a new hole. The prompt says which one is running.

---

## 5. Hole logic (Range Nine)

- **Par 3** = 1 shot. **Par 4** = 2 shots.
- **Par 5 = 2 or 3 shots, decided in play.** Tee shot, then a required **Lay up / Go for it** call
  on the second. Laying up always owes a third shot; going for it owes one only if the target is
  missed. Reach it and the hole closes in two.
- **Hole result, computed from Stack marks only:**
  - Par 3 — Stack Y → par or better; N → bogey, circle owed
  - Everything else — all Y → birdie look; zero Y → bogey, circle owed; anything between → par
- **Circle debt** (owed vs paid) is always visible during the pressure block, with a "Start drill"
  button when something is owed. Ending a session with debt outstanding asks for confirmation.
- Hole card shows number, name, par, yardage for the selected tee, HCP, planned clubs, the
  strategy line, and shot progress as dots plus "Shot 2 of 2–3".

---

## 6. What is captured per shot

`stack` (Y/N) · `mushin` (1–5) · `arousal` (1–10) · `calmed` (Y/N) · 20-char note ·
`block` · `fairway`/target (hit/miss) · `tee` · `hole` · `slot` · `decision` (layup/go) ·
`misEntry` · `edited`

**Node 3 (EMOTION) is in.** Arousal is rated 1–10 for every shot, positioned first in the log card
as a pre-shot reading. The "did you calm down before the stack?" question appears **only at
arousal 7+**, matching the plan's "above a 6, add one extra slow exhale" rule.

New stats surfaced: **average arousal**, and **% of high-arousal (7+) shots successfully calmed** —
that second number is the one that shows the skill developing rather than just the state.

---

## 7. Honesty rules, as implemented

- **Stack and Mushin are uneditable everywhere.** Fixing *where* a shot lives is allowed; revising
  *how it went* is not.
- **No delete during a live session.** The only option is **Flag as mis-entry**: the shot is removed
  from every stat, block and hole, but stays pinned in a Mis-entries section at the bottom of the log
  with its original values and timestamp. Reversible. Hard delete unlocks only after the session ends.
- **Reassignment** moves a shot between blocks, and to another hole/slot in Range Nine, keeping the
  original timestamp and marking it EDITED.
- **Circle Drill setup deduction** — raw duration is always kept; stats use effective time. Each
  record also stores the circle's shape (`putts`, `feet`), so a 6×4ft time is never silently
  averaged against an 8×3ft one.
- **Staging state** — "Consequence staged" / "Packed up", with a confirm if packing up while
  circles are owed.

---

## 8. Judgment calls I made that were not in the spec

Flagging these because they are mine, not Matt's, and any of them can be reversed:

1. **Calm-down question only at arousal 7+** rather than every shot. Resolved by the spec's own
   "choose quieter" principle — below 7 there was nothing to regulate, so asking is pure friction.
2. **Three-shot par 5 scoring** — all Stack Y = birdie look, zero = bogey, anything between = par.
   This generalises the spec's two-shot rule; the spec didn't cover a three-shot hole.
3. **A bogey adds to the debt but does not force the drill modal open** in Range Nine. The debt
   indicator and Start-drill button carry it. Classic still opens the drill immediately on a
   fairway miss, as before.
4. **Classic pressure counter reads "Shot 11"** instead of the old impossible "11 / 10". Classic is
   otherwise behaviourally unchanged.
5. **Planned clubs are stored globally per course+hole**, so they carry to the next time that card
   is played rather than being re-entered each session.
6. **Hole 9's name is "Cross Country" and is unconfirmed** — the course page gives no nickname;
   that phrase appears in the page text but reads more like guidance. Everything else on both
   nines is verified.

---

## 9. Known gaps

- **The mobile keyboard fix is structurally done but never verified on device.** Entry controls
  were moved above the session log so logging shouldn't require scrolling, plus `scrollIntoView`
  and `interactive-widget=resizes-content`. The acceptance test — a 20-shot list, every field
  reachable with the keyboard open — still needs a real phone.
- **Front nine ships with no planned clubs** (by design — the course pages don't publish them).
  Each hole shows an "add" button; entries persist per hole.
- **Course page yardages run slightly hot vs the rated card** (back nine Gold totals 3,026 vs 2,998).
  Per the spec the pages are the accepted source; not reconciled.

---

## 10. Temporary dev scaffolding — must be removed

Both were added at Matt's request and are explicitly not meant to ship:

1. **Colour scheme picker** in Settings — five palettes with live preview, including two Iowa State
   options (cardinal & gold, and gold-forward). Once Matt picks one it gets hard-coded and the
   picker deleted.
2. **`DEV_DELETE = true`** — allows deleting sessions that contain data, so tweak-run sessions
   don't taint real stats. Reverts to blank-sessions-only when the app settles.

Removal instructions live in `TRACKER-BACKLOG.md`.

---

## 11. Open questions for Matt

- Which colour palette to bake in.
- Does the three-shot par 5 scoring read right?
- Is the arousal 7+ threshold for the calm-down question correct, or should it be every shot?
- Do 30s / 2min feel right now that the pause is genuinely unskippable?
- Is hole 9 actually called "Cross Country"?
