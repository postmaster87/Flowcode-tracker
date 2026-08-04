# Handoff — Claude Chat (planning)

Current state of the FlowCode Range Tracker, for a planning conversation picking this up.
Written 2026-08-02. **This supersedes `APP-STATE.md`**, which was written before the
FocusCalm module and is now partly historical.

You likely have: `flowcode-mental-game-plan.md`, `range-nine-spec.md`, and the FocusCalm
handoff (Features 10–12). This tells you **what actually got built**, what was decided along
the way, and what still needs a human answer.

---

## 1. Where things stand

Matt's tracker is **live, in real use, and syncing.** He plays it at the range on a Galaxy
S26 launched from the home screen.

- **Production** — `https://postmaster87.github.io/Flowcode-tracker/`
  Features 1–9 complete. Google sign-in, Firestore cloud sync verified cross-device.
- **Beta** — `https://postmaster87.github.io/Flowcode-tracker/beta.html`
  Production plus the FocusCalm module (Features 10–12). **Local-only, never syncs**, and
  deliberately isolated so it cannot touch real range data.

Everything specified so far has been built. Nothing is half-finished.

---

## 2. What the app does now

**Three session modes.** Classic (routine block → open-ended pressure block), Range Nine
(routine block → hole-by-hole nine on a real card), and Demo (scripted, writes nothing —
exists so showing people the app stops creating junk sessions).

**Two Veenker cards**, pulled from the course's own hole pages: front and back nine, Gold and
Blue yardages, men's par and HCP, strategy line per hole. Per-shot Gold/Blue tee declaration,
locked when the shot is logged.

**Per shot:** Stack Y/N · Mushin 1–5 · **arousal 1–10** · **calmed Y/N** (asked only at
arousal 7+) · 20-char note · target found · tee · hole/slot · par-5 call.

**Node 3 is in.** Arousal is the first field on the log card, framed as a pre-shot reading.
Two new stats: average arousal, and **% of high-arousal (7+) shots successfully calmed** —
the second is the one that shows regulation-on-command developing rather than just baseline
calmness.

**Par 5s are decision holes.** Tee shot, then a required Lay up / Go for it call. Laying up
always owes a third shot; going for it owes one only if the target is missed.

**The pause is enforced.** The Log button locks and counts down — no skip. Two tiers in Range
Nine: short reset between shots on a hole (default 30s), full pause between holes (default
120s). Routine block has no timer at all.

**The session plan is declared at start** — routine count, both pauses, Circle Drill shape
(putts and distance), FocusCalm entry model. All copied onto the session and locked. Settings
is defaults-only and cannot touch a running session.

**Honesty rules as built:** Stack and Mushin uneditable everywhere; no hard delete during a
live session, only a mis-entry flag that removes a shot from every stat while keeping it
visible; reassignment keeps the original timestamp and marks the record EDITED; Circle Drill
keeps raw time alongside the setup-adjusted time.

**FocusCalm (beta):** post-session manual transcription only, three review charts (baseline
trend, band vs Mushin vs arousal, Circle Drill pressure), one-way mis-entry flag, deletion
only on a later calendar day.

---

## 3. Decisions made in implementation that you should know about

These were judgment calls filling gaps the specs didn't cover. **All are reversible** — flag
any you disagree with.

1. **The calm-down question appears only at arousal 7+**, not every shot. Resolved by the
   spec's own "choose quieter" principle: below 7 there was nothing to regulate, so asking is
   pure friction on a screen used between shots.
2. **Three-shot par-5 scoring**: all Stack Y = birdie look, zero Y = bogey + circle, anything
   between = par. Generalises the two-shot rule the spec gave; the spec didn't cover a
   three-shot hole.
3. **A bogey adds to the circle debt but doesn't force the drill modal open** in Range Nine —
   a visible owed/paid counter and a "Start drill" button carry it instead. Classic still
   opens the drill immediately on a fairway miss, as before. Ending a session with debt
   outstanding asks for confirmation.
4. **Review charts rescale everything to 0–100 and invert arousal**, so "calmer is up" on all
   three lines and the two gaps you care about read directly off the chart: band high +
   arousal-line low = miscalibrated self-perception; band high + Mushin low = late intrusion.
5. **Planned clubs are stored per course+hole globally**, so they carry to the next time that
   card is played rather than being re-entered each session.
6. **Hole 9's name is unverified.** The course page gives no nickname; "Cross Country" appears
   in the page text but reads like guidance. Everything else on both nines is verified against
   the course's own pages, including two hand-checked anchors and the deliberately-hot Gold
   totals.

---

## 4. Open questions — these need Matt

1. **FocusCalm storage architecture.** §3 of the FocusCalm handoff specifies a real Firestore
   collection with server-enforced immutability rules. The app doesn't work that way — it uses
   a single whole-state snapshot, which is what makes it work offline, and the Range Nine spec
   said "no new backends". Matt deferred the decision. Beta currently enforces immutability
   **in app code only**, which is the same strength as the existing mis-entry flag but is *not*
   tamper-proof. Draft rules are written and deliberately unpublished. **Acceptance criterion
   11 is deferred, not met.**
2. **Colour palette.** Five schemes are live behind a temporary picker, including two Iowa
   State options. Once he chooses, it gets hard-coded and the picker deleted.
3. **Three-shot par-5 scoring** — does decision 2 above read right?
4. **Arousal 7+ threshold** — right, or should the calm-down question be every shot?
5. **Pause durations** — 30s / 2min, now that they're genuinely unskippable.

---

## 5. One deviation from the FocusCalm spec

§6 chart 3 asks to annotate the Circle Drill chart with "drill outcome (completed 8 / restart
count)". **Restart counts don't exist** — the Range Nine spec explicitly said "no restart
counts, no Mushin ratings, no putting stats" for drills, so they were never collected. The
circle size is also per-session now, so "8" isn't fixed either. The chart annotates with what
exists: date, effective time, circle shape. Collecting restart counts means reversing that
earlier decision — worth a deliberate call rather than a silent one, since it adds putting
detail the plan chose to keep out.

---

## 6. Constraints to respect when specifying more

- **Used one-handed, on a phone, outdoors, between shots.** Friction in the log-shot path is
  treated as a bug of equal severity to a data bug. Prefer quieter over richer.
- **Single HTML file, no framework, no build step, no external dependencies.** This is why
  it's reliable in the field.
- **Adjustments are made in advance, never mid-pressure.** Anything changeable during a
  session is the monkey negotiating.
- **Recorded values are never revised.** Mistakes are flagged and stay visible.
- **Beta and production are separate on purpose.** Anything touching FocusCalm goes to beta
  until Matt says otherwise.

---

## 7. Not verified

**Viewport centering with the Android keyboard open.** The fix is in — entry controls sit
above the session log, plus `scrollIntoView` and `interactive-widget=resizes-content` — but
the acceptance test (20-shot list, every field reachable with the keyboard up) has never been
run on a real phone. It's the one outstanding item that can't be checked from a desktop.

Also worth knowing: **beta data lives on one device and never syncs**, so its review charts
start empty and need 3 entries each before they render.
