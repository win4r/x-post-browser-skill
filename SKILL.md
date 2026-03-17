---
name: x-post-browser
description: "Publish posts to X (Twitter) via browser automation using the `browser` tool with profile=\"user\" (x.com web UI, not API). Use when the user asks to post/tweet on X, publish to x.com, or says \"发 X post\", and wants the agent to open x.com and click Post. Supports two modes: (1) topic mode (draft then user confirmation, auto English hashtags), (2) raw-text mode (user provides full post text, publish immediately)."
---

# X Post via Browser (profile="user")

## Mode selection (topic vs raw text)

Default to **topic mode** unless the user clearly provided final post text.

Treat as **raw-text mode (publish immediately)** if any of the following is true:
- The user message starts with an explicit "final text" prefix, e.g. `发X：`, `原文：`, `正文：`, `内容：`, `post：`, `tweet：`.
- The user uses a command-style trigger like `/xpost ...` where the remainder reads like the final post.
- The user says "按以下内容发布/照这个发/直接发/就发这段" and includes the full text.

Otherwise treat it as **topic mode (draft + confirm)**:
- User gives a topic, angle, requirements (length/tone), or asks you to "写一条/帮我发一条".

## Topic mode: drafting rules

- Language: **Chinese**.
- Length target: **~80 Chinese characters** (Chinese chars count as 2 toward X's 280 limit — keep total weighted length ≤ 240 to leave room for hashtags).
- End with **3–5 English hashtags** derived from the topic.
  - Prefer high-signal, searchable tags; avoid irrelevant "trend hijacking".
  - Mix 1–2 broad tags + 2–3 precise tags.
- Separate body from hashtags with `\n\n` (two newlines).
- Output to the user:
  1) Draft text (exactly what will be posted)
  2) Ask for confirmation: user must reply **"发布"** / **"发"** to proceed.

## Raw-text mode: publishing rule

- Use the user-provided text **verbatim**.
- Publish immediately **without a second confirmation**.
- If the text is dangerously ambiguous (looks like a topic, not final copy), fall back to topic-mode confirmation.

## ⚠️ Character limit rule

X counts characters as:
- ASCII / Latin: 1 character each
- CJK (Chinese/Japanese/Korean), emoji: **2 characters each**
- Total limit: **280 weighted characters**

Before publishing, estimate the weighted count. If over 260, shorten the text (trim hashtags first, then body). **Never attempt to post text that exceeds 280 weighted characters** — the Post button will be disabled.

## Browser workflow (must use profile="user")

All browser calls MUST include `profile="user"`.

### Step 1: Navigate to profile page, then open compose

First navigate to the profile page so that after posting, the dialog closes back to the profile where you can verify the new post:

```
browser.navigate → url: "https://x.com/AISuperDomain", profile: "user"
```

Wait 2 seconds for the page to load:

```
browser.act → kind: "wait", timeMs: 2000
```

Then open the compose dialog overlay:

```
browser.navigate → url: "https://x.com/compose/post", profile: "user"
```

Wait 2 seconds for the dialog to render:

```
browser.act → kind: "wait", timeMs: 2000
```

> **Why two navigations?** `/compose/post` opens as an overlay on the current page. By landing on the profile first, the compose dialog closes back to the profile page after posting — making it easy to verify the new post appeared.

### Step 2: Snapshot and locate elements

```
browser.snapshot → refs: "aria"
```

In the snapshot, identify:
- `textbox "Post text"` → note its `ref` (e.g. `5_12`)
- `button "Post"` inside the `dialog` → note its `ref` (e.g. `5_26`)

> ⚠️ There are TWO "Post" elements on the page: a navigation `link "Post"` (sidebar) and the compose `button "Post"` (inside the dialog). You must click the **button** inside the dialog, not the sidebar link.

### Step 3: Click the textbox to focus it

```
browser.act → kind: "click", ref: <textbox_ref>
```

### Step 4: Insert text via ClipboardEvent paste (THE ONLY RELIABLE METHOD)

Use `browser.act kind="evaluate"` with a ClipboardEvent:

```js
() => {
  const textbox = document.querySelector('[data-testid="tweetTextarea_0"]')
    || document.querySelector('div[contenteditable="true"][role="textbox"]');
  if (!textbox) return 'textbox not found';

  textbox.focus();

  const text = "YOUR POST TEXT HERE";
  const dataTransfer = new DataTransfer();
  dataTransfer.setData('text/plain', text);
  const pasteEvent = new ClipboardEvent('paste', {
    bubbles: true,
    cancelable: true,
    clipboardData: dataTransfer
  });
  textbox.dispatchEvent(pasteEvent);
  return 'pasted';
}
```

Replace `"YOUR POST TEXT HERE"` with the actual post text. Use `\n` in the JS string for newlines — the ClipboardEvent will render them as real line breaks.

> **⚠️ Trailing space after last hashtag:** Always append a space after the final hashtag (e.g. `"#AI #OpenClaw "` not `"#AI #OpenClaw"`). Without the trailing space, X's hashtag autocomplete dropdown stays open and can intercept the Post button click. The trailing space closes the dropdown.

> **Why ClipboardEvent?** X uses a React/Draft.js contenteditable editor.
> - ❌ `browser.act kind="type"` — renders `\n` as literal backslash-n text; corrupts hashtags into `####`.
> - ❌ `document.execCommand('insertText')` — deprecated; unreliable on Draft.js; may silently fail.
> - ✅ `ClipboardEvent('paste')` — triggers React's internal paste handler; correctly renders newlines, hashtags (turn blue), and emoji.

### Step 5: Verify input before posting

Take a screenshot:
```
browser.screenshot
```

Check:
- ✅ Post text is visible in the compose box
- ✅ Hashtags appear in **blue** (recognized by X)
- ✅ No `####` corruption
- ✅ The `Post` button is **blue/enabled** (not grayed out)
- ✅ No "exceeded character limit" warning

If the Post button is grayed out or character limit is exceeded:
1. Clear the textbox (see "Clearing text" below)
2. Shorten the text
3. Re-paste

### Step 6: Click the Post button to publish

```
browser.act → kind: "click", ref: <post_button_ref>
```

Wait 3 seconds:
```
browser.act → kind: "wait", timeMs: 3000
```

### Step 7: Verify success

Take a screenshot:
```
browser.screenshot
```

Success indicators:
- ✅ Compose dialog has closed (back to home timeline)
- ✅ "Your post was sent" toast may appear
- ✅ The post is visible at the top of the timeline

If the dialog is still open, the click may not have registered — click the Post button ref again.

## Utility: Clearing text from the composer

If you need to clear existing text (e.g., to fix a mistake), use JS `execCommand`:

```js
() => {
  const textbox = document.querySelector('[data-testid="tweetTextarea_0"]')
    || document.querySelector('div[contenteditable="true"][role="textbox"]');
  if (!textbox) return 'not found';
  textbox.focus();
  document.execCommand('selectAll');
  document.execCommand('delete');
  return 'cleared';
}
```

> ⚠️ Do NOT paste new text on top of old text — it will accumulate, not replace. Always clear first, then paste.

## Utility: Discarding a draft and starting fresh

If you need to start over completely:
1. Click the `Close` button (the X in the dialog corner)
2. A "Save post?" dialog appears → click `Discard`
3. Wait 1.5 seconds
4. Navigate to `x.com/AISuperDomain` first, then `x.com/compose/post`

## Reliability notes

- **Post button not responding:** Click it a second time. X sometimes needs two clicks.
- **Compose dialog not appearing:** Refresh the page, wait 2s, then navigate to `/compose/post` again.
- **Login required:** If redirected to login, ask the user to complete login manually, then retry.
- **Rate limit / error toast:** Stop and report the exact error text to the user.

## Examples (intended triggers)

- Topic mode:
  - "发布一条 X post：话题是 OpenClaw 终极形态，100 字，中文"
  - "帮我发个 X，讲讲 memory 的治理"

- Raw-text mode:
  - "发X：我心中的 OpenClaw … #OpenClaw #AIAgents"
  - "/xpost 我心中的 OpenClaw … #OpenClaw #AIAgents"
