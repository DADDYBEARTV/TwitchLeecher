---
name: yt-thumb
description: Generate 4 thumbnail mockup PNGs and a concepts.md for the Drunk Cowboy channel (CowboyWithReceipts). Scrapes reference YouTube thumbnails, analyzes what is working, then applies a visual stun gun framework to produce black/gold/white mockups. Use when the user wants thumbnail options for Ryan Walker's videos. Inputs: video title, 3-5 reference YouTube URLs, output folder path (optional).
---

# yt-thumb — Drunk Cowboy / CowboyWithReceipts

Generate 4 thumbnail mockup PNGs that stop the scroll at 320×180 mobile size.
All outputs save to `/thumbnails/<slug>/`.

---

## Channel Brand

| Property | Value |
|---|---|
| Channel | Drunk Cowboy / CowboyWithReceipts |
| Host | Ryan Walker — Black man in a cowboy hat |
| Background | Black (`#000000`) |
| Primary accent | Gold (`#c8a84b`) |
| Text color | White (`#ffffff`) |
| Text rule | Thumbnail text **complements** the title — never repeats it. Max 3–4 words. |
| Hat rule | Cowboy hat must be visible in every layout. Reserve headshot space accordingly. |
| Size target | Designed at 1280×720, must read clearly at 320×180 |

---

## Inputs

Parse from skill `args`:

| Arg | Default | Notes |
|---|---|---|
| `title` | *(required)* | The video title |
| `urls` | *(required)* | 3–5 YouTube URLs, space or comma separated |
| `output` | `/thumbnails/<slug>/` | Output folder path |

**Parsing examples:**
- `title="Wrongful Convictions by Race" urls="https://youtu.be/abc123 https://youtu.be/xyz789"`
- `title="Sentencing Disparities" urls=https://youtube.com/watch?v=abc,https://youtu.be/def output=/thumbnails/custom-folder/`

---

## Workflow

Make a todo list and work through each step in order.

---

### Step 1 — Parse, Slug, Setup

1. Extract `title`, `urls` (split on spaces and commas), `output` from args.
2. Generate `<slug>`: lowercase title, spaces → hyphens, strip punctuation.
3. Set output directory to `/thumbnails/<slug>/` unless `output` was provided.
4. Run:
   ```bash
   mkdir -p <output>/references
   pip install Pillow --quiet
   ```

---

### Step 2 — Download Reference Thumbnails

For each YouTube URL, extract the video ID and download the highest-quality thumbnail available.

**Video ID extraction rules:**
- `youtube.com/watch?v=<ID>` → ID is the `v` query param
- `youtu.be/<ID>` → ID is the path segment
- `youtube.com/shorts/<ID>` → ID is the path segment
- `youtube.com/embed/<ID>` → ID is the path segment

**Download logic (try in order, stop at first success):**

Write and run this Python script as `/tmp/fetch_thumbs.py`:

```python
import requests
import sys
import os
from urllib.parse import urlparse, parse_qs

def extract_video_id(url):
    parsed = urlparse(url)
    if parsed.netloc in ('youtu.be',):
        return parsed.path.lstrip('/')
    qs = parse_qs(parsed.query)
    if 'v' in qs:
        return qs['v'][0]
    parts = parsed.path.split('/')
    for keyword in ('shorts', 'embed', 'v'):
        if keyword in parts:
            idx = parts.index(keyword)
            if idx + 1 < len(parts):
                return parts[idx + 1]
    return None

def download_thumbnail(video_id, dest_dir):
    qualities = ['maxresdefault', 'sddefault', 'hqdefault', 'mqdefault']
    for q in qualities:
        url = f"https://img.youtube.com/vi/{video_id}/{q}.jpg"
        r = requests.get(url, timeout=10)
        if r.status_code == 200 and len(r.content) > 5000:
            path = os.path.join(dest_dir, f"{video_id}_{q}.jpg")
            with open(path, 'wb') as f:
                f.write(r.content)
            print(f"OK  {video_id} → {q}.jpg")
            return path
    print(f"FAIL  {video_id} — no thumbnail found")
    return None

urls = sys.argv[1:]
dest = sys.argv[0] if os.path.isdir(sys.argv[0]) else '.'

# Called as: python3 fetch_thumbs.py <dest_dir> <url1> <url2> ...
dest_dir = sys.argv[1]
urls = sys.argv[2:]

results = []
for url in urls:
    vid = extract_video_id(url.strip())
    if vid:
        path = download_thumbnail(vid, dest_dir)
        if path:
            results.append((vid, path))
    else:
        print(f"SKIP  Could not parse video ID from: {url}")

print(f"\nDownloaded {len(results)} thumbnails to {dest_dir}")
```

Run it:
```bash
python3 /tmp/fetch_thumbs.py <output>/references <url1> <url2> ...
```

---

### Step 3 — Analyze Reference Thumbnails

Write and run this Python analysis script as `/tmp/analyze_thumbs.py`:

```python
#!/usr/bin/env python3
"""Analyze reference thumbnails for color, composition, and text patterns."""
import sys
import os
from PIL import Image
import colorsys

def get_dominant_color_zone(img, zone):
    """Sample colors from a zone: 'left', 'right', 'top', 'bottom', 'center'."""
    w, h = img.size
    zones = {
        'left':   (0,     0,     w//2,  h),
        'right':  (w//2,  0,     w,     h),
        'top':    (0,     0,     w,     h//2),
        'bottom': (0,     h//2,  w,     h),
        'center': (w//4,  h//4,  3*w//4, 3*h//4),
    }
    box = zones[zone]
    region = img.crop(box).resize((50, 30), Image.LANCZOS)
    pixels = list(region.getdata())
    r = sum(p[0] for p in pixels) // len(pixels)
    g = sum(p[1] for p in pixels) // len(pixels)
    b = sum(p[2] for p in pixels) // len(pixels)
    return (r, g, b)

def brightness(rgb):
    return (rgb[0] * 299 + rgb[1] * 587 + rgb[2] * 114) / 1000

def analyze(path):
    img = Image.open(path).convert('RGB')
    w, h = img.size

    # Sample brightness per zone
    zones = ['left', 'right', 'top', 'bottom', 'center']
    zone_brightness = {z: brightness(get_dominant_color_zone(img, z)) for z in zones}

    # Guess where text likely lives (darkest zone = likely text bg, or lightest = contrast text)
    sorted_zones = sorted(zone_brightness.items(), key=lambda x: x[1])
    darkest_zone = sorted_zones[0][0]
    brightest_zone = sorted_zones[-1][0]

    # Edge contrast (thumbnail impact at small size)
    corners = [img.getpixel((0,0)), img.getpixel((w-1,0)),
               img.getpixel((0,h-1)), img.getpixel((w-1,h-1))]
    corner_brightness = [brightness(c) for c in corners]
    edge_contrast = max(corner_brightness) - min(corner_brightness)

    # Small-size legibility test: downscale to 320x180
    small = img.resize((320, 180), Image.LANCZOS)
    small_pixels = list(small.getdata())
    small_brightness_vals = [brightness(p) for p in small_pixels]
    small_contrast = max(small_brightness_vals) - min(small_brightness_vals)

    return {
        'file': os.path.basename(path),
        'size': f"{w}x{h}",
        'darkest_zone': darkest_zone,
        'brightest_zone': brightest_zone,
        'edge_contrast': round(edge_contrast, 1),
        'small_size_contrast': round(small_contrast, 1),
        'zone_brightness': {k: round(v, 1) for k, v in zone_brightness.items()},
    }

paths = sys.argv[1:]
print("=== REFERENCE THUMBNAIL ANALYSIS ===\n")
for path in paths:
    if os.path.exists(path):
        result = analyze(path)
        print(f"File: {result['file']} ({result['size']})")
        print(f"  Darkest zone (likely text bg or face shadow): {result['darkest_zone']}")
        print(f"  Brightest zone (likely face/text/highlight):  {result['brightest_zone']}")
        print(f"  Edge contrast score: {result['edge_contrast']}/255")
        print(f"  Small-size (320x180) contrast: {result['small_size_contrast']}/255")
        print(f"  Zone brightness map: {result['zone_brightness']}")
        print()
```

Run it:
```bash
python3 /tmp/analyze_thumbs.py <output>/references/*.jpg
```

Read the output carefully. For each thumbnail, note:
- Which zone is darkest / where text likely lives
- Edge contrast score (higher = more stopping power)
- Small-size contrast (must be high for mobile legibility)
- Any pattern across references (e.g., "all have bright left, dark right")

Summarize the patterns you observed — you'll use this in Step 8 (concepts.md).

---

### Step 4 — Derive Thumbnail Text

From the video `title`, generate 4 short text options for the thumbnails. Rules:
- Max 3–4 words each
- Must **complement** the title, not repeat it
- Should create curiosity or name the stakes
- All-caps works for 1–2 word options; title case for 3–4 words
- Avoid: question marks that feel hollow, numbers that duplicate what's in the title

Write these down — you'll use them as the text in each layout.

---

### Step 5 — Generate Thumbnail Mockups

Write the following Python script to `/tmp/gen_thumbs.py`, then run it. The script generates 4 PNG files.

```python
#!/usr/bin/env python3
"""Generate 4 Drunk Cowboy thumbnail mockups."""
import sys
import os
from PIL import Image, ImageDraw, ImageFont

# ── Constants ────────────────────────────────────────────────────────────────
W, H = 1280, 720
BLACK  = (0, 0, 0)
GOLD   = (200, 168, 75)    # #c8a84b
WHITE  = (255, 255, 255)
DARK_GOLD = (140, 110, 35) # deeper gold for shadows/borders
HEADSHOT_BG = (18, 18, 18) # near-black for headshot placeholder area

FONT_PATHS = [
    "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf",
    "/usr/share/fonts/truetype/liberation/LiberationSans-Bold.ttf",
    "/usr/share/fonts/truetype/freefont/FreeSansBold.ttf",
]

def get_font(size):
    for p in FONT_PATHS:
        if os.path.exists(p):
            return ImageFont.truetype(p, size)
    return ImageFont.load_default()

def draw_text_with_stroke(draw, xy, text, font, fill, stroke_color, stroke_width=6):
    """Draw text with a thick stroke for maximum contrast."""
    x, y = xy
    for dx in range(-stroke_width, stroke_width + 1):
        for dy in range(-stroke_width, stroke_width + 1):
            if dx != 0 or dy != 0:
                draw.text((x + dx, y + dy), text, font=font, fill=stroke_color)
    draw.text((x, y), text, font=font, fill=fill)

def draw_headshot_placeholder(draw, box, label="HEADSHOT + HAT"):
    """Draw a labeled placeholder rectangle for the headshot cutout."""
    x1, y1, x2, y2 = box
    draw.rectangle(box, fill=HEADSHOT_BG)
    # Dashed gold border
    border = 4
    draw.rectangle([x1+border, y1+border, x2-border, y2-border],
                   outline=GOLD, width=3)
    # Cowboy hat silhouette (simple geometric suggestion)
    cx = (x1 + x2) // 2
    brim_y = y1 + (y2 - y1) // 3
    crown_y = brim_y - 120
    brim_w = (x2 - x1) // 2
    # Hat brim
    draw.ellipse([cx - brim_w, brim_y - 18, cx + brim_w, brim_y + 18],
                 fill=GOLD)
    # Hat crown
    draw.rounded_rectangle([cx - brim_w//2, crown_y, cx + brim_w//2, brim_y + 10],
                            radius=20, fill=GOLD)
    # Label
    font_sm = get_font(28)
    draw.text((cx, y2 - 60), label, font=font_sm, fill=WHITE, anchor="mm")
    draw.text((cx, y2 - 28), "CowboyWithReceipts", font=font_sm, fill=GOLD, anchor="mm")

def add_gold_bar(draw, orientation, position, length, thickness=10):
    """Add a gold accent bar. orientation: 'h' or 'v'."""
    if orientation == 'h':
        x, y = position
        draw.rectangle([x, y, x + length, y + thickness], fill=GOLD)
    else:
        x, y = position
        draw.rectangle([x, y, x + thickness, y + length], fill=GOLD)

def add_tagline(draw, y_pos):
    font = get_font(32)
    text = "CowboyWithReceipts"
    draw.text((40, y_pos), text, font=font, fill=GOLD)

# ── Parse args ───────────────────────────────────────────────────────────────
# Usage: python3 gen_thumbs.py <output_dir> "TEXT LINE 1" "TEXT LINE 2 (optional)" "TEXT LINE 3 (optional)"
output_dir = sys.argv[1]
text_lines = sys.argv[2:]  # up to 3 lines of text (each 1-2 words)

# Fallback text if not provided
if not text_lines:
    text_lines = ["THE", "RECEIPT", "IS HERE"]

# For layouts: use all lines; wrap into 2 chunks for large-text layouts
def render_text_block(draw, lines, start_xy, font_size, max_width=None):
    """Render stacked text lines, return final y position."""
    font = get_font(font_size)
    x, y = start_xy
    line_gap = int(font_size * 1.1)
    for line in lines:
        draw_text_with_stroke(draw, (x, y), line.upper(), font, WHITE, BLACK, stroke_width=8)
        y += line_gap
    return y

# ── LAYOUT A: Text Left, Headshot Right ──────────────────────────────────────
def layout_a(text_lines, output_path):
    img = Image.new('RGB', (W, H), BLACK)
    draw = ImageDraw.Draw(img)

    # Vertical gold bar on far left
    add_gold_bar(draw, 'v', (0, 0), H, thickness=12)

    # Headshot placeholder: right 44%
    headshot_box = (720, 0, W, H)
    draw_headshot_placeholder(draw, headshot_box)

    # Text block: left zone, vertically centered
    text_start_y = H // 2 - (len(text_lines) * 130) // 2
    final_y = render_text_block(draw, text_lines, (40, text_start_y), font_size=160)

    # Gold underline beneath text
    add_gold_bar(draw, 'h', (40, final_y + 10), 600, thickness=8)

    # Tagline bottom-left
    add_tagline(draw, H - 60)

    img.save(output_path)
    print(f"Saved: {output_path}")

# ── LAYOUT B: Headshot Left, Text Right ──────────────────────────────────────
def layout_b(text_lines, output_path):
    img = Image.new('RGB', (W, H), BLACK)
    draw = ImageDraw.Draw(img)

    # Headshot placeholder: left 44%
    headshot_box = (0, 0, 560, H)
    draw_headshot_placeholder(draw, headshot_box)

    # Vertical gold divider
    add_gold_bar(draw, 'v', (560, 0), H, thickness=10)

    # Text block: right zone, vertically centered
    text_start_y = H // 2 - (len(text_lines) * 130) // 2
    final_y = render_text_block(draw, text_lines, (600, text_start_y), font_size=160)

    # Gold underline
    add_gold_bar(draw, 'h', (600, final_y + 10), 640, thickness=8)

    # Tagline bottom-right
    font = get_font(32)
    draw.text((W - 40, H - 40), "CowboyWithReceipts", font=font, fill=GOLD, anchor="ra")

    img.save(output_path)
    print(f"Saved: {output_path}")

# ── LAYOUT C: Text Bottom, Headshot Top/Full ─────────────────────────────────
def layout_c(text_lines, output_path):
    img = Image.new('RGB', (W, H), BLACK)
    draw = ImageDraw.Draw(img)

    # Headshot placeholder: top 58%
    headshot_box = (0, 0, W, 420)
    draw_headshot_placeholder(draw, headshot_box, label="HEADSHOT — HAT UP HIGH")

    # Gold band separating headshot from text area
    add_gold_bar(draw, 'h', (0, 420), W, thickness=10)

    # Text block: bottom area, centered
    text_area_center_x = W // 2
    text_start_y = 445
    font_size = 140
    font = get_font(font_size)
    y = text_start_y
    for line in text_lines:
        bbox = draw.textbbox((0, 0), line.upper(), font=font)
        tw = bbox[2] - bbox[0]
        draw_text_with_stroke(draw, (text_area_center_x - tw // 2, y),
                               line.upper(), font, WHITE, BLACK, stroke_width=8)
        y += int(font_size * 1.05)

    # Gold underline
    add_gold_bar(draw, 'h', (W // 4, y + 8), W // 2, thickness=6)

    # Tagline
    add_tagline(draw, H - 44)

    img.save(output_path)
    print(f"Saved: {output_path}")

# ── LAYOUT D: Diagonal Split ─────────────────────────────────────────────────
def layout_d(text_lines, output_path):
    img = Image.new('RGB', (W, H), BLACK)
    draw = ImageDraw.Draw(img)

    # Diagonal divider: polygon covers left-black, right-dark area
    # The split runs from top ~55% to bottom ~40%
    split_top_x = int(W * 0.55)
    split_bot_x = int(W * 0.40)

    # Right region: slightly lighter for headshot contrast
    right_poly = [(split_top_x, 0), (W, 0), (W, H), (split_bot_x, H)]
    draw.polygon(right_poly, fill=HEADSHOT_BG)

    # Gold diagonal bar (the split line itself, as a thin polygon)
    bar_w = 14
    bar_poly = [
        (split_top_x,        0),
        (split_top_x + bar_w, 0),
        (split_bot_x + bar_w, H),
        (split_bot_x,        H),
    ]
    draw.polygon(bar_poly, fill=GOLD)

    # Headshot placeholder: right of split
    hs_cx = (split_top_x + bar_w + W) // 2
    hs_w = W - split_top_x - bar_w - 20
    hs_box = (split_top_x + bar_w + 20, 20, W - 20, H - 20)
    # Draw hat silhouette hint in right zone
    draw.text((hs_cx, H // 2), "HEADSHOT\n+ HAT", font=get_font(36),
              fill=GOLD, anchor="mm", align="center")
    draw.rectangle(hs_box, outline=GOLD, width=3)

    # Text block: left of split, vertically centered
    text_start_y = H // 2 - (len(text_lines) * 120) // 2
    font_size = 148
    font = get_font(font_size)
    max_x = split_bot_x - 30
    y = text_start_y
    for line in text_lines:
        bbox = draw.textbbox((0, 0), line.upper(), font=font)
        tw = bbox[2] - bbox[0]
        # If text too wide, reduce font
        fs = font_size
        while tw > max_x - 30 and fs > 60:
            fs -= 8
            font = get_font(fs)
            bbox = draw.textbbox((0, 0), line.upper(), font=font)
            tw = bbox[2] - bbox[0]
        draw_text_with_stroke(draw, (30, y), line.upper(), font, WHITE, BLACK, stroke_width=8)
        y += int(fs * 1.1)
        font = get_font(font_size)  # reset for next line

    # Gold accent bar left edge
    add_gold_bar(draw, 'v', (0, 0), H, thickness=10)

    # Tagline
    add_tagline(draw, H - 52)

    img.save(output_path)
    print(f"Saved: {output_path}")

# ── Run all layouts ───────────────────────────────────────────────────────────
layout_a(text_lines, os.path.join(output_dir, "thumb_A_text-left.png"))
layout_b(text_lines, os.path.join(output_dir, "thumb_B_text-right.png"))
layout_c(text_lines, os.path.join(output_dir, "thumb_C_text-bottom.png"))
layout_d(text_lines, os.path.join(output_dir, "thumb_D_diagonal-split.png"))

print("\nAll 4 thumbnail mockups generated.")
```

**Before running**, determine the best 2–3 word text chunks from Step 4, then run:

```bash
python3 /tmp/gen_thumbs.py <output_dir> "WORD ONE" "WORD TWO" "WORD THREE"
```

Each arg becomes one stacked line on the thumbnail. Use 2–3 args for best results.

---

### Step 6 — Verify Output Files

Confirm all 4 PNGs were created:

```bash
ls -lh <output>/thumb_*.png
```

If any file is missing or has 0 bytes, debug the Python script and re-run.

Optionally verify pixel dimensions:

```bash
python3 -c "
from PIL import Image
import glob
for f in sorted(glob.glob('<output>/thumb_*.png')):
    img = Image.open(f)
    print(f, img.size)
"
```

All should be 1280×720.

---

### Step 7 — Mobile Legibility Check

Run this quick contrast check to verify each thumbnail reads at 320×180:

```python
#!/usr/bin/env python3
import sys
from PIL import Image

def check_legibility(path):
    img = Image.open(path).convert('RGB').resize((320, 180), Image.LANCZOS)
    pixels = list(img.getdata())
    brightness_vals = [(r*299 + g*587 + b*114)//1000 for r, g, b in pixels]
    contrast = max(brightness_vals) - min(brightness_vals)
    avg = sum(brightness_vals) // len(brightness_vals)
    score = "PASS ✓" if contrast >= 150 else "WARN ⚠"
    print(f"{score}  {path.split('/')[-1]}  contrast={contrast}/255  avg_brightness={avg}/255")

for path in sys.argv[1:]:
    check_legibility(path)
```

Run:
```bash
python3 /tmp/check_legibility.py <output>/thumb_*.png
```

If any file scores below 150 contrast, note it in concepts.md and suggest a fix.

---

### Step 8 — Generate `concepts.md`

Write `/thumbnails/<slug>/concepts.md` using the Write tool.

**Structure:**

```markdown
# Thumbnail Concepts — [Video Title]
**Channel:** Drunk Cowboy / CowboyWithReceipts
**Generated:** [date]

---

## Reference Analysis

### What the competition is doing:
[Summarize patterns found in Step 3 — dominant zones, text placement, contrast scores]

### What's working (steal this):
- [Pattern 1]
- [Pattern 2]
- [Pattern 3]

### What's missing (your edge):
- [Gap in what competitors do that Drunk Cowboy can own]

---

## Visual Stun Gun Framework
At 320×180, a thumbnail has ~0.3 seconds to register. Every mockup passes 3 tests:
1. **Face test** — Is Ryan visible and emoting? (Headshot placeholder marks the zone)
2. **Text test** — Can you read the text in 0.3 seconds without zooming?
3. **Color test** — Does the black/gold/white palette snap against typical feed backgrounds?

---

## Option A — Text Left / Headshot Right
**File:** `thumb_A_text-left.png`
**Layout:** Text occupies left 56%. Headshot fills right 44%. Vertical gold bar anchors left edge.
**Why it pulls clicks:** [Specific reasoning — what the text position does for eye flow, why the gold bar creates a frame]
**Who this beats:** [Type of competing thumbnail this outperforms and why]
**Suggested tweaks:**
- [ ] Replace placeholder with Ryan's face at eye level, hat crown above frame edge
- [ ] If text reads short at small size: increase font weight or reduce to 2 words
- [ ] Test: gold bar on left vs. gold bar framing text top+bottom

---

## Option B — Headshot Left / Text Right
**File:** `thumb_B_text-right.png`
**Layout:** Headshot left 44%. Vertical gold divider at 560px. Text right 56%.
**Why it pulls clicks:** [Reasoning — eye typically enters from left, Ryan's face is the entry point, text is the destination]
**Who this beats:** [Competing pattern]
**Suggested tweaks:**
- [ ] Ryan should be looking toward the text (into-frame gaze direction)
- [ ] Consider gold-colored text for the most punchy word, white for the rest
- [ ] Tagline opacity: 80% so it reads but doesn't compete

---

## Option C — Text Bottom / Headshot Top
**File:** `thumb_C_text-bottom.png`
**Layout:** Headshot fills top 58%. Gold band at 420px. Text in bottom third, centered.
**Why it pulls clicks:** [Reasoning — establishes Ryan's authority visually before delivering the claim]
**Who this beats:** [Competing pattern]
**Suggested tweaks:**
- [ ] Ryan's cowboy hat should break the gold band line — hat above, chin below
- [ ] Bottom text: if 3 lines, consider dropping to 2 for bigger font size
- [ ] Gold band: can thicken to 20px if it needs more visual separation

---

## Option D — Diagonal Split
**File:** `thumb_D_diagonal-split.png`
**Layout:** Diagonal gold bar splits frame. Black/text zone left. Headshot zone right.
**Why it pulls clicks:** [Reasoning — diagonal is the most kinetic layout, creates visual tension, breaks grid expectations]
**Who this beats:** [Competing pattern]
**Suggested tweaks:**
- [ ] Diagonal angle: steeper (more dramatic) vs. shallower (more stable) — test both
- [ ] Ryan positioned so hat crosses the gold bar = visual continuity
- [ ] Risk: busiest layout. If text isn't crisp, simplify to 2 words

---

## Recommended Pick
**Option [X]** — [One sentence on why this is the strongest choice given the topic and what competitors do]

**Second choice:** Option [Y] for A/B testing.

---

## Pre-Production Checklist
- [ ] Export Ryan's headshot cutout with hat (PNG with transparent bg)
- [ ] Confirm hat is fully visible in final composite — at minimum brim + lower crown
- [ ] Final text: run past the "glance test" — cover for 0.3 sec, can you recall it?
- [ ] Check final file at 320×180 on a real phone screen before upload
- [ ] Thumbnail text does NOT repeat the title word-for-word
```

Fill in all `[bracketed]` sections based on your actual analysis from Step 3 and the mockup text you chose.

---

### Step 9 — Final Output Summary

Report to the user:

```
Thumbnail package ready: <output>/

References downloaded: [N] thumbnails → <output>/references/
Mockups generated:
  thumb_A_text-left.png      (1280×720)
  thumb_B_text-right.png     (1280×720)
  thumb_C_text-bottom.png    (1280×720)
  thumb_D_diagonal-split.png (1280×720)
concepts.md

Thumbnail text used: "[LINE 1]" / "[LINE 2]" / "[LINE 3]"

Legibility check: [PASS/WARN results from Step 7]

Next step: Drop Ryan's headshot cutout (PNG, transparent bg, cowboy hat visible)
into each mockup to replace the placeholder.
```

---

## Quality Checks

Before reporting done, verify:
- [ ] All 4 PNGs exist and are 1280×720
- [ ] All 4 PNGs pass the 320×180 legibility check (contrast ≥ 150)
- [ ] `concepts.md` exists and all `[bracketed]` placeholder text is filled in
- [ ] Thumbnail text (max 3–4 words total) complements the video title without repeating it
- [ ] Every layout has headshot space reserved with hat-visible guidance noted
- [ ] `concepts.md` includes a clear recommended pick
