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
- Length target: **~100 Chinese characters** (keep comfortably within X’s 280-char limit after adding hashtags).
- End with **3–6 English hashtags** derived from the topic.
  - Prefer high-signal, searchable tags; avoid irrelevant “trend hijacking”.
  - Mix 1–2 broad tags + 2–4 precise tags.
  - Examples (don’t hardcode): `#OpenClaw #AIAgents #AgenticAI #LLM #Automation`.
- Output to the user:
  1) Draft text (exactly what will be posted)
  2) Ask for confirmation: user must reply **“发布”** / **“发”** to proceed.

## Raw-text mode: publishing rule

- Use the user-provided text **verbatim**.
- Publish immediately **without a second confirmation**.
- If the text is dangerously ambiguous (looks like a topic, not final copy), fall back to topic-mode confirmation.

## Browser workflow (must use profile="user")

1) Open X home
- `browser.open` with `profile="user"` → `https://x.com/home`
- If redirected to login or blocked by consent prompts: ask the user to complete the login/consent in that browser tab, then continue.

2) Locate composer
- `browser.snapshot` with `refs="aria"`
- Find the textbox labeled like **“Post text”** / “What’s happening?”

3) Enter text (IMPORTANT)
- Click the textbox.
- Prefer **JS insertion via `browser.act kind="evaluate"`** (X’s composer is a `contenteditable`; `type` may corrupt hashtags and render them as `####`).

Example `evaluate` payload (embed the final post text in `text`):

```js
() => {
  const text = '...final post text...';
  const el = document.querySelector('[role="textbox"][aria-label="Post text"]');
  if (!el) return { ok: false, err: 'textbox not found' };
  el.focus();
  el.textContent = '';
  el.textContent = text;
  el.dispatchEvent(new InputEvent('input', { bubbles: true, data: text, inputType: 'insertText' }));
  return { ok: true };
}
```

- Only fall back to `browser.act kind="type"` if `evaluate` is blocked/unavailable.

4) Publish
- Click the **“Post”** button.
- Wait ~2–3 seconds.

5) Verify success
- `browser.snapshot` again and confirm a newly posted item appears in the timeline (often marked “Now”) containing the posted text.
- If possible, click into the post detail and report the final URL; if clicking times out, report that it was posted and offer to fetch the link on request.

## Reliability / fallbacks

- If elements aren’t interactive: refresh the page, re-snapshot, and retry once.
- If the composer is missing: ensure you are on `https://x.com/home` (navigate there explicitly).
- If hashtags show up as `####` or tag text disappears: clear the composer and re-insert the full text using the **`evaluate` method** above.
- If posting fails due to rate limits or UI errors: stop and report the exact on-screen error text.

## Examples (intended triggers)

- Topic mode:
  - “发布一条 X post：话题是 OpenClaw 终极形态，100 字，中文”
  - “帮我发个 X，讲讲 memory 的治理”

- Raw-text mode:
  - “发X：我心中的 OpenClaw … #OpenClaw #AIAgents”
  - “/xpost 我心中的 OpenClaw … #OpenClaw #AIAgents”
