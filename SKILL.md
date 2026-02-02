---
name: briefing-builder
version: 1.0.0
description: |
  Build personalized news briefings from 40+ curated sources. Supports beat coverage
  (climate, tech, defense), regional focus (Asia, Europe, LATAM), or general international
  news. Non-technical setup via interactive wizard.
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - WebFetch
  - AskUserQuestion
  - mcp__Claude_in_Chrome__navigate
  - mcp__Claude_in_Chrome__computer
  - mcp__Claude_in_Chrome__read_page
  - mcp__Claude_in_Chrome__get_page_text
  - mcp__Claude_in_Chrome__tabs_context_mcp
  - mcp__Claude_in_Chrome__tabs_create_mcp
---

# Briefing Builder

Create personalized news briefings from curated sources. Describe what you want in plain English.

---

## Triggers

Use this skill when the user says:
- "Build my briefing"
- "Generate my news briefing"
- "Morning briefing"
- "What's in the news?"
- `/briefing`

For setup:
- "Set up my briefing"
- "Configure my news sources"
- `/setup-briefing`

---

## Setup Wizard (`/setup-briefing`)

Walk the user through creating their personalized config. Be conversational, not robotic.

### Step 1: Focus Area

Ask what they cover:

> "What's your beat or focus? I can set you up with:
> - **General international** - broad global coverage
> - **Regional** - Asia, Europe, Middle East, Latin America, or Africa
> - **Beat** - climate, tech, business, politics, defense
> - **Custom** - describe it and I'll find sources"

### Step 2: Source Selection

Based on their answer, suggest sources from `sources.json` presets:

**For General International:**
> "I'll include the big three wires (Reuters, AP, AFP) plus BBC, Guardian, Al Jazeera, FT, and Economist. Want to add or remove any?"

**For Regional (e.g., Asia):**
> "For Asia coverage, I'll use: SCMP, Japan Times, Nikkei Asia, Yomiuri, Asahi. Plus Reuters and AP for wire coverage. Sound good, or want changes?"

**For Beat (e.g., Climate):**
> "For climate, I'd suggest: Reuters, AP, Guardian (strong climate desk), BBC, FT. Want me to add any specialist sources? I can try to find RSS feeds or scrape their homepage."

**For Custom:**
> "Name the outlets you want. I'll check if they have RSS feeds. If not, I can scrape their homepage live when you generate a briefing."

### Step 3: Custom Sources

If they name sources not in the preset:

1. **Check `sources.json`** - Look for the source by name or alias
2. **If found:** Add to their config
3. **If not found:** Offer to:
   - Search for RSS at common paths (`/feed`, `/rss`, `/rss.xml`)
   - Set up live homepage scraping via Chrome MCP

> "I don't have [Source] in my database. Want me to:
> 1. Try to find their RSS feed, or
> 2. Scrape their homepage live when you generate briefings?"

### Step 4: Live Scrape Sources (Optional)

Ask if they want homepage screenshots for editorial context:

> "Some paywalled sites (WSJ, FT) have thin RSS feeds but break scoops on their homepage. Want me to screenshot any sites for leads? I'd check these live each time you generate a briefing."

Common suggestions: WSJ, FT, Politico, Bloomberg

### Step 5: Output Style

> "How do you want your briefing formatted?
> - **Conversational** (like talking to a well-informed friend)
> - **Bullet summary** (quick scan, one line per story)
> - **Detailed** (includes excerpts for deep research)"

### Step 6: Save Config

Create the config file:

```
~/.claude/skills/briefing-builder/configs/{username}.json
```

Then ask:

> "Done! Your briefing is configured. Want me to also add this to your CLAUDE.md so it's always available? (recommended for redundancy)"

If yes, append a summary to their `~/.claude/CLAUDE.md`:

```markdown
## My Briefing Config
- Focus: [their focus]
- Sources: [list]
- Live scrape: [if any]
- Style: [their choice]
```

---

## Generate Briefing (`/briefing`)

When user asks for their briefing:

### 1. Load User Config

Check for config in this order:
1. `~/.claude/skills/briefing-builder/configs/{username}.json`
2. Briefing section in user's CLAUDE.md
3. If neither exists, offer to run setup wizard

### 2. Fetch RSS Sources

For each source in their config:

1. Look up the source in `sources.json`
2. Fetch the RSS feed(s) using WebFetch
3. Parse items, keep only last 24 hours
4. Note source attribution for each item

**Example WebFetch for RSS:**
```
WebFetch: https://feeds.bbci.co.uk/news/world/rss.xml
Prompt: "Extract all news items. For each, return: title, link, pubDate, description (first 200 chars)"
```

### 3. Fetch Live Scrape Sources (if configured)

For sources marked for live scraping:

1. Use Chrome MCP to navigate to the URL
2. Use `get_page_text` or `read_page` to extract headlines
3. Optionally take a screenshot for editorial context

### 3a. Economist World in Brief

Run the headless scraper to get top 2 briefs (paywall limits free access):

```bash
node ~/Downloads/nyt-concierge/economist-wib.js
```

Returns JSON with `briefs` array containing Economist's editorial pick of top global stories. Only 2 briefs due to paywall, but still valuable for editorial priority signals.

### 4. Deduplicate and Cluster

- Group stories by topic (fuzzy match on headlines, key entities)
- Note which outlets are covering the same story
- Flag stories with 3+ outlet coverage as "big stories"

### 5. Generate Briefing

Follow the user's style preference:

**Conversational (default):**
```
Good morning. Here's the state of play:

[2-3 narrative paragraphs synthesizing the lead stories, with embedded links]

**Business/Tech**
- [Story with context, source attribution, embedded link]
- [Another story]

**Around the World**
- **Asia**: [bullets]
- **Europe**: [bullets]
- **Middle East**: [bullets]
```

**Bullet Summary:**
```
## Top Stories
- [Headline] - [Source]
- [Headline] - [Source]

## Business
- [Headline] - [Source]

## Regional
- [Headline] - [Source]
```

**Detailed:**
```
## Lead Story: [Headline]
[First 2-3 sentences from article]
Source: [Outlet] | [Link]

## [Next Story]
...
```

---

## Source Resolution

When the user names a source (during setup or adding sources):

### 1. Check sources.json

Search by name and aliases (case-insensitive):
- "BBC" → matches `bbc` source
- "Wall Street Journal" or "WSJ" → matches `wsj` source
- "SCMP" or "South China Morning Post" → matches `scmp` source

### 2. Check domain_to_source for attribution

When displaying stories, use canonical source names:
- `wsj.com` → "Wall Street Journal"
- `ft.com` → "Financial Times"

### 3. Fallback: Find RSS or Scrape

If source not in database:

1. Ask user for the website URL
2. Try common RSS paths:
   - `{url}/feed`
   - `{url}/rss`
   - `{url}/rss.xml`
   - `{url}/feed.xml`
3. If RSS found, add to user's config
4. If no RSS, offer live scraping

---

## Writing Rules (MANDATORY)

All briefings must follow these rules:

### 1. No interpretive leads
Lead with facts, not commentary. Don't open with "connect the dots" analysis.

### 2. No forced connections
Don't manufacture themes between unrelated stories. Let unrelated stories stand alone. Avoid weak glue like "The international picture reflects similar tensions."

### 3. No 's = "is" contractions
Only use 's for possessives ("Amazon's CEO"), never for "is" or "has."
- BAD: "Amazon's cutting jobs"
- GOOD: "Amazon is cutting jobs"

### 4. Less editorializing
Cut loaded phrases:
- "saber-rattling"
- "belt-tightening"
- "reaching a crescendo"
- "makes diplomats nervous"

### 5. No tacked-on analysis
End with the fact, not the implication. Never add clauses explaining what news "means."

**Banned phrases:**
- "It's a reminder..."
- "highlighting how..."
- "a testament to..."
- "This signals..."
- "it suggests..."
- "It's exactly the kind of..."
- "which could..."

### 6. No editorializing in bullets
Bullets should be pure news, no commentary.
- BAD: "The AI arms race is getting expensive fast."
- GOOD: "OpenAI is raising prices for API access, citing compute costs."

### 7. Max 3 words per link
Hard rule for hyperlink text.
- BAD: `[Federal Reserve is investigating the bank](url)`
- GOOD: `[Fed is investigating](url) the bank`

### 8. Varied attribution
Mix it up: "Reuters reports", "according to Bloomberg", "per AP", "the BBC reports"
Use each phrasing only once per briefing.

---

## Edge Cases

### Source without RSS
1. Inform user: "[Source] doesn't have a public RSS feed"
2. Offer live scraping as alternative
3. If they decline, skip the source

### Non-English sources
1. Fetch headlines as-is
2. Translate during briefing generation
3. Attribute with country: "Bild (Germany) reports..."

### RSS fetch fails
1. Log the failure (don't crash)
2. Continue with other sources
3. Note at end: "Note: Could not reach [Source] feed"

### No config found
1. Inform user: "You haven't set up your briefing yet"
2. Offer to run setup wizard
3. Or offer a one-time briefing with default sources

---

## Config File Format

User configs are stored as JSON:

```json
{
  "user": "colleague_name",
  "focus": "asia_regional",
  "sources": {
    "rss": ["scmp", "japantimes", "nikkei", "reuters", "ap", "bbc"],
    "live_scrape": [
      {
        "name": "WSJ",
        "url": "https://www.wsj.com",
        "screenshot": true
      }
    ]
  },
  "style": {
    "format": "conversational",
    "max_stories": 15,
    "include_screenshots": false
  },
  "created": "2026-02-02",
  "last_updated": "2026-02-02"
}
```

---

## Quick Reference

| Command | Action |
|---------|--------|
| `/setup-briefing` | Run setup wizard |
| `/briefing` | Generate briefing |
| "Add [source] to my briefing" | Add source to config |
| "Remove [source]" | Remove from config |
| "Change my style to bullets" | Update output format |
| "Show my briefing config" | Display current settings |
