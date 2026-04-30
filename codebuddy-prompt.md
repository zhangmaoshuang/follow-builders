# Follow Builders Digest — Universal Agent Prompt

This file is a self-contained instruction for any AI coding agent (CodeBuddy,
Cursor, Continue, Cline, Aider, or any other) to fetch the daily AI Builders
Digest and deliver it via email.

If you are an AI agent reading this, follow the steps below **literally** —
do not skip steps, do not invent content, do not visit any URL not listed here.

---

## How the user invokes this

The user will say something like:
- "按 d:\AI\skills\follow-builders\codebuddy-prompt.md 的指令执行"
- "Run the follow-builders digest"
- "给我发今天的资讯"
- "Send me today's AI digest"

When you see any such request, execute Steps 1–6 below in order.

---

## Step 1 — Fetch the data

Run this shell command and capture stdout:

```bash
cd "d:/AI/skills/follow-builders/scripts" && node prepare-digest.js
```

The output is a single JSON blob. Parse it. Key fields you will use:

- `config.language` — `"en"`, `"zh"`, or `"bilingual"`
- `config.delivery.email` — destination email
- `x` — array of builders, each with `name`, `handle`, `bio`, `tweets[]`
- `podcasts` — array of podcast episodes (often empty)
- `blogs` — array of blog posts (often empty)
- `prompts.summarize_tweets` — rules for summarizing tweets
- `prompts.summarize_podcast` — rules for summarizing podcasts
- `prompts.digest_intro` — overall format rules
- `prompts.translate` — translation rules (only if language is `zh` or `bilingual`)

If the script fails entirely, tell the user "无法拉取 feed，请检查网络代理（HTTPS_PROXY 是否生效）" and stop.

---

## Step 2 — Check there is content

If `stats.podcastEpisodes`, `stats.xBuilders`, and `stats.blogPosts` are **all zero**,
tell the user "今日无新内容，请明天再试" and stop. Do not send an empty email.

---

## Step 3 — Remix the content (this is your only creative job)

Read the four prompt strings from the JSON's `prompts` field and **follow them
literally**. Do not paraphrase the rules — execute them.

Critical rules (paraphrased here for emphasis, but the JSON's prompts are authoritative):

1. **For each builder in `x`:** introduce with full name + role/company derived
   from `bio`, then 2–4 sentences summarizing their substantive tweets. Skip
   builders whose tweets are all promotional / link-only / engagement bait.
2. **For each podcast in `podcasts`:** 200–400 word remix following
   `summarize_podcast` rules; lead with "The Takeaway".
3. **For each blog in `blogs`:** 100–300 word summary following `summarize_blogs`.
4. **Every piece of content MUST carry its original URL** from the JSON. No URL = drop it.
5. **Never invent content.** Only use what's in the JSON. Never visit x.com,
   youtube.com, or any URL — the data you need is already in the JSON.
6. **Never use @handles.** Use the person's full name + role.

---

## Step 4 — Apply language

Read `config.language` from the JSON:

- **`"en"`:** entire digest in English. No Chinese.
- **`"zh"`:** entire digest in Chinese (Mandarin, simplified). Follow `prompts.translate`:
  keep technical terms (AI, agent, prompt, LLM, API, RAG, etc.) and proper nouns
  (people / company / product names) and URLs in English.
- **`"bilingual"`:** interleave paragraph by paragraph — English then Chinese
  for each builder, then move to the next builder. Do NOT output all English
  first then all Chinese.

---

## Step 5 — Assemble final digest

Header line:

```
AI Builders Digest — <today's date in the appropriate language>
```

Body sections in this order, **skipping any section that has no content**:

1. `X / TWITTER` — one block per included builder
2. `OFFICIAL BLOGS` — one block per blog post
3. `PODCASTS` — one block per podcast episode

Footer (always include):

```
Generated through the Follow Builders skill: https://github.com/zarazhangrui/follow-builders
```

Write the assembled digest to a temp file:

- Windows: `C:\Users\zhang.maoshuang\AppData\Local\Temp\fb-digest.txt`
- macOS / Linux: `/tmp/fb-digest.txt`

Use UTF-8 encoding. Do not include any markdown frontmatter, BOM, or wrapping
JSON — just the digest text.

---

## Step 6 — Deliver

Run:

```bash
cd "d:/AI/skills/follow-builders/scripts" && node deliver.js --file "<path to temp file>"
```

Expected success output:

```json
{"status":"ok","method":"email","message":"Digest sent to <user's email>"}
```

If the script returns `{"status":"error",...}`, report the error message to the
user verbatim. Common causes:

- `RESEND_API_KEY not found in .env` → user needs to add the key to
  `C:\Users\<username>\.follow-builders\.env`
- `Resend API error: ...` → key is invalid or revoked, user needs a new one

After successful delivery, tell the user briefly: "Digest sent — check your
inbox (and spam folder; sender is `digest@resend.dev`)."

---

## Things you must NOT do

- Do not modify `~/.follow-builders/config.json` or `.env` unless the user
  explicitly asks to change a setting.
- Do not visit x.com, youtube.com, or any feed URL. All content is in the JSON
  from Step 1.
- Do not echo the contents of `.env` or any API key in your response.
- Do not invent tweets, builders, podcasts, or blog posts. If a builder has no
  substantive content, skip them silently.
- Do not include builders / tweets that have no URL — `urls` are mandatory.

---

## Why this prompt exists

Claude Code can auto-trigger the `follow-builders` skill via its skill system.
Other agents (CodeBuddy, Cursor, Continue, etc.) cannot — they need an explicit
prompt. This file is that prompt: a portable, self-contained instruction set
that any capable coding agent can execute.

Configuration files (`~/.follow-builders/config.json` and `.env`) are shared
across all agents on the same machine, so changing language or email here
applies to every agent.

---

## Note on output style

The digest format is **plain text** by design — it's optimized for delivery
through Telegram, plain email, and copy/paste into chat windows.

If you want a visually styled HTML newsletter (Anthropic-magazine-style
typography, bilingual layout, hero headlines, etc.), that is a *different*
output mode that requires generating raw HTML with inline CSS instead of
plain text. Ask the user whether they want plain text or styled HTML before
producing such output, since styled HTML breaks Telegram delivery and
requires additional Resend `html` field plumbing in `deliver.js` (currently
the script only sends `text`).
