---
name: video-walkthrough
description: Record and send a video walkthrough of a web app using playwright-cli. Use when asked to record a UI walkthrough, demo a feature, or produce visual evidence for a PR.
allowed-tools: Bash(playwright-cli:*), mcp__nanoclaw__send_media, mcp__nanoclaw__send_message
---

# Video Walkthrough with playwright-cli

Record a video walkthrough of a running web app and send it.

## Prerequisites

### playwright-cli availability

If the project uses mise and declares `"npm:@playwright/cli"` as a tool (e.g. in `mise.toml`),
`playwright-cli` is available after `mise install`. For any other project, run it via mise:

```bash
mise x "npm:@playwright/cli@0.1.1" -- playwright-cli <command>
```

### Install the browser

On a fresh container (or after the browser cache is cleared), install the browser before opening:

```bash
playwright-cli install-browser
```

### Open a browser session

The web app must already be running. Start it using the project's dev command (e.g. `mise run dev`),
then open a browser session pointing at it:

```bash
playwright-cli open <url>
```

## Recording

```bash
playwright-cli video-start
# interact with the app (see below)
playwright-cli video-stop --filename=/workspace/group/walkthrough.webm
```

`video-start` and `video-stop` work across separate bash calls — recording state lives in the
persistent daemon process, which is tied to the workspace directory.

## Interacting with pages

```bash
playwright-cli goto <url>         # navigate
playwright-cli snapshot           # get element refs
playwright-cli fill <ref> <text>  # fill an input
playwright-cli click <ref>        # click an element
playwright-cli screenshot         # capture current state
```

**Always re-snapshot** after navigation or significant DOM changes — refs go stale after re-renders.

To run multi-step interactions in one call (avoids repeated bash overhead):

```bash
playwright-cli run-code "async (page) => {
  await page.goto('http://localhost:5173');
  await page.getByRole('textbox', { name: 'Email' }).fill('user@example.com');
  await page.getByRole('textbox', { name: 'Password' }).fill('password');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForURL('**/');
}"
```

To scroll an off-screen element into view before interacting:

```bash
playwright-cli eval "document.querySelector('...').scrollIntoView({behavior:'instant',block:'center'})"
```

## Sending the result

Convert to MP4 before sending — WebM isn't handled natively by most messaging apps (Telegram,
WhatsApp) and requires a download to view:

```bash
ffmpeg -y -i /workspace/group/walkthrough.webm \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -movflags +faststart \
  /workspace/group/walkthrough.mp4
```

The video must be in `/workspace/group/` — the `mcp__nanoclaw__send_media` tool only reads from
there. Then call the tool:

- `file_path`: `/workspace/group/walkthrough.mp4`
- `media_type`: `video`
- `caption`: brief description of what the video shows

## Troubleshooting

- `video-stop` → "Video recording has not been started" — daemon died; re-run
  `playwright-cli open <url>` and restart recording
- Browser not found — run `playwright-cli install-browser` (no arguments)
- Refs fail with "not an input" — stale refs; re-run `playwright-cli snapshot` to get fresh ones
