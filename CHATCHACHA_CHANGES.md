# Chat Chacha — Customization Log

Every change made to the original wacrm codebase. When pulling upstream
updates, these are the files to protect. See UPSTREAM_SYNC.md for the process.

## Branding

| File | What changed | Date |
|------|-------------|------|
| `messages/en.json` | `Sidebar.title` set to "Chat Chacha" | 2026-07 |
| `src/app/layout.tsx` | Page title, description, and favicon path | 2026-07 |
| `src/components/layout/sidebar.tsx` | Brand mark replaced with `/logo.png` | 2026-08-13 |
| `src/app/(auth)/login/page.tsx` | Brand mark replaced with `/logo.png` | 2026-08-13 |
| `src/app/(auth)/signup/page.tsx` | Brand mark replaced with `/logo.png` | 2026-08-13 |
| `src/app/icon.tsx` | **Deleted** — generated purple favicon | 2026-08-13 |

All brand-mark edits are commented `CHATCHACHA CUSTOM` in the source.

## Files added (not in upstream — cannot conflict)

| File | Purpose |
|------|---------|
| `public/logo.png` | Chat Chacha logo, used in sidebar and auth pages |
| `src/app/icon.png` | Static favicon, replaces the generated one |
| `.github/workflows/deploy.yml` | Auto-deploy to DigitalOcean on push to main |
| `CHATCHACHA_CHANGES.md` | This file |
| `UPSTREAM_SYNC.md` | Upstream merge runbook |

## Recurring fixes

**`uuid_generate_v4()` in migrations** — this Supabase project does not have the
`uuid-ossp` function available. Any new migration using it must be changed to
`gen_random_uuid()` before `supabase db push`:

```powershell
Get-ChildItem supabase\migrations\*.sql | ForEach-Object { (Get-Content $_.FullName) -replace 'uuid_generate_v4\(\)', 'gen_random_uuid()' | Set-Content $_.FullName }
```

**`@swc/helpers` missing from lockfile** — merges sometimes drop this entry,
which breaks `npm ci` in CI while local builds still pass. Fix:

```powershell
npm install @swc/helpers@0.5.23
```

## Bug fixes

| File | What changed | Date |
|------|-------------|------|
| `src/app/auth/confirm/route.ts` | **Added** — verifies Supabase's password-reset `token_hash` via `verifyOtp()` and redirects to `next`. The reset-password email link 404'd before this existed. | 2026-08-27 |
| `src/app/(auth)/reset-password/page.tsx` | **Added** — the "set a new password" form the reset link lands on. | 2026-08-27 |
| `src/app/(auth)/forgot-password/page.tsx` | `resetPasswordForEmail`'s `redirectTo` now points at `/reset-password` directly (previously pointed at a non-existent `/auth/callback` using the wrong PKCE `code` flow instead of the `token_hash` flow Supabase's recovery emails actually use). | 2026-08-27 |

**External config this fix also depends on (not tracked in git):**
- Supabase Dashboard → Authentication → Emails → **Reset Password** template — link changed from `{{ .ConfirmationURL }}` to `{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=recovery&next={{ .RedirectTo }}`. Editing this template requires Custom SMTP to be enabled first (Supabase locks template editing on the default mailer).
- Custom SMTP enabled via **Resend**, domain `chatchacha.in` verified. Needed both to unlock the template edit above and to get past Supabase's very tight default-mailer rate limit.

If either of these ever get reset (e.g. a new Supabase project, a dashboard misconfiguration), the reset-password flow will 404 or bounce to `/login` again even though the code above is untouched.

## Rebrand

| File | What changed | Date |
|------|-------------|------|
| `src/lib/api-keys/keys.ts` | `API_KEY_PREFIX` changed from `wacrm_live_` to `chatchacha_live_` (plus its explanatory comment). | 2026-08-28 |
| `src/lib/auth/api-context.ts` | Comment referencing the prefix updated to match. | 2026-08-28 |
| `src/lib/webhooks/endpoints.ts` | Comment referencing the prefix updated to match. | 2026-08-28 |

No client had generated an API key yet at the time of this change, so nothing existing was invalidated. `docs/public-api.md`, `docs/mcp.md`, and the `mcp-server/` docs still show `wacrm_live_` in their examples — intentionally left as-is for now, cosmetic only.

## Reverted / not currently applied

**Invite-only signup** — a guard in `src/app/(auth)/signup/page.tsx` reading
`NEXT_PUBLIC_INVITE_ONLY`. Reverted because it was placed above the `useState`
calls, violating React's rules of hooks and failing CI. If reinstated, it must
sit **after every hook** and before the first JSX return. The env var is