---
name: jolokia-configure-mcp-server
description: Connect (or disconnect) a Jolokia MCP server for the current project using `claude mcp add`/`remove` with local scope, instead of a committed .mcp.json entry or an environment variable. Use this whenever a jolokia-* skill is needed but no Jolokia MCP server is currently connected, or when the user wants to point Claude at a different Jolokia target.
---

# Connect a Jolokia MCP server (local scope, per-target)

The Jolokia target URL is personal and target-specific — it changes every time you point at a different host/pod, and it has no business being committed to a project's shared `.mcp.json` for every contributor to inherit. Register it per-machine instead, with `claude mcp add --scope local`.

## 1. Add the server, URL baked into the command

```bash
claude mcp add jolokia --scope local -- jbang org.jolokia.mcp:jolokia-mcp-server:0.5.1:runner -Djolokia.mcp.url=<jolokia-base-url>
```

`<jolokia-base-url>` is the full Jolokia HTTP endpoint, e.g. `http://localhost:8778/jolokia` or `http://some-host:8778/jolokia` — ask the user for it if it isn't already known; don't assume `localhost:8778`, that's only correct for a local demo target (see `jolokia-stream-jfr-recording` for why this matters when writing any script against the Jolokia HTTP endpoint directly).

This must be run from a terminal (`claude mcp add` is a CLI subcommand, not something callable through a running session's tools). It writes the entry to `~/.claude.json` under this project, not to the project's own `.mcp.json` — so it's private to this machine and never gets committed.

**Because the URL is baked into the add-time command, it's actually visible in the session** (via `claude mcp list` or the server's own launch args) — which is what makes the fallback script in `jolokia-stream-jfr-recording` step 3 possible: you can just ask the user, or find it via `claude mcp list`, rather than guessing.

## 2. It auto-connects from then on

Once added, `jolokia` reconnects automatically at the start of every future Claude Code session in this project — including sessions launched from an IDE plugin button, where there's no opportunity to set an environment variable before launch. No further setup needed until you want to point at a different target.

## 3. Switch targets or disconnect

There's no live "change the URL" for an already-added server — remove and re-add:

```bash
claude mcp remove jolokia
claude mcp add jolokia --scope local -- jbang org.jolokia.mcp:jolokia-mcp-server:0.5.1:runner -Djolokia.mcp.url=<new-jolokia-base-url>
```

If Jolokia isn't relevant to a session at all, just leave it unconfigured — an unadded server causes no error, no prompt, and no visible noise; there's nothing to silently fail.

## Why not an environment variable or a wrapper script

A `JOLOKIA_URL` env var (or a wrapper script requiring one) only works if the var is set in the shell *before* `claude` itself is launched — Claude Code connects all configured MCP servers eagerly at session startup, and a later `!export JOLOKIA_URL=...` inside a running session cannot retroactively reach an already-spawned process; there's also no mid-session "restart with new env" mechanism. Worse, launching Claude Code from an IDE plugin button (rather than a terminal) usually gives no chance to set an env var at all. `claude mcp add --scope local` sidesteps all of this: the URL lives in the add-time command itself, is stored once, and is picked up on every subsequent session launch regardless of how that session was started.
