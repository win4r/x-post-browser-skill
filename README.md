# x-post-browser-skill

Publish posts to **X (Twitter)** via **OpenClaw browser automation** (web UI) using:

- `browser` tool
- `profile="user"`
- X web UI (`https://x.com`)

This is **browser-only** (no API / no tokens / no Tweepy). It clicks the same UI you’d click.

## What it does

- Opens X
- Opens the **compose dialog** (preferred, more reliable than the inline home composer)
- Inserts your post text
- Clicks **Post**
- Verifies the post shows up and (best-effort) returns the permalink

It supports **two input modes**:

1) **Topic mode (draft + confirm)**
- You provide a topic/requirements
- The agent writes a Chinese post (~100 chars)
- Automatically appends **3–6 English hashtags**
- Sends you the draft first; only posts after you reply **“发布/发”**

2) **Raw-text mode (publish immediately)**
- You provide the final post text
- The agent posts it **verbatim**
- No second confirmation (by design)

## Requirements

- OpenClaw with the `browser` tool available
- A working **user browser profile** (`profile="user"`)
- You are already **logged in to X** in that browser profile

If X shows a login / consent / security prompt, complete it manually in the opened tab and then ask the agent to continue.

## Usage examples

### Topic mode (recommended)

Any of these should trigger topic mode:

- `发布一条X post：话题是 OpenClaw 终极形态，用中文，约100字`
- `帮我发个X：讲讲 agent memory governance`

Expected behavior:
1) Agent returns a **draft** (with English hashtags)
2) You reply `发布` (or `发`)
3) Agent publishes via browser

### Raw-text mode (direct publish)

Use a clear “final text” prefix, e.g.:

- `发X：我心中的 OpenClaw 终极形态…… #OpenClaw #AIAgents #AgenticAI`
- `原文：...（完整内容）... #OpenClaw`
- `/xpost ...（看起来像最终文案的完整文本）...`

Expected behavior:
- Agent posts immediately.

## Hashtag policy

- **English hashtags only** (default)
- Total **3–6** tags
- 1–2 broad searchable tags + 2–4 precise tags
- No irrelevant “trend hijacking”

## Important reliability note (X composer)

X’s post composer is `contenteditable` (Draft.js-like). Synthetic keystroke typing can:

- corrupt hashtags (e.g. render them as `####`)
- leave trailing residue

So this skill prefers **DOM insertion** via `evaluate` + `document.execCommand('insertText')`, plus a read-back verification before clicking Post.

## Safety notes

- Posting to X is a **public action**.
- Topic mode includes a confirmation step to reduce mis-posts.
- Raw-text mode skips confirmation for speed—use it only when you are sure the text is final.

## Troubleshooting

- **Composer / Post button not found**: ensure you’re on `https://x.com/home`, then retry opening the compose dialog.
- **Not interactive / timed out**: refresh, re-snapshot, retry once.
- **Redirected to login**: log in manually in the opened tab, then retry.
- **Post button disabled**: insertion may not be recognized; re-run the insertion method; if still disabled, stop and inspect UI errors.

## Repo contents

This repo is intentionally minimal:
- `SKILL.md` (the OpenClaw skill)
- `README.md` (this file)

## Packaging (optional)

If you want a distributable `.skill` file:

```bash
python3 /opt/homebrew/lib/node_modules/openclaw/skills/skill-creator/scripts/quick_validate.py .
python3 /opt/homebrew/lib/node_modules/openclaw/skills/skill-creator/scripts/package_skill.py . ./dist
```

You’ll get `./dist/x-post-browser.skill`.
