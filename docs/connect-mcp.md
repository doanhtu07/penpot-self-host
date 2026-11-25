# Connecting to the Penpot MCP server

How to plug this self-hosted Penpot instance into an AI client (OpenCode, Claude Code,
Claude Desktop) over MCP, and how to get the Penpot MCP plugin to actually see a file.

Endpoint (remote MCP, proxied by the Penpot frontend):

```text
http://localhost:9001/mcp/stream?userToken=<your MCP key>
```

> Tested against `penpotapp/*:2.18` images, OpenCode `2.4.1`, Claude Code `2.1.332`.

## Already installed on this machine

Both clients are configured **globally** (no per-project config, nothing in the repo):

| Client      | Config file                         | Scope         |
| ----------- | ----------------------------------- | ------------- |
| OpenCode    | `~/.config/opencode/opencode.jsonc` | every project |
| Claude Code | `~/.claude.json` (`--scope user`)   | every project |

The URL is embedded literally in both, so nothing has to be exported before launching
either client. Sections 6 and 7 cover reinstalling and rotating the key.

---

## 1. How it fits together

There are four moving parts. Every one of them has to be up for a tool call to succeed.

| Piece                  | What it is                                                     | Where it lives                           |
| ---------------------- | -------------------------------------------------------------- | ---------------------------------------- |
| `penpot-mcp` container | The MCP server (`penpotapp/mcp`)                               | internal Docker network, not published   |
| Penpot frontend        | Reverse proxy for `/mcp/*`                                     | `http://localhost:9001`                  |
| MCP plugin             | Runs **inside your Penpot browser tab** and owns the open file | Penpot → **File → MCP Server → Connect** |
| AI client              | OpenCode / Claude Code / Desktop                               | your machine                             |

Two consequences worth internalizing early:

- The client talks to the MCP server, but the MCP server can only act through the plugin.
  **No connected plugin = no access to any design file.**
- The plugin lives in one browser tab. Tools operate on the **currently focused page** of
  that tab.

---

## 2. Prerequisites

1. The stack is running:

   ```sh
   pnpm start
   docker compose -p penpot ps
   ```

   You need `penpot-frontend`, `penpot-backend` and `penpot-mcp` healthy.

2. The `enable-mcp` flag is on. It already is in `docker-compose.yaml`:

   ```yaml
   x-flags: &penpot-flags
     PENPOT_FLAGS: disable-email-verification enable-smtp enable-prepl-server disable-secure-session-cookies enable-mcp
   ```

   With that flag the frontend proxies `/mcp/*` to `PENPOT_MCP_URI` (default
   `http://penpot-mcp:4401`). If you renamed the service or changed ports, set
   `PENPOT_MCP_URI` / `PENPOT_MCP_URI_WS` on the **frontend** container.

3. Sanity check that the proxy is alive:

   ```sh
   curl -s -o /dev/null -w "%{http_code}\n" http://localhost:9001/
   ```

   A `200` means the frontend is serving.

---

## 3. Get your MCP URL (once per user)

In Penpot:

1. **Your account → Integrations → MCP Server**.
2. Flip **Status** to enabled.
3. Generate an **MCP key** if you do not have one. It is shown **only once** — copy it
   immediately.
4. Copy the **server URL**; it already contains the key as `userToken`:

   ```text
   http://localhost:9001/mcp/stream?userToken=eyJhbGciOi...
   ```

Notes:

- One key per user. Generating a new one (or **Regenerate**) immediately revokes the old
  one, and every client using it stops working.
- The key has an expiry date. An expired key fails the same way a wrong key does.
- Regenerating is also the correct response if the key ever leaks.

---

## 4. Put it in `.env`

`.env` is already gitignored (`.gitignore` lists `.env` and `.env*.local`). Create it
from the template:

```sh
pnpm copy-env
```

Then add:

```sh
PENPOT_MCP_TOKEN=eyJhbGciOi...
PENPOT_MCP_URL=http://localhost:9001/mcp/stream?userToken=eyJhbGciOi...
```

`PENPOT_MCP_TOKEN` is used by env-referencing configs; `PENPOT_MCP_URL` is the convenience
copy for shell commands (`curl`, `claude mcp add --url …`, `opencode mcp add --url …`).

### Load it into your shell — only if your config references the variable

The global setup used on this machine (§6, §7) **embeds the URL in each client's own config
file**, so `.env` is just the place you keep the key, and nothing needs exporting.

If instead you configure a client with `{env:PENPOT_MCP_TOKEN}` or `${PENPOT_MCP_TOKEN}`
(the git-safe variants), then the variable must exist in the environment of the process that
starts the client — **neither OpenCode nor Claude Code reads `.env` on its own**:

```sh
set -a; source .env; set +a
```

Verify it stuck:

```sh
echo "${PENPOT_MCP_TOKEN:0:8}..."
```

If you would rather not think about this, use [`direnv`](https://direnv.net) with a
`.envrc` containing `dotenv`. Otherwise a missing variable silently becomes an empty
string, which produces the confusing `...?userToken=` URL described in
[Troubleshooting](#9-troubleshooting).

---

## 5. Smoke test before touching any client

`initialize` and `tools/list` succeed even with a bogus token — the token is only checked
when a tool needs a Penpot user. So the useful check is: can we complete a handshake and
see the tool list?

```sh
SID=$(
  curl -s -D - -o /dev/null -X POST "$PENPOT_MCP_URL" \
    -H 'Content-Type: application/json' \
    -H 'Accept: application/json, text/event-stream' \
    -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"0"}}}' \
  | tr -d '\r' | awk -F': ' 'tolower($1)=="mcp-session-id"{print $2}'
)
echo "session: $SID"

curl -s -X POST "$PENPOT_MCP_URL" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
| grep -o '"name":"[a-z_]*"' | sort -u
```

Expected tools:

```text
"name":"execute_code"
"name":"export_shape"
"name":"high_level_overview"
"name":"penpot_api_info"
```

The transport is Streamable HTTP, so responses arrive as SSE frames:

```text
event: message
data: {"result":{"protocolVersion":"2025-06-18", ...},"jsonrpc":"2.0","id":1}
```

A `404` here means you hit `/mcp` instead of `/mcp/stream`.

---

## 6. OpenCode

OpenCode (V2) keeps servers under `mcp.servers`.

### Option A — global CLI (this machine's setup)

```sh
set -a; source .env; set +a
opencode mcp add penpot --global --url "$PENPOT_MCP_URL"
```

Writes `~/.config/opencode/opencode.jsonc`, applied to every project:

```json
{
  "mcp": {
    "servers": {
      "penpot": {
        "type": "remote",
        "url": "http://localhost:9001/mcp/stream?userToken=eyJ..."
      }
    }
  }
}
```

Two things the CLI does not do for you:

- The key now sits in a file outside the repo — `chmod 600 ~/.config/opencode/opencode.jsonc`.
- Add `"oauth": false` to the entry (see the bullets below).

Because the URL is literal, nothing needs exporting when you launch OpenCode, and the
background service's environment stops mattering. The cost is duplication: regenerating the
key means re-running this command.

### Option B — project scope

Drop `--global` and the server goes to `./opencode.json` instead — this project only. Handy,
but the key lands in a file _inside_ the repo, so keep it out of git or use Option C.

### Option C — env reference (git-safe, needs the export)

Create `opencode.jsonc` in the project root (or `.opencode/opencode.jsonc`):

```jsonc title="opencode.jsonc"
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "penpot": {
        "type": "remote",
        "url": "http://localhost:9001/mcp/stream?userToken={env:PENPOT_MCP_TOKEN}",
        // Penpot authenticates with the userToken query param, not OAuth.
        "oauth": false,
        // Optional: keep MCP tools on the native tool list instead of Code Mode.
        "codemode": false,
      },
    },
  },
}
```

- `oauth: false` matters: remote servers default to OAuth, and Penpot answers the
  handshake without auth, so you would otherwise be sent down a sign-in flow that can
  never succeed.
- With the default Code Mode, tools are grouped and called from `execute`, e.g.
  `tools.penpot.execute_code({ code: "..." })`. Set `"codemode": false` if you want them
  as plain tools.
- A `?` in a shell argument needs quoting: `opencode mcp add penpot --url "$PENPOT_MCP_URL"`.

### Verify

```sh
opencode mcp list
```

```text
✓ penpot  connected
```

Inside OpenCode, `/mcps` shows the same state and lets you reconnect. If the list looks
empty on a fresh terminal, run it once more or with `--print-logs`; the background service
needs a moment to pick up a directory it has not opened yet.

> `✓ connected` is **not** proof the token resolved. OpenCode completes the MCP handshake
> against `...?userToken=` just fine, then every real tool call fails with
> `No userToken found in session context`. That happens whenever the background service was
> started before `.env` was exported, because the service keeps the environment it was born
> with. Fix with `set -a; source .env; set +a` then `opencode service restart` (or test with
> `opencode --standalone`), or start OpenCode from a shell that already has the variable.

To confirm the token actually resolved to something, dump the effective config:

```sh
opencode debug config | grep -A6 '"penpot"'
```

The URL is printed as `"***"` (secrets are redacted), so instead confirm that OpenCode
found the entry at all and that `$PENPOT_MCP_TOKEN` is non-empty in the same shell.

### Scope note

`opencode mcp add --global` puts the server in `~/.config/opencode/opencode.json(c)` for
every project. Project config wins over global config for the same server name, and a
higher-precedence entry replaces the whole object — repeat `type`, `url` and `oauth` if you
override it.

---

## 7. Claude Code

### Add it — global (this machine's setup)

```sh
set -a; source .env; set +a
claude mcp add --transport http penpot "$PENPOT_MCP_URL" --scope user
```

`--scope user` writes to `~/.claude.json` for every project, so nothing lands in the repo
and no variable needs exporting when you launch `claude`. The key is stored in that file;
regenerating it means re-running this command (`claude mcp remove penpot --scope user`
first if you want a clean swap).

Scope flags:

| Flag                      | Stored in                     | Use when                               |
| ------------------------- | ----------------------------- | -------------------------------------- |
| `--scope local` (default) | `~/.claude.json`, per project | personal key, this repo only           |
| `--scope user`            | `~/.claude.json`, global      | personal key, every project            |
| `--scope project`         | `.mcp.json` at repo root      | shared config (keep the key out of it) |

### Or a shared `.mcp.json`

```json
{
  "mcpServers": {
    "penpot": {
      "type": "http",
      "url": "http://localhost:9001/mcp/stream?userToken=${PENPOT_MCP_TOKEN}"
    }
  }
}
```

`.mcp.json` supports `${VAR}` expansion, so this file is safe to commit. `"type": "http"`
is required — an entry with a `url` and no `type` is treated as a stdio server and
skipped. Project-scoped servers also need a one-time approval when you start `claude` in
the repo.

### Verify

```sh
claude mcp list          # → penpot: http://localhost:9001/mcp/stream?userToken=… - ✔ Connected
claude mcp get penpot    # full detail, including status and scope
```

Inside a session, `/mcp` shows the server, its tool count, and reconnect actions. Tools are
exposed as `mcp__penpot__execute_code`, `mcp__penpot__high_level_overview`, etc.

Remove with `claude mcp remove penpot --scope user`.

### Claude Desktop / other clients

Desktop only accepts HTTPS connectors, so `http://localhost:9001` is not addable directly.
Bridge it over stdio:

```json
{
  "mcpServers": {
    "penpot": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "http://localhost:9001/mcp/stream?userToken=YOUR_MCP_KEY",
        "--allow-http"
      ]
    }
  }
}
```

Or front Penpot with HTTPS (Traefik/Caddy) and register
`https://<your-domain>/mcp/stream?userToken=…` as a custom connector.

---

## 8. Everyday workflow

Every session, in this order:

1. Penpot → **Your account → Integrations → MCP Server** → status **Enabled**.
2. Open the design file you want the agent to work on.
3. **File → MCP Server → Connect**.
4. Keep that tab focused-ish and awake. MCP follows the focused page, and only one tab can
   own MCP at a time. If you have several Penpot tabs open, pick one explicitly.
5. Start the client, then send a **read-only** prompt first:

   ```text
   List the pages in this file.
   ```

Then move to writes. Be deliberate here: a connected agent can rename, move, delete and
restyle anything on the focused page. Ask for a plan before large edits, and prefer small
steps.

Browsers park background tabs. To avoid the plugin being frozen mid-session:

| Browser | Setting                                                                                                |
| ------- | ------------------------------------------------------------------------------------------------------ |
| Chrome  | Settings → Performance → **Always keep these sites active** → add `http://localhost:9001`; pin the tab |
| Edge    | Settings → System and performance → never put this site to sleep                                       |
| Firefox | Keep the tab active; advanced: `browser.tabs.unloadOnLowMemory=false`                                  |
| Safari  | Keep the tab active; no per-site control                                                               |

---

## 9. Troubleshooting

| Symptom                                                                                                                                              | Likely cause                                                                                          | Fix                                                                                                   |
| ---------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `No Penpot instance connected for user token. Please ensure that Penpot is connected and that the MCP client connection is using the correct token.` | Plugin not connected, wrong/expired key, or empty `userToken`                                         | Steps 1–4 of §8; check the key in `.env`; regenerate if expired                                       |
| URL shows `?userToken=` with nothing after it                                                                                                        | `PENPOT_MCP_TOKEN` was not exported before starting the client                                        | `set -a; source .env; set +a`, then fully restart the client                                          |
| `initialize` + `tools/list` work but every real tool call fails                                                                                      | Token never reached the server, or no plugin connection                                               | Same as the first row                                                                                 |
| OpenCode: `No userToken found in session context. Multi-user mode requires authentication.`                                                          | The background service started before `.env` was exported, so `{env:PENPOT_MCP_TOKEN}` resolved empty | `set -a; source .env; set +a` then `opencode service restart`, or launch with `opencode --standalone` |
| `claude mcp list` shows `✘ Failed to connect`                                                                                                        | Container down, or wrong port/host                                                                    | `docker compose -p penpot ps`; `curl -s -o /dev/null -w "%{http_code}" http://localhost:9001/`        |
| `✘ Needs authentication` / an OAuth prompt you cannot finish                                                                                         | Remote servers default to OAuth                                                                       | Remove and re-add with `"oauth": false` (OpenCode) or a plain `--transport http` URL (Claude Code)    |
| `404` from the endpoint                                                                                                                              | Wrong path                                                                                            | Use `/mcp/stream`, not `/mcp`                                                                         |
| Tools listed, but the agent says it cannot see the design                                                                                            | Penpot tab suspended/frozen, or a different tab owns MCP                                              | Focus the Penpot tab, wait for reconnection, retry                                                    |
| Worked yesterday, broken today                                                                                                                       | Key expired, or regenerated                                                                           | Regenerate, update `.env`, restart the client                                                         |
| OpenCode `mcp list` prints "No MCP servers configured"                                                                                               | Background service has not loaded this directory yet                                                  | Re-run; check `opencode debug config` lists your file                                                 |
| OpenCode sees the server but no Penpot tools                                                                                                         | Code Mode is on                                                                                       | Call through `execute` as `tools.penpot.*`, or set `"codemode": false`                                |
| Everything looks right after an upgrade                                                                                                              | `penpot-mcp` container version drift                                                                  | `docker compose -p penpot up -d` to recreate all services on the pinned tag                           |

Debug lens: watch the MCP container while you retry a prompt.

```sh
docker compose -p penpot logs -f penpot-mcp
```

---

## 10. Security

- Treat the MCP key like a password. It grants read **and write** access to your Penpot
  files through any connected client, and it is embedded in a URL, so it easily ends up in
  shell history, screenshots and logs.
- Never commit it. `.env` is gitignored; if you use `.mcp.json`, use `${VAR}`; if you use
  a project `opencode.json` from `opencode mcp add`, add that file to `.gitignore` or switch
  to `{env:PENPOT_MCP_TOKEN}`.
- The global setup used here puts the key in two files outside the repo:
  `~/.config/opencode/opencode.jsonc` and `~/.claude.json`. Both are plain text. Keep the
  first at `chmod 600`, and treat both like `~/.ssh` — regenerating the key means updating
  both (`opencode mcp add penpot --global …`, `claude mcp remove/add --scope user`).
- Clear shell history or use a leading space (with `HIST_IGNORE_SPACE`) when pasting the URL
  on the command line, since `claude mcp add` echoes it.
- Rotate by regenerating the key in Penpot; the old one dies immediately.
- Turning the Integrations toggle off, or disconnecting the plugin, stops agents from
  touching files even while a client stays configured. That is your fastest kill switch.
- This setup has no TLS because it is `localhost`. If you expose the instance, put it
  behind HTTPS; the key travels in a query string.

---

## 11. Alternative: local MCP server

If you need local filesystem access (import images from disk, export straight to a path),
run the local server instead of the proxied one:

```sh
npx -y @penpot/mcp@stable
```

Then in Penpot: **Plugins → Load from URL →** `http://localhost:4400/manifest.json`, run the
plugin, click **Connect to MCP server**, and point your client at `http://localhost:4401/mcp`
(no token needed; it uses the plugin session). Keep the terminal running.

Remote (this doc) is the simpler default; local adds file access at the cost of a process
you must babysit.

---

## References

- Penpot MCP help: <https://help.penpot.app/mcp/>
- Penpot configuration (`enable-mcp`, `PENPOT_MCP_URI`): <https://help.penpot.app/technical-guide/configuration/>
- MCP README (repo): <https://github.com/penpot/penpot/blob/main/mcp/README.md>
- OpenCode MCP: <https://opencode.ai/docs/mcp-servers/>
- Claude Code MCP: <https://code.claude.com/docs/en/mcp>
