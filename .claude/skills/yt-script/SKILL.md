---
name: yt-script
description: Generate a full YouTube/Short/TikTok content package for the Drunk Cowboy channel (CowboyWithReceipts). Produces titles, hook, outline, full script, shot list, and Shorts repurposing ideas saved to /videos/<slug>/. Use when the user wants to write a script for Ryan Walker's channel. Inputs: topic, length (minutes), platform (youtube-longform, short, tiktok).
---

# yt-script — Drunk Cowboy / CowboyWithReceipts

Generate a production-ready content package for **Ryan Walker's** YouTube channel. Every file lands in `/videos/<slug>/`.

## Channel Brand

| Property | Value |
|---|---|
| Channel | Drunk Cowboy / CowboyWithReceipts |
| Host | Ryan Walker — Black man in a cowboy hat |
| Tagline | "They bring stats. I bring context." |
| Voice | Authoritative, unapologetic, non-partisan, data-driven |
| Source standard | Federal only: FBI UCR, US Sentencing Commission, National Registry of Exonerations, BJS, USSC |
| Receipt rule | **Never say it without the receipt.** Every claim = `[SOURCE]`. Every on-screen doc = `[ON SCREEN RECEIPT]`. |
| Title rule | Truth-forward. No clickbait. No all-caps single words. Max 70 chars. |

---

## Inputs

Parse these from the skill `args` string. Use these defaults if not provided:

| Arg | Default | Notes |
|---|---|---|
| `topic` | *(required)* | The subject of the video |
| `length` | `10` | Target runtime in minutes |
| `platform` | `youtube-longform` | One of: `youtube-longform`, `short`, `tiktok` |

**Parsing examples:**
- `topic="wrongful convictions by race" length=12 platform=youtube-longform`
- `topic=police reform length=8`
- `topic="sentencing disparities" platform=short`

---

## Platform Specs

| Platform | Approx WPM | Target script length |
|---|---|---|
| `youtube-longform` | 130 wpm | `length × 130` words |
| `short` | 150 wpm | 150–225 words (60–90 seconds) |
| `tiktok` | 150 wpm | 40–150 words (15–60 seconds) |

---

## Workflow

Make a todo list and work through each step in order. Complete each step before moving to the next.

### Step 1 — Parse & Slug

- Extract `topic`, `length`, `platform` from the args string.
- Generate `<slug>`: lowercase the topic, replace spaces and special characters with hyphens, strip punctuation.
  - Example: `"Wrongful Convictions by Race"` → `wrongful-convictions-by-race`

### Step 2 — Create Output Directory

```bash
mkdir -p /videos/<slug>
```

### Step 3 — Generate `titles.txt`

Write **10 title options** for the video. Save to `/videos/<slug>/titles.txt`.

**Rules:**
- Max 70 characters each
- Truth-forward — the title is accurate, not misleading
- No clickbait phrasing, no manufactured surprise, no all-caps shock words
- Optimized for search intent + CTR: lead with the data or the finding
- Numbered list, one title per line
- Vary the angle: some lead with the stat, some lead with the question, some name the system

**Format:**
```
1. [title]
2. [title]
...
10. [title]
```

### Step 4 — Generate `hook.txt`

Write the **opening 0–30 seconds** of the video. Save to `/videos/<slug>/hook.txt`.

**Rules:**
- Word-for-word, direct to camera — Ryan is speaking, not narrating
- No music cue, no cold open b-roll — this is a straight-to-face open
- Ryan's hook formula:
  1. **Drop the number or the finding** — lead with the receipt, not the question
  2. **Name what "they" say** — one sentence on the mainstream/popular claim
  3. **The pivot** — "But here's what the data actually shows." or similar
  4. **Tease the receipt** — tell the viewer exactly what document you have
- Use `[BEAT]` for a deliberate rhetorical pause (1–2 seconds)
- Use `[PAUSE]` for a shorter natural pause (half a second)
- Include a timestamp header: `[0:00 – 0:30 | HOOK]`

### Step 5 — Generate `outline.txt`

Write a **3-act structure** with timestamp ranges. Save to `/videos/<slug>/outline.txt`.

Calculate timestamps based on `length` (in minutes):
- **Act 1 — Setup:** `0:00` to `~{length × 0.25}` min
- **Act 2 — Evidence:** `~{length × 0.25}` to `~{length × 0.75}` min
- **Act 3 — Context & Verdict:** `~{length × 0.75}` to `{length}:00`

**Format for each act:**
```
ACT [N] — [TITLE] ([start] – [end])
  Beat 1: [description] (~[duration])
  Beat 2: [description] (~[duration])
  ...
```

Include 3–5 beats per act. Each beat is a distinct section of argument, evidence, or narrative.

### Step 6 — Generate `script.txt`

Write the **full word-for-word script**. Save to `/videos/<slug>/script.txt`.

**Voice — Ryan Walker:**
- Speaks directly to camera. First person. Present tense.
- Authoritative, not aggressive. Unapologetic, not inflammatory.
- Non-partisan — the data is the data, regardless of who it implicates.
- Data-driven — never editorialize beyond what the numbers support.
- Uses plain English. No jargon unless defined immediately after.
- Rhetorical tools: repetition, contrast, direct address ("Look at this number.")
- Does not hedge claims. Does not say "some might argue." States what the data shows.

**Required markers — use these exactly:**
- `[SOURCE: <agency or dataset name>]` — after every factual claim
- `[ON SCREEN RECEIPT: <description of what document/graphic appears>]` — whenever a document, table, chart, or screenshot is shown on screen
- `[BEAT]` — deliberate rhetorical pause, 1–2 seconds
- `[PAUSE]` — shorter natural pause, half a second

**Script format:**
```
[TIMESTAMP | SECTION NAME]

[Script text with markers inline...]

[SOURCE: FBI Uniform Crime Report 2022]
[ON SCREEN RECEIPT: FBI UCR Table 43 — Arrests by Race]
[BEAT]
```

Target the word count for the platform (see Platform Specs above). For `youtube-longform`, this is a full script — do not summarize or abbreviate.

**Source disciplines:**
- Every numerical claim cites a federal source
- Every percentage has a denominator explained in the script
- Every comparison names both sides of the comparison
- If a claim cannot be sourced to a federal dataset, do not include it

### Step 7 — Generate `shotlist.txt`

Write the production shot list. Save to `/videos/<slug>/shotlist.txt`.

**Four sections:**

#### A-Roll
Direct-to-camera segments. List each timestamp range and what Ryan is doing/saying at that moment.
```
[0:00 – 0:30] Ryan on camera — hook delivery. Cowboy hat. Eye contact. No notes.
```

#### B-Roll
Suggested footage for cutting away from the talking head. Be specific.
```
[3:15 – 3:45] B-roll: exterior shot of federal courthouse / stock courtroom footage
```

#### Receipt Inserts
Every `[ON SCREEN RECEIPT]` in the script gets a line here with the timestamp and the specific document.
```
[4:00] INSERT: FBI UCR 2022, Table 43. Zoom to "Black or African American" arrest row.
```

#### Platform Screenshots Needed
List specific government websites, databases, or documents that need to be captured on screen (URL, what to show, when).
```
[5:30] SCREENSHOT: bjs.gov — Prisoners in 2022 report, Table 2. Show full table.
```

### Step 8 — Generate `shorts.txt`

Write **3–5 Short/TikTok repurposing ideas** pulled directly from the main script. Save to `/videos/<slug>/shorts.txt`.

**For each idea:**
```
SHORT [N]: [Working title for the Short]
Timestamp range: [start] – [end]
Why it works standalone: [1–2 sentences]
Hook for the Short: [first line Ryan says, word for word]
Platform fit: [YouTube Short / TikTok / both]
```

Each Short should:
- Have a self-contained argument or finding (viewer doesn't need to have seen the full video)
- Open with a hook strong enough to stop a scroll
- Be completable in under 60 seconds (under 90 for YouTube Shorts)

### Step 9 — Write All Files

Use the Write tool to save each file to `/videos/<slug>/`. Confirm each file is written before proceeding.

Files to write:
- `/videos/<slug>/titles.txt`
- `/videos/<slug>/hook.txt`
- `/videos/<slug>/outline.txt`
- `/videos/<slug>/script.txt`
- `/videos/<slug>/shotlist.txt`
- `/videos/<slug>/shorts.txt`

### Step 10 — Wrap Up

Report to the user:

```
Content package ready: /videos/<slug>/

Files:
  titles.txt   — 10 title options
  hook.txt     — 0:00–0:30 hook
  outline.txt  — 3-act structure
  script.txt   — ~[word count] words (~[estimated runtime] min at [wpm] wpm)
  shotlist.txt — [N] A-roll beats, [N] receipt inserts, [N] B-roll suggestions
  shorts.txt   — [N] Short ideas

Receipts required (pull these before shoot):
  [List every [SOURCE] cited in the script, deduplicated]
```

---

## Quality Checks

Before writing the files, verify:
- [ ] Every factual claim in `script.txt` has a `[SOURCE]` tag
- [ ] Every `[ON SCREEN RECEIPT]` in `script.txt` has a matching entry in `shotlist.txt`
- [ ] All titles in `titles.txt` are ≤ 70 characters
- [ ] Act timestamps in `outline.txt` sum to approximately `length` minutes
- [ ] `shorts.txt` has 3–5 entries with timestamp ranges that exist in the script
- [ ] No claims are made that cannot be attributed to FBI UCR, US Sentencing Commission, National Registry of Exonerations, BJS, or USSC
