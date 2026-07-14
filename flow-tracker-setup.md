# FlowCode Range Tracker — Setup Guide

Two parts: publish the app, then wire up cloud sync. The app works fully offline without Part 2 — cloud sync just stays off.

---

## Part 1 — Publish on GitHub Pages

Same workflow as the Austin album.

1. Create a new repo (e.g. `flow-tracker`). **Recommendation: keep it private is not an option for free Pages — it will be public.** That's fine (see Security notes), but rename the file to something non-obvious if you'd rather not have it easily found.
2. Upload `flow-tracker.html` and rename it to `index.html`.
3. Settings → Pages → Deploy from branch → `main` / root → Save.
4. Your app lives at `https://<username>.github.io/flow-tracker/`.
5. On the S26: open it in Chrome → menu (⋮) → **Add to Home screen**. It opens app-like, and localStorage persists correctly from this entry point (this is what fixed the Downloads data-loss bug — always launch from the home screen icon, never from a downloaded file).

---

## Part 2 — Firebase cloud sync

### 1. Create the project
1. [console.firebase.google.com](https://console.firebase.google.com) → Add project → name it (e.g. `flowcode-tracker`). Disable Analytics — not needed.
2. In the project: click the **</>** (web) icon → register app → copy the `firebaseConfig` block.
3. Paste those values into the marked section at the top of `index.html`'s script (replacing every `PASTE_...` placeholder). Commit the change.

### 2. Enable Google Sign-In
1. Build → **Authentication** → Get started → Sign-in method → **Google** → Enable → set support email → Save.
2. Authentication → Settings → **Authorized domains** → Add domain → `<username>.github.io`. Without this, the Google popup will be blocked on the live site.

### 3. Create Firestore + lock it down
1. Build → **Firestore Database** → Create database → production mode → nearest region (us-central1).
2. Rules tab → replace everything with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null
                         && request.auth.uid == userId;
    }
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

3. Publish. This is the actual security boundary: only a signed-in user can touch a document, and only their own. Everything else is denied by default.

### 4. Verify
Open the live site → tap **Sign in** → complete Google sign-in → chip shows your name → log a test shot → check Firestore console for `users/<your-uid>`. Then open the site in an incognito window without signing in and confirm nothing loads from the cloud.

---

## Security notes

**The API key in your public repo is not a secret.** Firebase web config is designed to be public — it identifies the project, it doesn't authorize anything. Authorization comes entirely from the Firestore rules above plus Authentication. Anyone who copies your config can only ever read/write *their own* document under *their own* Google account.

**Hardening checklist:**
- [ ] Firestore rules published exactly as above (auth-only, own-doc-only, deny-all fallback)
- [ ] Authorized domains list contains **only** `<username>.github.io` and `localhost` — remove anything else
- [ ] Authentication → Settings → User actions → consider disabling "Create" via unused providers (only Google should be enabled)
- [ ] Firebase console → Project settings → confirm no other apps registered
- [ ] Optional: Firestore → Usage → set a budget alert so a surprise traffic spike emails you

**MFA — you already have it (probably):**
Because sign-in is *Google* sign-in, your account security **is** your Google account security. If your Google account has 2-Step Verification on, every sign-in to this app already goes through it — nothing to build, zero code. Check at [myaccount.google.com/security](https://myaccount.google.com/security). That's the single highest-value security step here.

**App-level password:** technically easy (Firebase email/password provider is a checkbox), but it's a downgrade — a password you invent is weaker than Google + 2SV, and it adds a second credential to lose. Skip it.

**Firebase-enforced MFA:** exists (SMS/TOTP challenge on top of the provider), but requires upgrading the project to Identity Platform and is aimed at multi-user apps handling sensitive data. Overkill for a single-user range log — Google 2SV gives you the same protection for free.

**What's actually at risk:** golf shot logs. Worst realistic case if everything were misconfigured is someone reading Mushin scores. The rules above close even that.
