# PGRST303 — "JWT issued at future"

## Symptom

```json
{
  "error": "Failed to load data",
  "details": {
    "code": "PGRST303",
    "details": null,
    "hint": null,
    "message": "JWT issued at future"
  }
}
```

Typically surfaces as a `500` from any API route that calls Supabase/PostgREST with a Clerk-issued JWT (e.g. `/api/bootstrap`).

## Root cause

PostgREST rejects the request because the JWT's `iat` (issued-at) claim is later than what the Postgres/PostgREST server thinks "now" is. In practice this means the local dev machine's system clock is not synced to real time (Windows Time service never completed an NTP sync, running off the drifting CMOS/hardware clock instead).

Not a code bug — no custom JWT signing in the app; tokens come straight from Clerk (`session.getToken()` client-side / `auth().getToken()` server-side) against a hosted (non-local) Supabase project.

## Diagnose

```powershell
w32tm /query /status
```

Red flags:

- `Leap Indicator: 3 (not synchronized)`
- `Source: Local CMOS Clock`
- `Last Successful Sync Time: unspecified`

## Fix

1. Open PowerShell as Administrator (Start → type PowerShell → Run as administrator).
2. Force a resync:

```powershell
w32tm /resync /force
```

3. If that errors out (common when the time service was never configured), reset it first, then retry:

```powershell
w32tm /config /manualpeerlist:"time.windows.com" /syncfromflags:manual /update
Restart-Service w32time
w32tm /resync /force
```

4. Verify:

```powershell
w32tm /query /status
```

Expect `Leap Indicator: 0 (no warning)` and a real `Last Successful Sync Time`.

5. Belt-and-suspenders: Settings → Time & Language → Date & time → confirm "Set time automatically" is On, and click Sync now.
6. Restart `next dev` and retry the failing request.

## Notes

- Clerk mints the JWT server-side with an accurate `iat`; the mismatch happens because Clerk's local session-verification step (in `@clerk/nextjs` middleware/`auth()`) and PostgREST's own `iat` check both compare against clocks that can disagree with the local machine — an unsynced local clock is the most common trigger reported in Clerk + Supabase local-dev setups.
- If this happens again shortly after a laptop sleep/resume, that's consistent with clock drift too — rerun the resync steps above.

## Related

- `shell/kill-process-on-port.md`
