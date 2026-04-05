---
name: video-walkthrough
description: Record and send a video walkthrough of the Tend web app using playwright-cli. Use when asked to record a UI walkthrough, demo a feature, or produce visual evidence for a PR.
allowed-tools: Bash(playwright-cli:*), Bash(mise:*), Bash(curl:*), Bash(go:*), Bash(node:*), Bash(pnpm:*), mcp__nanoclaw__send_media, mcp__nanoclaw__send_message
---

# Video Walkthrough Skill

Record a video of the Tend web app and send it to the user.

## Prerequisites: Dev Environment

Before recording, the dev environment must be running. Check first:

```bash
NO_PROXY=localhost,127.0.0.1 curl -s http://localhost:8080/healthz
```

If the server isn't up, start everything manually:

```bash
# 1. Start postgres (branch-namespaced PGDATA from mise env)
cd /workspace/group/tend
PGDATA=$(mise exec -- bash -c 'echo $PGDATA')
mkdir -p "$PGDATA"
[ ! -f "$PGDATA/PG_VERSION" ] && mise exec -- initdb -D "$PGDATA" -U postgres --auth=trust --no-locale -E UTF8
mise exec -- postgres -D "$PGDATA" -p 5432 > /tmp/postgres.log 2>&1 &
sleep 3 && mise exec -- pg_isready -h localhost -p 5432 -U postgres

# 2. Create DB and run migrations
mise exec -- createdb -h localhost -p 5432 -U postgres tend 2>/dev/null || true
mise exec -- bash -c 'TEND_DB="postgres://postgres@localhost:5432/tend?sslmode=disable" TEND_JWT_SECRET=dev-secret go run ./server/cmd/server migrate up'

# 3. Start server (with initial admin user seeding on first run)
mise exec -- bash -c 'TEND_DB="postgres://postgres@localhost:5432/tend?sslmode=disable" TEND_JWT_SECRET=dev-secret TEND_HIBP_ENABLED=false TEND_INIT_OWNER_EMAIL=admin@localhost TEND_INIT_OWNER_NAME=Admin TEND_INIT_OWNER_PASSWORD=password go run ./server/cmd/server serve' > /tmp/server.log 2>&1 &
sleep 6 && NO_PROXY=localhost,127.0.0.1 curl -s http://localhost:8080/healthz

# 4. Start web dev server
cd /workspace/group/tend && mise run //web:dev > /tmp/web.log 2>&1 &
sleep 6 && NO_PROXY=localhost,127.0.0.1 curl -s -o /dev/null -w "%{http_code}" http://localhost:5173/
```

**Login credentials**: `admin@localhost` / `password`

## Prerequisites: Browser

Ensure the playwright-cli browser session is open:

```bash
NO_PROXY=localhost,127.0.0.1 no_proxy=localhost,127.0.0.1 playwright-cli list
```

If no browsers are listed:

```bash
NO_PROXY=localhost,127.0.0.1 no_proxy=localhost,127.0.0.1 playwright-cli open http://localhost:5173
```

If `open` fails with "Browser not installed":

```bash
node /home/node/.local/share/mise/installs/npm-playwright-cli/0.1.1/lib/node_modules/@playwright/cli/node_modules/playwright/cli.js install chromium
NO_PROXY=localhost,127.0.0.1 no_proxy=localhost,127.0.0.1 playwright-cli open http://localhost:5173
```

**Always set `NO_PROXY=localhost,127.0.0.1 no_proxy=localhost,127.0.0.1`** — the container proxy intercepts localhost traffic.

## Simple Recording

For functional walkthroughs (PR evidence, quick demos):

```bash
# Start recording
NO_PROXY=localhost,127.0.0.1 no_proxy=localhost,127.0.0.1 playwright-cli video-start

# Log in
playwright-cli goto http://localhost:5173
playwright-cli snapshot          # get refs
playwright-cli fill <email-ref> admin@localhost
playwright-cli fill <password-ref> password
playwright-cli click <submit-ref>

# Walk through the feature...
playwright-cli goto http://localhost:5173/spaces/demo/tasks/T1
playwright-cli eval "document.querySelector('...').scrollIntoView({behavior:'instant',block:'center'})"
playwright-cli screenshot        # capture key moments

# Stop and save
NO_PROXY=localhost,127.0.0.1 no_proxy=localhost,127.0.0.1 playwright-cli video-stop --filename=/tmp/walkthrough.webm
```

**Key**: `video-start` and `video-stop` work across separate bash calls — state lives in the persistent daemon. All intermediate `playwright-cli` commands don't need `NO_PROXY` (only `open`/`video-start`/`video-stop` communicate with the daemon).

## Polished Recording (Chapter Cards + Overlays)

For demos with chapter titles, callout overlays, and smooth narration — use `run-code`:

```bash
playwright-cli run-code --file /tmp/walkthrough-script.js
```

```js
// /tmp/walkthrough-script.js
// Run after browser is open: playwright-cli run-code --file /tmp/walkthrough-script.js

await page.screencast.start({ path: '/tmp/walkthrough.webm', size: { width: 1280, height: 720 } });

// Chapter card (full-screen title card with blurred backdrop)
await page.screencast.showChapter('Task Relations', {
  description: 'Add, view, and remove dependencies between tasks',
  duration: 2500
});

// Navigate and interact
await page.goto('http://localhost:5173');
await page.getByRole('textbox', { name: 'Email' }).fill('admin@localhost');
await page.getByRole('textbox', { name: 'Password' }).fill('password');
await page.getByRole('button', { name: 'Sign in' }).click();
await page.waitForURL('**/');

// Highlight an element with an overlay (pointer-events: none, won't block clicks)
const box = await page.locator('input[placeholder="T42"]').boundingBox();
const highlight = await page.screencast.showOverlay(
  `<div style="position:fixed;top:${box.y - 4}px;left:${box.x - 4}px;width:${box.width + 8}px;height:${box.height + 8}px;border:2px solid #e67;border-radius:4px;pointer-events:none"></div>`
);
await page.locator('input[placeholder="T42"]').fill('T1');
highlight.dispose();

await page.screencast.stop();
```

Key APIs:
- `page.screencast.start({ path, size })` — begin recording
- `page.screencast.showChapter(title, { description, duration })` — full-screen chapter card
- `page.screencast.showOverlay(html, { duration? })` — HTML overlay (sticky unless disposed)
- `disposable.dispose()` — remove an overlay
- `page.screencast.stop()` — finalize the file

## Sending the Video

After recording, send to the user with a caption:

```js
// Use the MCP send_media tool:
mcp__nanoclaw__send_media({
  file_path: '/tmp/walkthrough.webm',
  media_type: 'video',
  caption: 'SV-XX walkthrough — brief description of what was shown'
})
```

Or copy to the persistent workspace first (survives container reset):

```bash
cp /tmp/walkthrough.webm /workspace/group/walkthrough.webm
```

## Working with the Tend App

### Snapshot workflow

```bash
playwright-cli snapshot          # get current page elements
# Output example:
# - textbox "Email" [ref=e14]
# - button "Sign in" [ref=e18]

playwright-cli fill e14 admin@localhost
playwright-cli click e18
playwright-cli snapshot          # re-snapshot after navigation
```

**Always re-snapshot after**: navigation, clicking buttons that cause re-renders, selecting dropdowns. Refs go stale after React re-renders.

### Scrolling to off-screen elements

```bash
playwright-cli eval "document.querySelector('input[placeholder=\"T42\"]')?.scrollIntoView({behavior:'instant',block:'center'})"
```

### Common Tend routes

| Route | Purpose |
|---|---|
| `/` | My Tasks (home) |
| `/spaces/new` | Create space |
| `/spaces/:slug` | Space task list |
| `/spaces/:slug/tasks/:id` | Task detail (properties, relations) |
| `/login` | Login page |

### Task ID format

Tasks are identified as `T1`, `T2`, etc. (short IDs shown in the UI, not UUIDs).

## Troubleshooting

| Problem | Fix |
|---|---|
| `playwright-cli open` → "Browser not installed" | Run `node .../playwright/cli.js install chromium` (see Prerequisites) |
| `video-stop` → "Video recording has not been started" | The daemon died — re-open the browser and re-start recording |
| `curl localhost` times out | Set `NO_PROXY=localhost,127.0.0.1 no_proxy=localhost,127.0.0.1` |
| Login fails with 400 | Server running but no admin user — restart server with `TEND_INIT_OWNER_*` vars |
| `playwright-cli fill` fails with "not an input" | Refs are stale — re-run `playwright-cli snapshot` to get fresh refs |
