# FocusCalm module — beta build notes

Features 10–12 of the FocusCalm handoff, built 2026-08-02.

**This lives in `beta.html`, not `index.html`.** Production is untouched.

- Beta: `https://postmaster87.github.io/Flowcode-tracker/beta.html`
- Production: `https://postmaster87.github.io/Flowcode-tracker/` — unchanged

---

## Beta isolation — why it matters

`beta.html` shares an origin with the production app, which means it shares localStorage
and would share the same Firestore document. Two guards, both verified:

1. **Separate storage keys** — `flowtracker_beta_v1` (+ two backups). Beta never reads or
   writes the production keys. Confirmed: production keys byte-identical after beta writes.
2. **No Firebase at all.** The config, auth and sync code are removed from this build, not
   merely disabled — there is no code path that could reach `users/{uid}`. The chip reads
   "Beta — local only".

Consequence: **beta data does not sync and does not reach your phone.** Use beta on one
device at a time, and don't expect it to appear alongside real range data.

---

## ⚠️ STANDING QUESTION — storage architecture (unanswered)

§3 of the handoff specifies `focuscalmEntries` as a **Firestore collection with server-enforced
security rules**: update permitted only when the sole change is `flagged: false → true`, delete
permitted only when `flagged == true` and `createdAt` is before the current calendar day.

The app does not work that way. Everything currently lives in a single whole-state snapshot at
`users/{uid}`, which is what makes it work offline. The Range Nine spec said "no new backends".

**Matt deferred this decision.** For the beta build, entries live in the existing state object
as `state.focuscalm[]`, and the immutability rules are **enforced in app code only**:

- values are write-once; the form is the only writer
- `flagged` is one-way (`false → true`), never reversible
- deletion is refused unless `flagged === true` **and** `createdAt` is on an earlier calendar day

That is the same strength of guarantee the existing shot mis-entry flag has. It is **not**
tamper-proof — the restore-from-paste backup could defeat it, and the calendar-day rule trusts
the device clock. Server-enforced immutability requires the real collection.

**Acceptance criterion 11 (Firestore rules) is therefore deferred, not met.** When the decision
is made, the rules to publish would be roughly:

```
match /focuscalmEntries/{id} {
  allow create: if request.auth != null
                && request.resource.data.uid == request.auth.uid
                && request.resource.data.flagged == false;
  allow update: if request.auth != null
                && resource.data.uid == request.auth.uid
                && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['flagged'])
                && resource.data.flagged == false
                && request.resource.data.flagged == true;
  allow delete: if request.auth != null
                && resource.data.uid == request.auth.uid
                && resource.data.flagged == true
                && resource.data.createdAt < timestamp.date(
                     request.time.year(), request.time.month(), request.time.day());
}
```

Not published — do not paste this until the architecture decision is made.

---

## What is built

**Feature 10 — entry model at session setup.** Per-block | Blended joins the session plan,
defaults from Settings (`perblock`), copied onto the session and locked once running. Blended
enforces exactly one live entry per session; per-block requires a block tag and allows several.

**Feature 11 — post-session entry.** Reachable only when no session is active: a "+ FocusCalm
entry" button for standalone (`training`, no session link), and a "+ FC" button on each finished
session in the history. Form order matches the spec: type → block → start time → duration →
avg → high → low → time-in-calm % → note. Numeric keypads, all fields centre on focus.
Validation refuses anything outside 0–100 and anything where `low ≤ avg ≤ high` fails.
Save takes one confirm, then locks.

**Feature 12 — Review tab.** Hidden entirely while a session is active. Three hand-rolled inline
SVG charts (no libraries), each showing a placeholder with a progress count until 3 relevant
entries exist:

1. Baseline trend — `baseline` + `training` avg over time
2. Session comparison — band avg vs Mushin vs arousal. Mushin (1–5) and arousal (1–10) are
   rescaled to 0–100 and **arousal is inverted**, so "calmer" is up on every line and the two
   gaps the spec cares about read directly: band high + arousal-line low = miscalibrated
   self-perception; band high + Mushin low = a thought crept in on the backswing.
3. Circle Drill pressure — avg and low for `circle-drill` overlays.

A Mis-entries section lists flagged entries, excluded from every chart.

---

## Deviation worth knowing

§6 chart 3 asks for annotation with "drill outcome (completed 8 / restart count)". **Restart
counts do not exist in the data** — the Range Nine spec explicitly said "no restart counts" and
they were never collected. The circle size is also per-session now, so "8" isn't fixed. The chart
annotates with what does exist: date, effective drill time, and the circle's shape (e.g. `8×3ft`).
Collecting restart counts would mean reversing that earlier decision — say so if you want it.

---

## Acceptance criteria — verification status

Checked against §9 of the handoff on the **deployed** beta build, 2026-08-02.

| # | Criterion | Status |
|---|---|---|
| 1 | Toggle at session setup, defaults from Settings, locks on start | ✅ verified |
| 2 | Entry screen unreachable during an active session | ✅ verified — card, nav and form all blocked |
| 3 | Blended enforces one entry; per-block requires a block tag | ✅ verified both ways |
| 4 | Entries immutable; flag one-way; flagged drop out of charts | ✅ verified |
| 5 | Flagged entries deletable only on a later calendar day | ✅ verified — same-day delete refused |
| 6 | Standalone "+ FocusCalm" with no session link | ✅ verified — defaults to `training`, block field hidden |
| 7 | Demo shows no FocusCalm section, produces no entries | ✅ verified — data unchanged across a demo |
| 8 | All three Review charts render; placeholders under 3 entries | ✅ verified with seeded multi-date data |
| 9 | Review tab hidden during active sessions | ✅ verified |
| 10 | Fields centre in viewport on focus (Galaxy S26 / Chrome) | ❌ **not verified — needs a real device** |
| 11 | Firestore rules enforce §3 immutability | ⏸ **deferred** — see the standing question above |

**Chart values were checked numerically**, not just for presence: plotted y-coordinates were
back-computed and compared against hand-calculated expectations. All four series matched
exactly (band 50/56/62/68, Mushin 50/83/75/93, calm-from-arousal 48/74/67/86, drill avg and
low). The seeded data also produced the intended divergence signal — one session where band
rose while Mushin dipped, i.e. a late intrusion.

## Known limitation

**A session with zero shots gets no "+ FC" button**, so a FocusCalm entry can't be linked to a
completely empty session. This is deliberate-ish: blank sessions are the ones that can be
deleted outright, and a band-only check-in is better recorded as a standalone `baseline` or
`training` entry, which is exactly what the standalone path is for. Flagged here so it's a
known choice rather than a silent gap.

## Not verified

- **Viewport centering on the Galaxy S26 with the keyboard open.** The FocusCalm form is long;
  every field has the `scrollIntoView` treatment, but this shares the untested status of the
  original Feature 6 fix. Needs a real device.

## Untouched, as instructed

- `index.html` — production. Not modified in this work. Re-verified on the live deploy:
  Firebase and popup auth intact, no FocusCalm code present, no beta banner, no console errors.
