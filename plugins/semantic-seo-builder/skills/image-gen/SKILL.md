---
name: image-gen
description: Context-aware AI image generation for local business websites using the Kie API (Nano Banana / Google Gemini Imagen). Reads page content and structure, identifies every section that needs an image, generates section-appropriate images, converts to WebP, and returns ready-to-place HTML with correct alt text. Works for any page type — service pages, blog articles, about sections, process sections, fabric/material sections, etc. Use when user says "generate images", "add images to this page", "create hero image", "images for my site", "/image-gen", or when weekly-pipeline or content-brief needs images after writing a page.
tools:
  - Read
  - Write
  - Bash
---

# Image Generation Skill — `/image-gen`

Context-aware AI image generation. Reads any page (HTML, markdown, or brief), maps every section that needs an image, generates the right image for each section, converts to WebP, and returns placement-ready HTML.

**This skill works for:**
- New pages written by `/content-brief` or `/weekly-pipeline`
- Existing HTML preview pages
- Any future page — service, blog, about, location, landing page
- Batch site-wide image generation at launch

---

## How context-aware image selection works

Before generating a single image, the skill reads the page and categorises every section:

| Section type | Image style | Example |
|---|---|---|
| Hero / page header | Wide cinematic, establishes the service/entity | Tailor workshop exterior, suit on dress form |
| About / story | Interior warmth, craftsmanship, human element | Workshop interior, tools, fabric detail |
| Service section | Editorial product/process, clean and focused | Close-up of specific garment type |
| Process / how-it-works | Hands at work, step in progress | Tailor measuring, pinning, cutting |
| Fabric / material | Flat lay or close-up texture, studio lighting | Fabric swatches, bolt of wool |
| Wedding / occasion | Lifestyle, aspirational, soft tones | Wedding suit on hanger, boutonniere |
| Location / find us | Exterior architecture, tropical setting | Boutique entrance, street view |
| Blog article hero | Editorial documentary, scene-setting | Context shot for the article topic |
| CTA / contact | Warm, inviting, action-oriented | Workshop door, welcoming space |
| Testimonials | Social proof context — optional | Garment detail, finished product |

---

## Step 1 — Read and map the page

Read the full page content. Extract every section with its heading and content summary.

For each section, determine:
- Does it need an image? (not all sections do — FAQ, pricing tables, CTA text usually don't)
- What type of image is appropriate?
- What is the primary entity of this section?

Output a silent image map (don't show unless asked):

```
Section: Hero
  → Image type: Wide cinematic hero
  → Entity: bespoke tailoring workshop Phuket
  → Filename: bespoke-tailoring-workshop-hero-phuket.webp

Section: About / Our Story
  → Image type: Workshop interior
  → Entity: tailor workshop interior Bangtao
  → Filename: tailor-workshop-interior-bangtao-phuket.webp

Section: Fabric Selection
  → Image type: Flat lay fabric swatches
  → Entity: premium suit fabrics wool linen
  → Filename: premium-suit-fabric-swatches-phuket.webp

Section: Process
  → Image type: Hands / process close-up
  → Entity: tailor measuring fitting suit
  → Filename: tailor-measuring-fitting-bespoke-suit.webp
```

**Sections that do NOT need images (skip):**
- Pure text FAQ sections
- Pricing tables
- Navigation, footers, breadcrumbs
- Testimonial/review quote blocks (text is the content)
- CTA buttons (unless it's a full CTA banner with background)

---

## Step 2 — Check existing images

Before generating, check `images/` directory for any existing WebP files that match a section's entity. If a suitable image already exists, reuse it rather than generating a duplicate.

```bash
ls images/*.webp 2>/dev/null || echo "No existing images"
```

---

## Step 3 — Build section-specific prompts

Use this formula for each image type:

### Hero images (wide, 1200×460–630px)
```
Wide editorial [photography style] of [primary entity + business type] in [location context].
[Lighting — warm golden light / soft window light / moody ambient].
[Setting detail — tropical workshop / dark boutique / lush greenery].
[Aesthetic — luxury fashion, cinematic, dark moody background with warm tones].
No people. No text. No watermarks.
```

### Service section images (square or 4:3, 800×600px)
```
Editorial [photography style] of [specific service entity — garment type, process step].
[Detail focus — close-up of fabric / garment / specific element].
[Lighting — warm natural / soft studio].
[Aesthetic — luxury editorial, clean background or shallow DOF].
No faces. No text.
```

### About / interior images (wide or 3:2, 1200×800px)
```
Interior editorial photography of a [business type] in [location].
[Warm / ambient / natural light]. [Setting details — tables, tools, fabrics, shelving].
[Mood — refined, crafted, welcoming, tropical luxury].
No people. No text.
```

### Process / hands images (square or 4:3)
```
Close-up editorial photography of [specific craft action — measuring, pinning, cutting, hand-stitching].
[Material detail — fabric type, tools visible].
[Lighting — warm window light, shallow DOF].
Luxury bespoke tailoring aesthetic. No faces. No text.
```

### Fabric / material images (wide flat lay)
```
Aerial editorial flat lay photography of [fabric types and colours].
[Surface — dark wooden table / marble / craft paper].
[Props — measuring tape, tailor's chalk, buttons].
Soft natural light. Top-down composition. No text.
```

### Blog article hero (wide editorial)
```
Editorial [documentary / lifestyle / product] photography of [article primary entity].
[Scene context matching article topic]. [Warm natural light].
Luxury [niche] aesthetic. No text. No faces unless specified.
```

### Location / exterior images
```
Elegant exterior photography of a [business type] boutique in a tropical setting.
[Time of day — warm evening / golden hour / soft morning light].
[Architectural details — signage, entrance, greenery].
No people. No text.
```

---

## Step 4 — Generate images (batch)

Fire ALL image jobs simultaneously, then poll. Never generate one at a time.

```python
task_ids = {}

for section_name, prompt, filename in image_queue:
    result = mcp__kie-ai-mcp-server__generate_nano_banana(
        api_key=KIE_API_KEY,
        prompt=prompt
    )
    task_ids[section_name] = {
        "task_id": result["data"]["taskId"],
        "filename": filename
    }

print(f"Fired {len(task_ids)} image jobs. Waiting 30 seconds...")
```

---

## Step 5 — Poll and download

```python
import time, requests

time.sleep(30)

for section_name, meta in task_ids.items():
    status = mcp__kie-ai-mcp-server__get_task_status(
        api_key=KIE_API_KEY,
        task_id=meta["task_id"]
    )
    state = status["api_response"]["data"]["state"]
    
    if state == "success":
        result_json = json.loads(status["api_response"]["data"]["resultJson"])
        url = result_json["resultUrls"][0]
        
        # Download PNG
        subprocess.run(["curl", "-s", "-o", f"images/{meta['filename'].replace('.webp', '.png')}", url])
        
        # Convert PNG → WebP with compression
        # Target: under 200KB per image (typically 20-150KB after this command)
        subprocess.run(["convert",
            f"images/{meta['filename'].replace('.webp', '.png')}",
            "-resize", "1600x1200>",      # max 1600px wide, never upscale
            "-quality", "78",             # NOT 85 — 85 produces 1-2MB files
            "-define", "webp:method=6",   # slower encode, better compression
            f"images/{meta['filename']}"])
        
        # Remove PNG
        os.remove(f"images/{meta['filename'].replace('.webp', '.png')}")
        
        print(f"{section_name}: saved as images/{meta['filename']}")
    else:
        print(f"{section_name}: still processing ({state}) — retry in 10s")
```

Poll every 10 seconds for any outstanding jobs until all are complete.

---

## Step 5b — Verify file sizes after compression

After all downloads and conversions are complete, always run:

```bash
ls -lh images/*.webp
```

**Acceptance criteria:**
- Every image must be under 200KB
- Hero images: target 50-150KB
- Card/section images: target 20-80KB
- If any image is over 200KB, re-compress with lower quality:

```bash
# Re-compress oversized images
for f in images/*.webp; do
  SIZE=$(du -k "$f" | cut -f1)
  if [ "$SIZE" -gt 200 ]; then
    echo "Re-compressing $f ($SIZE KB)"
    convert "$f" -resize "1200x900>" -quality 72 -define webp:method=6 "$f"
  fi
done
ls -lh images/*.webp
```

**Why this matters:** Kie API returns ~1-2MB PNG files. Without compression the page loads 10-15MB of images. The `-quality 78 -resize 1600x1200> -define webp:method=6` command reliably produces 14-180KB WebP files with no visible quality loss.

---

## Step 6 — Place images in sections

After all images are downloaded and verified under 200KB, place them using the correct pattern for each section type.

---

### Pattern A — Card grids (services, process steps, style cards)

This is the most common pattern. Any grid of cards where each card describes a service, process step, or garment type gets an image on top.

**CSS to add to the page `<style>` block** (add once per page, not per card):
```css
.card.has-img { padding: 0; }
.card.has-img img { width: 100%; aspect-ratio: 4/3; object-fit: cover; display: block; }
.card.has-img .card-body { padding: 24px 28px 28px; }
```

If the cards use a different class name (`.service-card`, `.alt-card`, `.shirt-card`, `.style-card`), add the same three rules using that class name:
```css
.service-card.has-img { padding: 0; }
.service-card.has-img img { width: 100%; aspect-ratio: 4/3; object-fit: cover; display: block; }
.service-card.has-img .card-body { padding: 24px 28px 28px; }
```

**HTML pattern — each card becomes:**
```html
<!-- BEFORE (no image): -->
<div class="card">
  <h3>Service Name</h3>
  <p>Description text.</p>
</div>

<!-- AFTER (with image): -->
<div class="card has-img">
  <img src="images/{filename}.webp"
       alt="{entity-based alt text describing this specific service}"
       width="800" height="600"
       loading="lazy">
  <div class="card-body">
    <h3>Service Name</h3>
    <p>Description text.</p>
  </div>
</div>
```

**Which sections use Pattern A:**
- Service cards (Bespoke Suits, Wedding Suits, Alterations, etc.)
- Process step cards (01 Consultation, 02 Measurements, etc.)
- Style/product cards (2-Piece Suit, 3-Piece Suit, etc.)
- Any repeating grid of 3–6 cards

**Each card gets its own image.** Do not use one image for an entire section — assign one image per card, choosing the most contextually relevant image. Reuse existing images across cards if a perfect unique image doesn't exist for every card.

---

### Pattern B — Two-column sections (text left, image right)

Used for "Our Story", "About", fabric sections, "Why Choose Us" etc.

```html
<div class="fabric-header"> <!-- or whatever two-col class is used -->
  <div>
    <!-- text content, headings, tags -->
  </div>
  <div>
    <img src="images/{filename}.webp"
         alt="{entity-based alt text}"
         style="width:100%;aspect-ratio:4/3;object-fit:cover;border:1px solid rgba(201,169,110,0.18);"
         loading="lazy" width="800" height="600">
  </div>
</div>
```

---

### Pattern C — Full-width hero image

Every page gets a full-width hero image immediately after the page header / breadcrumb section.

```html
<section style="padding:0;">
  <img src="images/{page-slug}-hero.webp"
       alt="{primary entity} at {business name}, {location}"
       style="width:100%;max-height:480px;object-fit:cover;display:block;"
       loading="eager" width="1600" height="640">
</section>
```

`loading="eager"` on heroes only — all other images use `loading="lazy"`.

---

### Pattern D — Section intro image (above a grid, below heading)

For large sections with a heading + description followed by a card grid, optionally add one wide image between the intro text and the grid.

```html
<img src="images/{filename}.webp"
     alt="{section entity at business name}"
     style="width:100%;max-height:360px;object-fit:cover;margin-bottom:40px;border:1px solid rgba(201,169,110,0.18);"
     loading="lazy" width="1600" height="480">
```

---

### Pattern E — Blog article inline image

```html
<figure style="margin:40px 0;border:1px solid var(--border);overflow:hidden;">
  <img src="images/{filename}.webp"
       alt="{entity-based alt text}"
       width="760" height="420"
       style="width:100%;height:300px;object-fit:cover;display:block;"
       loading="lazy">
  <figcaption style="padding:10px 14px;font-size:0.76rem;color:var(--muted);background:var(--bg2);">
    {short caption}
  </figcaption>
</figure>
```

---

## Step 7 — Alt text formula

Every image must have entity-based alt text:

```
{Primary entity} {action or context} at {business name}, {location detail}

Examples:
"Bespoke suit on dress form at The Outfit Custom Tailor workshop, Bangtao Beach Phuket"
"Expert tailor hands measuring suit jacket at The Outfit, Choeng Thale Phuket"
"Premium wool and linen fabric swatches for bespoke tailoring at The Outfit, Phuket"
"Tailoring workshop interior at The Outfit Custom Tailor, Bangtao Beach Phuket"
"Wedding suit on hanger at The Outfit bespoke tailor, Phuket Thailand"
```

Never use generic alt text like "image of suit" or "photo".

---

## Step 8 — Semantic filename convention

```
{entity}-{qualifier}-{location}-{context}.webp

Max 60 characters. Lowercase. Hyphens only. No underscores.

Examples:
bespoke-suit-dress-form-phuket-tailor.webp
tailor-workshop-interior-bangtao-phuket.webp
premium-suit-fabric-swatches-phuket.webp
tailor-measuring-fitting-bespoke-suit.webp
wedding-suit-hanger-tropical-boutique-phuket.webp
garment-alteration-hands-phuket-tailor.webp
tailoring-atelier-wide-shot-phuket.webp
```

---

## Integration with content-brief and weekly-pipeline

When `/content-brief` finishes a page brief, or `/weekly-pipeline` finishes an article:

1. Extract all section headings and card grids from the written content
2. Map each section to a pattern (A=card grid, B=two-column, C=hero, D=section intro, E=blog inline)
3. For card grids: plan **one image per card** — not one image per section
4. Ask: "Should I generate images for this page? I'll create [X] images — [list each: hero, 3 service cards, 4 process step cards, etc.]"
5. If yes, run Steps 1–7 for that page automatically
6. Return all images compressed and verified under 200KB, with placement HTML inserted using the correct pattern

**The image-gen skill always covers:**
- 1x full-width hero (Pattern C)
- 1x image per card in every service/process/style card grid (Pattern A)
- 1x image per two-column section with text + image layout (Pattern B)

**The image-gen skill does NOT add images to:**
- FAQ accordion sections
- Pricing tables
- Navigation, footer, breadcrumbs
- Pure text CTA buttons
- Testimonial quote blocks

---

## Output summary format

After all images are placed, report:

```
Images generated and placed for: [page name]

Hero:
  → images/bespoke-tailoring-workshop-hero-phuket.webp (87KB) — placed after <header>

About section:
  → images/tailor-workshop-interior-bangtao-phuket.webp (112KB) — placed in .about-section

Process section:
  → images/tailor-measuring-fitting-bespoke-suit.webp (64KB) — placed before process steps

Fabric section:
  → images/premium-suit-fabric-swatches-phuket.webp (143KB) — placed before fabric grid

Total: 4 images, 406KB combined
```

---

## WordPress integration (Novamira MCP)

After generating and converting, upload each WebP to the WordPress media library:

```
mcp__novamira__mcp-adapter-execute-ability(
    ability="upload_media",
    parameters={
        "file_path": "images/{filename}.webp",
        "alt_text": "{entity-based alt text}",
        "title": "{descriptive title}"
    }
)
```

Then use the returned media URL when inserting into WordPress page content.
