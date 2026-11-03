# RENOBOOK Security Fix Brief (for Gemini Argon)

**Scope:** `/workspace` — static site `index.html` (CNAME: `book.renocamp.io`). Invoice/finance app (“香港裝修師傅大本營 / RENOBOOK PRO”).

**Constraint:** Produce a hardened production build. Do **not** leave Gemini/Bard canvas runtime in the deployed page.

---

## Goals

1. Remove leaked/injected secrets and canvas tooling from production HTML.
2. Replace client-only “security” with real access control (Supabase Auth + proper RLS).
3. Stop treating browser-visible strings as secrets (PIN digests, custom headers, PATs).
4. Reduce XSS / supply-chain risk (escaping, SRI/CSP).

---

## Critical issues to fix

### 1. Gemini/Bard canvas scripts shipped to production

- File starts with many `<script data-bard-client-injected="true">…`.
- Includes Firebase config for `bard-frontend` and `__initial_auth_token` (JWT).
- WebSocket/fetch/`postMessage` bridges meant for Gemini iframe, not a public site.
- `buildCleanDeployableHtml()` does **not** strip these; redeploys can keep re-publishing them.

**Fix**

- Delete **all** `data-bard-client-injected` scripts and related globals (`__firebase_config`, `__initial_auth_token`, `__app_id`, `requestNewFirebaseToken`, etc.).
- Ensure deploy path writes a clean app-only HTML (or better: build from source, not DOM clone of the live page).
- Never commit canvas exports as production.

### 2. Default PIN is trivial and client-only

- `DEFAULT_PIN_DIGEST = 'ecdeco_v2_f8a6ef71_de29224b'` → PIN **`1668`**.
- Unlock flag: `sessionStorage.INV_PIN_UNLOCKED` (UI lock only).
- Lockout: 5 attempts / 60 seconds — easy to brute force.
- Hash salts are hardcoded in JS (`HK_MASTER_CAMP_SALT_2026`, etc.).

**Fix**

- Remove reliance on a shared default PIN for data protection.
- Require real login (Supabase Auth) before loading financial data.
- If a PIN overlay remains, treat it as optional UX only; change default; use proper `crypto.subtle` hashing; do not store plaintext `INV_APP_PIN`.

### 3. Hardcoded RLS “secret” in frontend

- `const ECDECO_RLS_ACCESS_TOKEN = 'RENOBOOK_HK_2026_SECURE'`
- Sent on every Supabase client as `x-ecdeco-access-token`.
- Suggested RLS SQL only checks that header equals the same public string.

**Fix**

- **Remove** shared header gate entirely.
- Use Supabase Auth JWT + RLS on `auth.uid()` / roles / org membership.
- Rotate any policies that depended on `RENOBOOK_HK_2026_SECURE`.

### 4. Supabase anon URL/key in page + weak table protection

- `DEFAULT_SUPABASE_URL` / `DEFAULT_SUPABASE_KEY` (anon role) embedded (split-string obfuscation only).
- Tables used include: `projects`, `expenses`, `incomes`, `labor_days`, `invoices`, `companies`, `clients`, `catalog_items`, `categories`, `workers`, `v_project_pnl`.
- No `supabase.auth` / `signIn` usage found.

**Fix**

- Keep anon key only if RLS is strict per authenticated user (anon key is public by design).
- Enable Auth; deny all unauthenticated access by default.
- Audit every table/view; add `SELECT/INSERT/UPDATE/DELETE` policies; revoke public grants as needed.
- Rotate the anon key after assuming prior exposure.

---

## High-priority issues

### 5. GitHub PAT stored / syncable from the browser

- PAT in `localStorage` (`RENOBOOK_GH_PAT_SAVED`, `RENOBOOK_GITHUB_CFG`).
- Encrypted copy can sync via `catalog_items` (`__GH_CFG__:`), AES key derived from PIN digest (default PIN known ⇒ recoverable).
- App can push `index.html` to GitHub via API.

**Fix**

- Remove in-browser PAT storage and cloud sync of PATs.
- Revoke any PAT ever entered in this UI.
- Prefer GitHub Actions / backend deploy with fine-scoped credentials — not a user PAT in localStorage.

### 6. CDN scripts without SRI; no CSP

Scripts include Tailwind CDN, Supabase JS, Chart.js, html2canvas, jsPDF, SheetJS — no `integrity=`, no CSP.

**Fix**

- Prefer a bundled build (Vite/etc.) or pin CDN URLs with Subresource Integrity + `crossorigin`.
- Add CSP (and other security headers via hosting where possible).

---

## Medium issues

### 7. XSS via `innerHTML`

- `escapeHtml()` exists and is used often — keep it.
- Still many `` el.innerHTML = `...${...}` `` paths (status, deploy errors, labels).

**Fix**

- Prefer `textContent` / DOM APIs.
- Always `escapeHtml()` (or sanitize) for any user/API/error string inserted as HTML.

### 8. `postMessage(..., '*')` without origin checks

- Mostly from Bard injects; remove with those scripts.
- If any app `message` listeners remain, verify `event.origin`.

---

## Suggested implementation order

1. **Sanitize HTML:** strip all Bard/Firebase injects; verify clean deploy artifact.
2. **Rotate credentials:** Supabase anon key; revoke GitHub PATs; invalidate old RLS header policy.
3. **Auth + RLS:** Supabase email/OAuth (or magic link); policies per user/org; lock down tables listed above.
4. **Remove false security:** shared `x-ecdeco-access-token`, default PIN as data gate, PAT-in-localStorage deploy.
5. **Harden client:** SRI/CSP, reduce `innerHTML`, stop committing secrets/digests that encode known PINs.
6. **Retest:** unauthenticated REST access to all tables must fail; authenticated access scoped correctly; PIN overlay optional only.

---

## Acceptance criteria

- [ ] Production `index.html` has **zero** `data-bard-client-injected` / `__initial_auth_token` / Bard Firebase config.
- [ ] Unauthenticated callers cannot read/write finance tables with the public anon key.
- [ ] No shared static header secret required for API access.
- [ ] No GitHub PAT in `localStorage` or Supabase rows.
- [ ] Default PIN `1668` is gone / irrelevant to data access.
- [ ] Third-party scripts pinned (SRI) or self-hosted; basic CSP present.
- [ ] User-controlled strings are not assigned raw into `innerHTML`.

---

## Out of scope / notes for the fixer

- Repo today is essentially one large `index.html` (~16k lines) — prefer splitting/building if feasible, but security fixes above matter first.
- Obfuscating keys by string-split (`.join('')`) is **not** security; do not “fix” by more obfuscation.
- Do not reintroduce canvas export tooling into the public deploy path.
