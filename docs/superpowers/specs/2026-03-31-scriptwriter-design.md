# Scriptwriter Tool — Design Spec
**Date:** 2026-03-31
**Status:** Approved

---

## Overview

A local web app that takes a creator from zero to a finished short-form video script in ~2-3 minutes. It researches trending topics, competitor angles, and keywords, then uses the Claude API to generate a script, caption, and hashtags — all tailored to the user's personal brand and niche.

**Target platforms:** TikTok, YouTube Shorts, Instagram Reels
**Target user:** Personal brand creator with a business angle

---

## Architecture

| Layer | Technology |
|-------|-----------|
| Backend | Python + Flask |
| Frontend | Plain HTML / CSS / JavaScript |
| AI | Claude API (claude-sonnet-4-6) |
| Research | Web search via HTTP requests + HTML parsing (no API key needed) |
| Storage | Local JSON files |

The app runs locally (`python app.py`) and is accessed via browser at `localhost:5000`. No cloud infrastructure required beyond the Claude API. A Claude API key must be set in a `.env` file before first run. If the topic field is left blank, Claude generates 5 topic ideas based on the user's niche and recent trends before the research step.

---

## User Flow

1. **Dashboard** — Shows past scripts and a "Create New Script" button
2. **Step 1 — Topic Input** — User enters a rough idea or leaves blank for suggestions; niche is pre-filled from settings
3. **Step 2 — Research** — App searches for:
   - Trending topics in the user's niche
   - High-performing competitor content angles
   - Relevant search keywords
   - Results are displayed; user picks an angle to pursue
4. **Step 3 — Format Selection** — User chooses output format:
   - Full word-for-word script
   - Structured outline with talking points
   - Hook + bullet points + CTA
5. **Step 4 — Generation** — Claude writes the script using research findings, chosen angle, format, and user's niche/tone settings
6. **Step 5 — Results** — Script, caption, and hashtags displayed on one page with copy buttons and a regenerate option
7. **Save** — Script saved locally with date, topic, and format metadata

---

## Features

### Niche Settings
- User saves their niche, tone (professional / casual / hype), and brand details once
- These are injected into every research query and AI prompt automatically

### Research Panel
- Runs three parallel searches: trending topics, competitor content, keyword data
- Displays findings before generation so user can choose the best angle
- Research is shown as scannable cards, not raw data

### Script Generator
- Produces content with a strong hook, structured body, and clear CTA
- Always tailored to short-form video (60–90 second pacing)
- Uses niche settings + chosen angle + format preference

### Caption Generator
- Auto-generates a platform-optimized caption from the finished script
- Tone matches the user's brand settings
- Includes a natural CTA in the caption

### Hashtag Generator
- Suggests 10–15 relevant hashtags based on topic, niche, and research keywords
- Mix of high-volume and niche-specific tags

### Regenerate
- Available for script, caption, and hashtags independently
- Re-runs generation with a slightly different prompt for variation

### Script History
- All saved scripts stored locally in JSON
- Searchable/filterable by date and topic on the dashboard

### Copy to Clipboard
- Separate copy buttons for script, caption, and hashtags

---

## Data Model

```json
// Saved script entry (stored in data/scripts.json)
{
  "id": "uuid",
  "created_at": "2026-03-31T12:00:00",
  "topic": "How I grew my brand in 30 days",
  "angle": "Personal story with lessons",
  "format": "full_script",
  "niche": "personal brand / business",
  "tone": "casual",
  "script": "...",
  "caption": "...",
  "hashtags": ["#personalbrand", "..."],
  "research_summary": "..."
}
```

```json
// Settings (stored in data/settings.json)
{
  "niche": "personal brand and business",
  "tone": "casual",
  "brand_details": "...",
  "platforms": ["tiktok", "youtube_shorts", "instagram_reels"]
}
```

---

## Pages

| Route | Purpose |
|-------|---------|
| `GET /` | Dashboard — script history + new script button |
| `GET /settings` | Niche/tone/brand settings form |
| `POST /settings` | Save settings |
| `GET /create` | Wizard step 1 — topic input |
| `POST /research` | Runs research, returns results |
| `POST /generate` | Generates script + caption + hashtags |
| `POST /save` | Saves script to local JSON |
| `GET /history` | Full script history view |

---

## Out of Scope (this build)

- Direct posting to social platforms
- Video editing suggestions
- Scheduling / content calendar
- Multi-user support

These are planned for future phases of the broader social media manager.
