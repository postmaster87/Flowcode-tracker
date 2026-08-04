# Flowcode-tracker

Mental-game tracker for golf range practice. Single-file web app, no build step,
published on GitHub Pages with Firebase cloud sync.

- **Production** — https://postmaster87.github.io/Flowcode-tracker/ (`index.html`)
- **Beta** — https://postmaster87.github.io/Flowcode-tracker/beta.html (`beta.html`,
  FocusCalm module, local-only, never syncs)

Launch from the phone's home-screen icon, never from a downloaded copy — a downloaded file
gets its own localStorage, which caused data loss once already.

## Start here

| If you are… | Read |
|---|---|
| Claude Code, picking up the build | [HANDOFF-CLAUDE-CODE.md](HANDOFF-CLAUDE-CODE.md) |
| A planning conversation getting up to speed | [HANDOFF-CLAUDE-CHAT.md](HANDOFF-CLAUDE-CHAT.md) |

## Reference

| File | What it is |
|---|---|
| [flowcode-mental-game-plan.md](flowcode-mental-game-plan.md) | The training system the app exists to serve. Source of truth for *why*. |
| [range-nine-spec.md](range-nine-spec.md) | Features 1–9 spec. Implemented. |
| [FOCUSCALM-NOTES.md](FOCUSCALM-NOTES.md) | Beta module notes, the standing storage question, draft Firestore rules (unpublished). |
| [TRACKER-BACKLOG.md](TRACKER-BACKLOG.md) | Backlog and the temporary dev scaffolding that must be removed. |
| [APP-STATE.md](APP-STATE.md) | Historical snapshot, 2026-08-02. Superseded by the chat handoff. |
| [PUBLISH-FLOWCODE.md](PUBLISH-FLOWCODE.md) | Publish checklist and auth troubleshooting log. |
| [flow-tracker-setup.md](flow-tracker-setup.md) | Original Firebase setup guide and security notes. |
