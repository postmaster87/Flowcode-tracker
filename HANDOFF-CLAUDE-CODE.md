# Handoff — Claude Code

Everything a fresh Claude Code session needs to pick this up. Read this first, then
`APP-STATE.md` for feature detail. Written 2026-08-02.

---

## 1. What this is

A single-file mental-game tracker Matt uses **one-handed, on a phone, at a driving range,
between shots**. That constraint drives most design decisions: big tap targets, minimal
typing, few taps per shot. A change that looks fine on desktop can be unusable in the field.

Two builds, both plain HTML files on GitHub Pages, no build step, no framework, no bundler:

| | File | URL | Cloud |
|---|---|---|---|
| **Production** | `index.html` | `https://postmaster87.github.io/Flowcode-tracker/` | Firebase sync ON |
| **Beta** | `beta.html` | `.../Flowcode-tracker/beta.html` | **No Firebase at all** |

Repo: `postmaster87/Flowcode-tracker`, public, branch `main`. Pages serves `main` at root.

---

## 2. Repo map

| File | What it is |
|---|---|
| `index.html` | **Production app.** Single file: CSS, HTML, ES module script, Firebase config. |
| `beta.html` | **Beta app.** Copy of production + FocusCalm module, Firebase stripped. |
| `flowcode-mental-game-plan.md` | Matt's mental game plan. **The source of truth for what the app is for.** Nodes 1–8, range session structure, Circle Drill. |
| `range-nine-spec.md` | Features 1–9 spec (Range Nine, reassignment, drill deduction…). Implemented. |
| `APP-STATE.md` | Feature-level current state. Superseded in part by `HANDOFF-CLAUDE-CHAT.md`. |
| `FOCUSCALM-NOTES.md` | Beta module notes + **the standing storage-architecture question** + draft Firestore rules (not published). |
| `TRACKER-BACKLOG.md` | Backlog + **the dev scaffolding that must be removed**. |
| `PUBLISH-FLOWCODE.md` | Publish checklist (done) + auth troubleshooting log. |
| `flow-tracker-setup.md` | Original Firebase setup guide + security notes. |

---

## 3. Architecture in one page

**No framework.** `$(id)` helper, direct DOM manipulation, one `render()` that redraws
everything from `state`. Every mutation goes through `persist()`, which writes state and
calls `render()`. Don't introduce a framework or a build step.

**State shape** (production):

```js
{
  settings: { pauseSec, shotPauseSec, shotsPerBlock, nineRoutine,
              circlePutts, circleFeet, defaultTee, theme },
  sessions: [{
    id, startedAt, endedAt,
    mode: "classic" | "rangenine",
    routineCount, pauseSec, shotPauseSec, circlePutts, circleFeet,  // the session PLAN
    course?, tee?, staged,
    shots: [{ t, stack, mushin, arousal, calmed, note, block,
               fairway, hole?, slot?, tee?, decision?, misEntry?, edited? }],
    drills: [{ t, durationSec, setupSec, putts, feet }]
  }],
  courseClubs: { [courseId]: { [holeNumber]: ["Driver","Wedge"] } },
  activeDrill: { startedAt } | null,
  updatedAt
}
```

Beta additionally has `settings.fcModel`, `session.fcModel`, and `state.focuscalm[]`.

**Persistence.** Whole-state JSON snapshot written to **three** localStorage keys
(`flowtracker_v1`, `_backup`, `_backup2`) for redundancy. `loadState()` tries each in turn
and runs `migrate()`.

**Sync (production only).** Same snapshot pushed to Firestore `users/{uid}` with
`{ data: JSON.stringify(state), updatedAt }`. Last-write-wins on `updatedAt`. Own-document
security pattern — each Google account can only touch its own doc. There is no per-entity
collection anywhere.

**Migration.** `migrate()` fills gaps in old data and must **never rewrite recorded values**.
Always tolerate missing fields (`arousal`, `tee`, `setupSec`, `putts` etc. may be absent on
older records) — a stat reader that assumes they exist will break history.

---

## 4. Hard invariants — do not break these

1. **Auth is popup-only.** `signInWithPopup`, with redirect only as a fallback for genuine
   popup blocks. **Never switch to `signInWithRedirect`** — the app is served from
   `github.io` while `authDomain` is `firebaseapp.com`, and Chrome's third-party storage
   partitioning silently breaks the redirect handoff. This was already diagnosed the hard
   way; the "obvious" mobile fix is the wrong one.
2. **Beta must never reach production data.** Separate localStorage keys *and* zero Firebase
   code in `beta.html`. Same origin means shared localStorage — this is the only thing
   standing between beta and Matt's real range data.
3. **Stack and Mushin are never editable.** Anywhere. Fixing *where* a shot lives is fine;
   revising *how it went* is not. This is the core of the honesty design.
4. **No hard delete during a live session.** Mis-entry flag only; flagged records stay
   visible and drop out of stats. Hard delete unlocks after the session ends.
5. **The session plan is fixed at start.** Counts and pauses are copied onto the session.
   Settings is defaults-only and must never affect a running session.
6. **Nothing about FocusCalm appears in session mode.** No live data, no entry path.
7. **Demo mode writes nothing**, ever — not to localStorage, not to the cloud.

---

## 5. Environment gotchas (this cost real time — read it)

- **`gh` CLI, `node`, and `python` are not installed.** No local server, no JS syntax check
  via node. Verify by loading the file in the browser instead.
- **Use PowerShell for git.** The Bash tool intermittently lacks `git`, `tail`, `curl`.
- **PowerShell here-strings break `git commit -m`** when the message contains quotes —
  arguments get split and you get `error: pathspec ... did not match`. **Write the message
  to a file and use `git commit -F <file>`.** Multiple `-m` flags also work for simple text.
- **The browser preview pane does not re-execute on reload.** `location.reload()` and
  re-`navigate` to a `file://` URL keep the previous in-memory `state`, so edits made after
  first load appear absent and cleared localStorage appears to come back. Workarounds:
  - Verify final edits against the **deployed https URL**, which does load fresh.
  - Or reset through the UI: end all sessions, then dev-delete them.
- **`computer{action:"screenshot"}` fails** when the pane isn't displayed. Use `read_page`,
  `get_page_text`, and `javascript_tool` instead — they work fine and are more precise.
- **`javascript_tool`: wrap everything in an IIFE.** Bare `const g = ...` persists between
  calls and throws "already declared" on the next one.
- **Foreground `sleep` is blocked in Bash.** For deploy polling use `run_in_background: true`
  with an `until curl -s <url> | grep -q "<marker>"; do sleep 5; done` loop.

---

## 6. Workflow that works

1. Edit the file.
2. Load `file:///C:/Temp/gitRepos/Flowcode-tracker/<file>.html` in the browser tool.
3. Drive it with `javascript_tool` — click real buttons via `.click()` rather than
   reimplementing logic, and read back DOM state as JSON. Module-scope functions are **not**
   on `window`, so you must go through the UI.
4. Check `read_console_messages({onlyErrors:true})`. **A clean load with the auth chip
   populated proves the whole module parsed and ran** — that's the cheapest smoke test.
5. Commit via PowerShell, push, then poll the deploy in the background (~15–60s).
6. Re-verify the last edits on the live URL, because of the preview-pane staleness above.

Useful cleanup snippet for test data:

```js
(()=>{const g=id=>document.getElementById(id); window.confirm=()=>true;
let n=0; while(g('activeSession').style.display!=='none' && n++<10) g('endBtn').click();
let d=0; while(g('sessionList').querySelector('[data-del]') && d++<40)
  g('sessionList').querySelector('[data-del]').click();
return 'cleaned';})()
```

---

## 7. Temporary scaffolding that must come out

Both were added at Matt's explicit request and are **not meant to ship**:

1. **Colour scheme picker** — Settings → "Colour scheme (dev)". Five palettes incl. two Iowa
   State options. When Matt picks one: hard-code it into `:root`, then delete `THEMES`,
   `applyTheme()`, the `#setTheme` field, and its three handler references.
2. **`DEV_DELETE = true`** — allows deleting sessions containing data. Set to `false` when
   the app settles; blank-session delete then remains.

Both are documented with removal steps in `TRACKER-BACKLOG.md`.

---

## 8. Open work

**Blocked on Matt:**
- Which palette to bake in.
- **Storage architecture for FocusCalm** — see the standing question in `FOCUSCALM-NOTES.md`.
  Real Firestore collection (server-enforced immutability, needs him to publish rules) vs the
  current in-snapshot approach. Acceptance criterion 11 is deferred until this is answered.
- Whether 3-shot par-5 scoring reads right (all-Y = birdie look, zero-Y = bogey, else par —
  my generalisation of his 2-shot rule, not something he specified).
- Whether the arousal ≥7 threshold for the calm-down question is right.

**Verification status:** every FocusCalm acceptance criterion except viewport-centering and
the deferred Firestore rules has been checked on the deployed build — see the table in
`FOCUSCALM-NOTES.md`. Chart output was verified numerically, not just for presence.

**Needs a real device — never verified:**
- **Viewport centering with the Android keyboard open.** Entry controls were moved above the
  session log and `scrollIntoView` added, but the acceptance test (20-shot list, every field
  reachable, keyboard open) has never been run. The beta FocusCalm form is the longest in the
  app and the best test of it.

**Known data gap:** Circle Drill restart counts were deliberately never collected, so the
beta's drill chart annotates with time and circle shape instead. Reversing that means
collecting them going forward.

---

## 9. Don't

- Don't touch `index.html` for beta/FocusCalm work — production is deliberately frozen on
  that module until Matt says otherwise.
- Don't publish Firestore rules. The draft in `FOCUSCALM-NOTES.md` is a draft.
- Don't add a build step, framework, or external runtime dependency. The single-file,
  no-dependency property is why this thing is reliable at the range.
- Don't add friction to the log-shot path. Per the spec, that's a bug of the same severity
  as a data bug.
