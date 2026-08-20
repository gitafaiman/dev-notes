# Running the Dev Server on a Different Port

How to override the port for `npm run dev`, depending on what the script actually wraps.

## Commands / Steps

First, check `package.json`'s `"scripts"` section for the `"dev"` entry — the right command depends on which tool it wraps.

**Next.js:**

```bash
npm run dev -- -p 3001
```

**Vite:**

```bash
npm run dev -- --port 3001
```

**Create React App:**

```bash
PORT=3001 npm run dev
```

**Plain Node/Express (custom `server.js`):** depends on how the port is read in the server file — often an env var like `PORT` or `NODE_PORT`. Same pattern as CRA above, or check the script for something like `node server.js` and look at how it reads `process.env`.

## Notes

- The `--` before the flag matters for Next.js and Vite — it tells npm "everything after this goes to the underlying command, not to npm itself." Without it, npm tries to parse the flag as its own argument.
- On Windows Git Bash, `PORT=3001 npm run dev` usually works as-is. If it doesn't, use `cross-env PORT=3001 npm run dev` (works cross-shell), or in `cmd.exe` specifically: `set PORT=3001 && npm run dev`.
- If you're not sure which tool the `dev` script wraps, look at the actual command string in `package.json` — e.g. `"dev": "next dev"` vs. `"dev": "vite"` vs. `"dev": "node server.js"`.

## Related

- `shell/kill-process-on-port.md`
