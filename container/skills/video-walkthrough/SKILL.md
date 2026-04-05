---
name: video-walkthrough
description: Record and send a video walkthrough of a web app using playwright-cli. Use when asked to record a UI walkthrough, demo a feature, or produce visual evidence for a PR.
allowed-tools: Bash(playwright-cli:*), mcp__nanoclaw__send_media, mcp__nanoclaw__send_message
---

# Video Walkthrough with playwright-cli

Record a video walkthrough of a running web app and send it.

## Prerequisites

The web app must already be running. Start it using the project's dev command (e.g. `mise run dev`),
then open a browser session pointing at it:

```bash
playwright-cli open <url>
```

## Basic recording

```bash
playwright-cli video-start
# interact with the app (see below)
playwright-cli video-stop --filename=/tmp/walkthrough.webm
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

To scroll an off-screen element into view before interacting:

```bash
playwright-cli eval "document.querySelector('...').scrollIntoView({behavior:'instant',block:'center'})"
```

## Polished recordings

For demos with chapter title cards and callout overlays, use `run-code` with the `page.screencast`
API:

```bash
playwright-cli run-code --file /tmp/walkthrough-script.js
```

```js
// /tmp/walkthrough-script.js — run after browser is already open
await page.screencast.start({ path: '/tmp/walkthrough.webm', size: { width: 1280, height: 720 } });

// Full-screen chapter card with blurred backdrop
await page.screencast.showChapter('Feature Name', {
  description: 'What this section demonstrates',
  duration: 2500,
});

// Navigate and interact
await page.goto('http://localhost:...');
await page.getByRole('textbox', { name: 'Email' }).fill('user@example.com');

// HTML overlay (pointer-events: none — won't block clicks)
const box = await page.locator('button').boundingBox();
const highlight = await page.screencast.showOverlay(
  `<div style="position:fixed;top:${box.y}px;left:${box.x}px;width:${box.width}px;height:${box.height}px;outline:2px solid #e55;border-radius:4px;pointer-events:none"></div>`
);
await page.click('button');
highlight.dispose();

await page.screencast.stop();
```

Key `page.screencast` APIs:
- `start({ path, size })` — begin recording to a file
- `showChapter(title, { description?, duration? })` — full-screen title card
- `showOverlay(html, { duration? })` — sticky HTML overlay; returns a disposable
- `disposable.dispose()` — remove a specific overlay
- `stop()` — finalize and write the video file

For natural-looking typing use `pressSequentially` with a delay:

```js
await page.getByRole('textbox').pressSequentially('hello world', { delay: 60 });
```

## Sending the result

After recording (either method), use the `mcp__nanoclaw__send_media` tool — not from inside a
`run-code` script, but as an agent tool call:

- `file_path`: `/tmp/walkthrough.webm`
- `media_type`: `video`
- `caption`: brief description of what the video shows

Copy to persistent workspace if you want it to survive a container reset:

```bash
cp /tmp/walkthrough.webm /workspace/group/walkthrough.webm
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `video-stop` → "Video recording has not been started" | Daemon died — re-run `playwright-cli open <url>` and restart recording |
| Refs fail with "not an input" | Stale refs — re-run `playwright-cli snapshot` to get fresh ones |
