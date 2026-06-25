---
name: foundation
description: Runs Phase 1 of the Semantic SEO framework — Foundation & Strategy. Performs market research, market segmentation, ICP construction, search intent mapping, and competitor semantic gap analysis for a local business. Produces a 01-foundation.md deliverable. Use when the user says "foundation", "/foundation", "market research", "ICP research", "search intent map", "competitor gap analysis", "who is my target customer", or "start Phase 1".
tools:
  - Write
  - WebSearch
---

# Foundation & Strategy (Phase 1)

Ask the user for:
1. Business name and niche/service type
2. City and service area
3. Full address including building name if applicable
4. **Is the business inside a multi-story building?** If yes, what floor? (e.g. "2nd Floor", "5th Floor") — needed for `floorLevel` schema property
5. Names of 2–3 main competitors
6. Does a website already exist for this business?

Store the floor level answer. If the business is in a multi-story building (mall, office tower, shared commercial building), the `floorLevel` property MUST be added to the LocalBusiness schema in Phase 5. This is a Google-introduced schema.org property that disambiguates businesses sharing the same building — critical for businesses inside malls, plazas, or multi-tenant towers.

## Step 0a — Existing site baseline (existing site rebuilds only)

**If the user confirmed an existing website in the intake:**

Invoke the `seo-plan` skill first:
- Run `/seo plan [domain]` to get a quick competitive baseline: current rankings, content gaps, what's working, industry-specific opportunities
- This takes 5–10 minutes and produces a snapshot that feeds directly into the Koray research below
- Do NOT skip this for rebuilds — it prevents duplicating what already works and reveals what's failing

Then proceed with GSC data pull below.

**If new site (no existing domain):** skip this step entirely. Go to Step 0b.

---

## Step 0b — Customer voice research (always run)

Before any market research, run `blog-discourse` to capture what real customers say about this business type in this city:

Invoke `/blog discourse [business type] [city]` with `--days 30`

For example: `/blog discourse custom tailor Bangkok --days 30`

This searches Reddit, TripAdvisor reviews, Google reviews, Facebook groups, TikTok comments, Instagram hashtags, YouTube videos, and travel forums. It produces `DISCOURSE.md` containing:
- What customers complain about (pain points → ICP problems)
- What they praise (trust signals → content angles)
- The exact language they use (vocabulary → H1s, meta descriptions, ad copy)
- Influencer and blogger mentions (link building targets)
- Trending questions (FAQ content, PAA opportunities)

**After `DISCOURSE.md` is produced:** read it before writing the ICP profiles and search intent map. Use the real customer language — not generic marketing copy.

---

## Step 0c — Brand voice (always run)

Invoke `/blog persona` to define the brand voice before any content is written:
- Tone dimensions: Funny↔Serious, Formal↔Casual, Respectful↔Irreverent, Enthusiastic↔Matter-of-fact
- Sets vocabulary tier, sentence length, contraction frequency
- Saves as a persona file used automatically by blog-write, blog-brief, and weekly-pipeline

Ask the user: "How would you describe the brand voice? (e.g. professional and precise, friendly and approachable, luxury and exclusive)" — use their answer to configure the persona.

---

**If a website already exists:** use Composio's `GOOGLE_SEARCH_CONSOLE_LIST_SITES` to find the connected GSC property. Then call `GOOGLE_SEARCH_CONSOLE_SEARCH_ANALYTICS_QUERY` with dimensions=["query"] for the past 3 months. Use the real query data to:
- Identify which queries the site already ranks for (skip researching these in competitor gap — focus on what's MISSING)
- Identify queries where the site has impressions but low clicks (intent mismatch — flag for content brief improvement)
- Identify queries where position > 20 (site is visible but weak — these become topical map priorities)

Then perform the following and write everything to `01-foundation.md`:

## Market Research
Search the primary service + city query. Survey the top 10 ranking pages. Note:
- What services dominate the results
- What trust signals appear (certifications, years in business, awards)
- Any featured snippets or local packs

## Market Segmentation
Create a table of 4–6 customer segments: Segment Name | Description | Search Behavior | Value to Business

## ICP Profiles
Write 2–3 ideal customer profiles. Each includes: name, demographics, problem, exact queries they use, what makes them convert, which landing page they reach first.

## Search Intent Map
Table mapping queries to intent types (Informational / Commercial / Transactional / Navigational) with the page type that should target each.

## Competitor Gap Analysis
For each competitor: Semantic Gap | Topical Gap | Entity Gap | Opportunity Rating

---

## CoR Step — Central Entity Attribute Mapping

After the competitor gap analysis, define the **EAV attribute hierarchy** for the Central Entity (CE). This feeds directly into Phase 2 entity research and every content brief. Reference `cor/frameworks/eav-architecture.md` for full rules.

**Attribute types (priority order — highest to lowest):**

| Type | Definition | Rule |
|------|-----------|------|
| **Unique Attributes** | Features that uniquely identify this CE — no other entity shares the same attribute+value | Place early in all content. These prove expertise. |
| **Root Attributes** | Essential for basic definition of the CE — without these, the entity is incomplete | Must be covered on every core page. |
| **Rare Attributes** | Specific details not commonly found — proves depth of knowledge | Use to differentiate from competitors. |

**Output a table in `01-foundation.md`:**

```
Central Entity (CE): [business type + city]

| Attribute Type | Attribute | Value | Notes |
|----------------|-----------|-------|-------|
| Unique | [e.g. speciality not offered by competitors] | [specific value] | |
| Unique | | | |
| Root | [e.g. price range, turnaround time, location] | [specific value] | |
| Root | | | |
| Root | | | |
| Rare | [e.g. specific technique, certification, niche process] | [specific value] | |
| Rare | | | |
```

**KBT rule:** once these values are written here, they are locked. Every page, schema markup, and GBP listing must use the exact same values. Contradictions across pages destroy Knowledge-Based Trust (KBT) and increase Cost of Retrieval. See `cor/concepts/cost-of-retrieval.md`.

Produce the file `01-foundation.md` and confirm it was saved.
