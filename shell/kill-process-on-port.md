# Kill a Process on a Port (Windows / Git Bash)

Find and terminate whatever's listening on a given port — e.g. a stuck Next.js dev server on port 3000.

## Commands / Steps

1. Check if something's running on the port:

```bash
netstat -ano | grep :3000
```

If you see a line like this, the server is up:

```
TCP    0.0.0.0:3000    0.0.0.0:0    LISTENING    12345
```

The number at the end (`12345`) is the PID — you'll need it in the next step.

2. Confirm it in the browser (optional):

Visit `http://localhost:3000` — if the app loads, it's running.

3. Terminate it:

```bash
taskkill //PID 12345 //F
```

Swap `12345` for the actual PID from step 1.

4. One-liner to do it all at once (find + kill in one go):

```bash
taskkill //PID $(netstat -ano | grep :3000 | awk '{print $5}' | head -1) //F
```

5. If nothing shows up on `netstat`, nothing's running on that port — restart normally:

```bash
npm run dev
```

## Notes

- Swap `3000` for whatever port you're checking — this isn't Next.js-specific, it's just the common default for `npm run dev`.
- Works the same for any process bound to a local port, not just dev servers.

## Gotchas

- The double slashes in `taskkill //PID ... //F` are required in Git Bash — a single slash (`/PID`, `/F`) gets misinterpreted as a Windows file path and the command fails silently or errors oddly.
- If multiple processes show up on `netstat -ano | grep :3000` (e.g. one for IPv4, one for IPv6), the one-liner grabs the first PID via `head -1` — double-check that's the right one before killing if it matters.

## Related

- `shell/useful-one-liners.md`
