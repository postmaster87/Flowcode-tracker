# PUBLISH-FLOWCODE — FlowCode Range Tracker

**Task for Claude Code:** publish Matt's FlowCode Range Tracker on GitHub Pages and verify the Firebase wiring, following the same workflow used for `matts-strength-tracker`. GitHub username: `postmaster87`. Repo name: ask Matt (fill in: ____________). Companion reference: `flow-tracker-setup.md` if present in the repo.

**CRITICAL — do not copy from the strength tracker:** this app uses a **single-user, own-document Firebase pattern** (`users/{userId}`, each Google account touches only its own doc). The strength tracker uses a shared-document pattern with an email allowlist. The two are different projects with different rules — never mix their Firestore rules or Firebase configs. This app also has nothing to do with the strength tracker or any future golf rounds tracker; it is its own thing (a golf range/practice shot logger).

Work through in order, check items off, commit this file at the end.

Repo name: `Flowcode-tracker`. Live URL: `https://postmaster87.github.io/Flowcode-tracker/`.

## 1. Get the app file in place

- [x] Confirm the repo is cloned locally and you're in it
- [x] Matt supplies `flow-tracker.html` (it was built outside this repo — if it's not in the folder, STOP and ask him to drop it in before continuing)
- [x] Rename/commit it as `index.html` at repo root (exact lowercase) — done in commit 4a3cace
- [x] If `flow-tracker-setup.md` isn't in the repo and Matt has it handy, add it too (useful reference) — present
- [x] Push; confirm `git status` clean and nothing unpushed

## 2. Enable GitHub Pages

- [x] Already enabled prior to this session (`gh` CLI not installed locally; not needed)
- [x] Live URL returns HTTP 200
- [x] Fetched page contains expected FlowCode/tracker markup (sanity check it's the right file)

## 3. Firebase config (Matt clicks, you verify)

Matt does these in the browser at console.firebase.google.com — print this list for him:
- New project (e.g. `flowcode-tracker`) — MUST be separate from `matts-strength-tracker`
- Gear → Project settings → Add app → Web (`</>`) → copy firebaseConfig
- Security → Authentication → Sign-in method → Google → Enable (support email required) → Save
- Authentication → Settings → Authorized domains → add `postmaster87.github.io`
- Databases & Storage → Firestore → Create (production mode, us-central1) → Rules tab → paste the own-document rules from flow-tracker-setup.md (users/{userId}, auth-only, own-doc-only, deny-all fallback) → Publish

Then you:
- [x] Take the firebaseConfig Matt pastes into chat and insert it over the placeholder section in `index.html` (projectId `flowcode-tracker`, commit 21c5319)
- [x] `grep -c "PASTE_" index.html` returns 0 (only remaining match is the `startsWith("PASTE_")` guard logic)
- [x] Commit + push; redeploy confirmed — live page serves real projectId, zero placeholders

## 4. Verify + wrap up

- [ ] **Matt to do:** open the live URL on the S26 → Sign in with Google → chip shows name → log a test shot → check Firestore console for `users/<uid>` → incognito window without sign-in loads nothing from cloud
- [ ] **Matt to confirm in console:** Authentication → Google enabled + support email set; Authorized domains includes `postmaster87.github.io`; Firestore rules published as the own-document pattern from `flow-tracker-setup.md`
- [ ] Remind Matt: on the phone, Add to Home Screen from the live URL and ALWAYS launch from that icon (downloaded-file launches get separate localStorage — this caused data loss before)
- [x] Update checkboxes in this file, commit `publish flowcode tracker`, push

## Notes

- Firebase web config in the public repo is fine to be public; rules + auth are the boundary
- The repo is public (free Pages requirement) — Matt has accepted this
- If Matt's console layout differs from docs: Authentication is under **Security**, Firestore under **Databases & Storage** (newer sidebar)
