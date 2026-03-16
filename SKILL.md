---
name: x-post-browser
description: "Publish posts to X (Twitter) via browser automation using the `browser` tool with profile=\"user\" (x.com web UI, not API). Use when the user asks to post/tweet on X, publish to x.com, or says \"发 X post\", and wants the agent to open x.com and click Post. Supports two modes: (1) topic mode (draft then user confirmation, auto English hashtags), (2) raw-text mode (user provides full post text, publish immediately)."
---

# X Post via Browser (profile="user")

## Mode selection (topic vs raw text)

Default to **topic mode** unless the user clearly provided final post text.

Treat as **raw-text mode (publish immediately)** if any of the following is true:
- The user message starts with an explicit “final text” prefix, e.g. `发X：`, `原文：`, `正文：`, `内容：`, `post：`, `tweet：`.
- The user uses a command-style trigger like `/xpost ...` where the remainder reads like the final post.
- The user says “按以下内容发布/照这个发/直接发/就发这段” and includes the full text.

Otherwise treat it as **topic mode (draft + confirm)**:
- User gives a topic, angle, requirements (length/tone), or asks you to “写一条/帮我发一条”.

## Topic mode: drafting rules

- Language: **Chinese**.
- Length target: **~100 Chinese characters** (stay comfortably within X’s limits after hashtags).
- End with **3–6 English hashtags** derived from the topic.
  - Prefer high-signal, searchable tags; avoid irrelevant “trend hijacking”.
  - Mix 1–2 broad tags + 2–4 precise tags.
- Output to the user:
  1) Draft text (exactly what will be posted)
  2) Ask for confirmation: user must reply **“发布”** / **“发”** to proceed.

## Raw-text mode: publishing rule

- Use the user-provided text **verbatim**.
- Publish immediately **without a second confirmation**.
- If the text is dangerously ambiguous (looks like a topic, not final copy), fall back to topic-mode confirmation.

## Browser workflow (must use profile="user")

### 1) Open X home

- `browser.open` with `profile="user"` → `https://x.com/home`
- If redirected to login/consent: ask the user to complete it in that tab, then continue.

### 2) Open the compose dialog (preferred)

The home inline composer can be flaky; prefer the dedicated composer:

- Click the left-side **“Post”** button/link (often `data-testid="SideNav_NewTweet_Button"` or `a[href="/compose/post"][aria-label="Post"]`).
- Confirm you now see a **dialog** composer with:
  - textbox `data-testid="tweetTextarea_0"` (aria-label: "Post text")
  - Post button `data-testid="tweetButton"`

If the dialog cannot be opened, fall back to the home inline composer.

### 3) Insert the post text (IMPORTANT)

X’s composer is `contenteditable` (Draft.js-like).

- Do **not** rely on keystroke `type` for hashtags/mentions. It can corrupt tags (e.g., render as `####`).
- Prefer `browser.act kind="evaluate"` with **`document.execCommand('insertText')`**.

Recommended `evaluate` payload (embed the final post text in `text`):

```js
() => {
  const text = '...final post text...';

  // Prefer dialog composer
  const el =
    document.querySelector('div[role="dialog"] [data-testid="tweetTextarea_0"]') ||
    document.querySelector('[role="textbox"][aria-label="Post text"]');

  if (!el) return { ok: false, err: 'composer not found' };

  el.focus();
  document.execCommand('selectAll', false, null);
  document.execCommand('delete', false, null);
  document.execCommand('insertText', false, text);

  // Verify exact text (prevents residue like trailing debug chars)
  const actual = (el.innerText || '').replace(/\r/g, '');
  if (actual !== text) {
    document.execCommand('selectAll', false, null);
    document.execCommand('delete', false, null);
    document.execCommand('insertText', false, text);
  }

  return { ok: true, actual: (el.innerText || '').replace(/\r/g, '') };
}
```

### 4) Preflight checks (must pass before clicking Post)

- **Text integrity check**
  - Read back the composer text (via `evaluate` or snapshot value).
  - Must match the intended text exactly.
  - Must not contain `####`.
- **Button enabled check**
  - In dialog mode: `button[data-testid="tweetButton"]` must not be disabled / `aria-disabled=true`.
  - If disabled: re-run the insertion once (Step 3). If still disabled, stop and report.

### 5) Publish

- Click the dialog Post button (`data-testid="tweetButton"`) if present; otherwise click the inline Post button.
- Wait ~2–3 seconds.

### 6) Verify success + capture permalink

- Navigate/return to `https://x.com/home`.
- Find the newly posted item (often marked “Now/1s”) containing the beginning of the text.
- Click into the post detail (click the timestamp like “1s” is usually easiest), then capture `location.href`.

## Reliability / fallbacks

- If elements aren’t interactive: refresh, re-snapshot, and retry once.
- If the composer is missing: navigate to `https://x.com/home`, then open compose dialog again.
- If hashtags appear as `####` or disappear: clear and re-insert using the `execCommand('insertText')` method.
- If posting fails due to UI/rate-limit errors: stop and report the exact on-screen error text.

## Examples (intended triggers)

- Topic mode:
  - “发布一条 X post：话题是 OpenClaw 终极形态，100 字，中文”
  - “帮我发个 X，讲讲 memory 的治理”

- Raw-text mode:
  - “发X：我心中的 OpenClaw … #OpenClaw #AIAgents”
  - “/xpost 我心中的 OpenClaw … #OpenClaw #AIAgents”
