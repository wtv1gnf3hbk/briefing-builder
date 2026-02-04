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
> - **Conversational** (like talking to a well-informed friend, but never use 's as a contraction for "is")
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

## Time-Aware Greeting

Before generating the briefing, detect the user's local time and select an appropriate greeting.

### 1. Detect Local Time

Run this command to get the user's current local time and timezone:

```bash
date "+%H:%M %p %Z"
```

This returns format like: `07:32 AM EST` or `21:32 PM KST`

The timezone is detected from the user's machine settings, so it automatically adjusts when they travel.

### 2. Select Greeting Based on Local Hour

| Local Hour (24h) | Greeting |
|------------------|----------|
| 05:00 - 11:59 | Good morning |
| 12:00 - 16:59 | Good afternoon |
| 17:00 - 20:59 | Good evening |
| 21:00 - 04:59 | (no greeting, start with "Here's the state of play") |

### 3. Format the Briefing Header

Include the timestamp in the greeting:

```
Good morning. Here's the state of play as of 7:32 AM EST:
```

Or for late night:
```
Here's the state of play as of 2:15 AM EST:
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

### 4a. Business/Tech Keyword Scan (MANDATORY)

Before generating the briefing, explicitly search ALL RSS results for these high-value keywords:
- **Company names**: Apple, Google, Microsoft, Amazon, Meta, Tesla, SpaceX, OpenAI, Nvidia, xAI
- **Deal terms**: merger, acquisition, acquires, IPO, funding, billion, trillion
- **People**: Musk, Bezos, Zuckerberg, Altman, Nadella, Cook, Pichai

Run a grep/search through the full feed output for these terms. If any story contains these keywords and you haven't already flagged it, add it to the briefing.

**Why this matters:** Business/tech stories often appear in only 1-2 feeds (Japan Times, SCMP, Bloomberg) and can be missed when scanning headlines quickly. A $1.2 trillion merger should never be missed.

### 5. Generate Briefing

Follow the user's style preference:

**Conversational (default):**
```
[Time-appropriate greeting per Time-Aware Greeting section]. Here's the state of play as of [HH:MM AM/PM TZ]:

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

## Lead Story Selection (MANDATORY)

### 0. Check Yesterday's Lead (First!)
Before selecting today's lead, check the lead log:

```
~/.claude/skills/briefing-builder/lead-log.json
```

Format:
```json
{
  "2026-02-01": {"lead": "Immigration enforcement in Minnesota", "topic": "immigration"},
  "2026-01-31": {"lead": "Trump tariff announcement", "topic": "tariffs"}
}
```

After generating each briefing, append today's lead to this log.

When selecting the lead story for the briefing, apply these filters in order:

### 1. Second-Day Filter
Exclude anything that led yesterday's briefing (check `lead-log.json`) UNLESS there are significant new developments. New analysis, new commentary, or "the story continues" framing does NOT count as a new development. A new development means: new facts, new official statements, new data, or a meaningful escalation/de-escalation.

### 2. Recency Filter
Story must have broken or significantly developed in the last 12-18 hours. If NYT published it yesterday morning, it's not today's lead.

### 3. Scoop Detection
Prefer scoops over aggregated coverage. Detect scoops via:

1. **Language signals**: Look for "first reported by", "according to [outlet] reporting", "exclusively learned", "[Outlet] has learned"
2. **Timestamp comparison**: When multiple outlets cover the same story, check pubDates. If one outlet has it 6+ hours before others, that's likely the scoop. Credit the original outlet.
3. **Single-source stories**: If only one major outlet is covering something newsworthy, it's probably their scoop.

A fresh exclusive beats ongoing coverage of a known situation.

### 4. Surprise Test
Apply the "would an informed reader be surprised?" test. Low-surprise stories (predictable developments, expected outcomes) should not lead. High-surprise stories (unexpected revelations, contradictions of conventional wisdom) should be weighted higher.

**Low surprise examples:**
- "ICE raids cause protests" (expected outcome)
- "Stock market reacts to Fed decision" (routine)
- "Politician criticized by opposition" (dog bites man)

**High surprise examples:**
- "UAE firm quietly bought into president's crypto company" (revelation)
- "Administration official contradicts stated policy" (unexpected)
- "[Country] reverses decades-old position on [issue]" (shift)

### 5. Coverage Saturation Penalty
If 15 outlets are leading with the same thing, your readers have probably already seen it. The briefing's value-add is surfacing what they missed, not echoing what everyone else is saying.

### 6. Reasoning on Demand
Do NOT include lead selection reasoning in the briefing output by default. Instead:
- Mentally apply all filters and make lead selection decisions
- Keep reasoning ready but hidden
- If Adam asks "why did you lead with X?", "show your reasoning", "why not Y?", or similar → provide the full reasoning

**When providing reasoning, explain:**
- Why the lead story was chosen (which filters it passed)
- What stories were considered but demoted, and why
- Any close calls or judgment calls made

**Example reasoning (only when asked):**
```
The UAE crypto story leads because it's a fresh scoop (WSJ), high surprise (unexpected foreign investment in president's company), and not saturated across outlets. Immigration was demoted to a bullet because it's been the lead for multiple days and today's developments (release of one family) don't constitute a significant escalation.
```

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

### 7. Link Text Optimization
Hard rule: max 3 words per link, and link the NEWS not the attribution.

**Link the news:**
- BAD: `the [BBC reports](url)` (link on attribution)
- GOOD: `[killed 12 miners](url), the BBC reports` (link on news)

**Keep links short:**
- BAD: `[Federal Reserve is investigating the bank](url)`
- GOOD: `[Fed is investigating](url) the bank`

The linked text should be the most informative/clickable part of the sentence.

### 8. Varied attribution
Mix it up: "Reuters reports", "according to Bloomberg", "per AP", "the BBC reports"
Use each phrasing only once per briefing.

### 9. Non-NYT Source Quota (MANDATORY)
At least 2 bullets in the "Around the World" section must link to non-NYT sources (BBC, Al Jazeera, Reuters, AP, SCMP, Guardian, FT, etc.).

This is a hard requirement, not a suggestion. If the briefing only has NYT links in the international section, go back and find non-NYT coverage for at least 2 stories.

---

## Story Clustering (Before Lead Selection)

Before picking the lead, cluster all gathered stories by topic:

1. **Group stories**: Match headlines by key entities (people, places, events) and keywords
2. **Create clusters**: "Trump immigration", "Epstein files", "Ukraine war", "Iran nuclear", etc.
3. **Score clusters**: Weight by (a) number of outlets covering, (b) recency, (c) surprise factor
4. **Pick lead from top cluster**: The lead should come from the most newsworthy cluster, not picked in isolation

This prevents picking a minor scoop that's actually a subplot of a bigger story.

---

## Coverage Check (After Drafting)

After drafting the briefing, scan for missing regions:

**Required regions** (at least a mention or explicit acknowledgment):
- Europe
- Asia
- Middle East
- Africa
- Latin America

If any region has zero coverage:
1. First, try to find something from that region in the gathered sources
2. If genuinely nothing newsworthy, that's fine - but be aware of the gap
3. Don't manufacture coverage just to fill the slot

**Flag for Adam** if 2+ regions are missing - this might indicate a source gap or unusually quiet news day in those areas.

---

## Pre-Publish Validation (MANDATORY)

Before outputting the briefing, run these automated checks. This is NOT optional.

### 1. Contraction Detection (STOP AND LIST)

**MANDATORY STEP:** Before outputting the briefing, you MUST:
1. Find every instance of `'s` in your draft
2. List them out loud (in your thinking, not to user)
3. For each one, check: is the next word a VERB or a NOUN?

**Common verbs that follow contractions (FIX THESE):**
- named, announced, said, reported, confirmed, denied
- merging, acquiring, raising, heading, planning, considering, expanding
- reportedly, expected, set, planning

**Example violations and HOW TO FIX:**
- `Disney's named` → "Disney named" (past tense verb: DROP the 's)
- `OpenAI's said` → "OpenAI said" (past tense verb: DROP the 's)
- `Musk's merging` → "Musk is merging" (present continuous: EXPAND to "is")
- `Amazon's raising` → "Amazon is raising" (present continuous: EXPAND to "is")
- `India's heading` → "India is heading" (present continuous: EXPAND to "is")
- `Tesla's reportedly` → "Tesla is reportedly" (adverb: EXPAND to "is")

**The fix depends on tense:**
- Past tense verbs (named, said, announced, confirmed): DROP the 's
- Present continuous (-ing verbs): EXPAND 's to "is"

**SAFE patterns (possessives - keep):**
- `Amazon's CEO` → CEO is a noun = possessive ✓
- `Trump's administration` → administration is a noun ✓
- `Disney's board` → board is a noun ✓

### 2. Banned Phrase Scan (Exact String Match)

Search for these EXACT strings. If found, delete the phrase or rewrite the sentence:

```
It's a reminder
highlighting how
a testament to
This signals
it suggests
It's exactly the kind of
which could
This isn't just about
makes diplomats nervous
reaching a crescendo
```

### 3. Business/Tech Highlights Check

The MCP response includes a `businessTechHighlights` section with pre-extracted stories matching keywords (SpaceX, merger, IPO, billion, etc.).

**MANDATORY:** Review this section. If it contains stories not already in your briefing draft, add them.

### 4. Feed Health Check

The MCP response includes `feedHealth.failed` with any RSS feeds that couldn't be fetched.

**If 3+ feeds failed:** Add a note at the end of the briefing:
> "Note: Some sources were unreachable today: [list failed sources]"

### 5. NYT Coverage Check (MANDATORY - MOST IMPORTANT)

**NYT is the primary news source.** Before outputting, verify NYT stories are included:

1. **Count NYT links**: Scan the draft for `nytimes.com` URLs
2. **Minimum threshold**: At least 5 NYT links must appear in the briefing
3. **If below threshold**:
   - STOP - do not output the briefing yet
   - Scrape NYT directly using:
     ```bash
     node -e '...' # (NYT scraper in nyt-concierge)
     ```
     Or via the `generate_briefing` MCP tool
   - Add missing NYT stories to relevant sections
   - Re-run this check

**NYT scrape fallback (if MCP times out):**
```bash
cd /Users/adampasick/Downloads/nyt-concierge && node -e '
const https = require("https");
const cheerio = require("cheerio");
// ... [scraper code - see server.js scrapeNYT function]
'
```

**Section coverage**: Ensure NYT links appear in:
- Lead paragraphs (at least 1)
- Business/Tech section (at least 1)
- Around the World section (at least 2)

**Why this matters**: NYT is Adam's employer and the most important source for his briefing. A briefing without NYT stories is incomplete. Other sources (BBC, FT, SCMP, Al Jazeera) provide international perspective, but NYT must always be present.

### 6. Validation Loop (Retry Until Pass)

**Process:**
1. Draft the briefing
2. Run checks 1-5 (contraction, banned phrase, business/tech, feed health, NYT coverage)
3. If ANY violation found:
   - Fix the violation
   - Increment attempt counter
   - Re-run checks 1-2 and 5 (contraction, banned phrase, NYT coverage)
4. Repeat until ALL checks pass

**NYT check is blocking**: If NYT coverage is below threshold, you MUST fetch NYT stories before continuing. Do not output a briefing with fewer than 5 NYT links.

**Attempt tracking:**
- Attempt 1-2: Fix silently, continue
- Attempt 3: Fix, then warn Adam: "⚠️ Validation took 3 attempts. Issues found: [list what was fixed]"
- Attempt 4+: Fix, warn with stronger message: "🚨 Validation required {N} attempts. Consider reviewing the draft more carefully."

**What to report on warn:**
- Which 's contractions were fixed (and how)
- Which banned phrases were removed
- Do NOT include business/tech or feed health in attempt count (those are one-time checks)

**Do not skip this.** The briefing must pass validation before output. The loop ensures quality without blocking output indefinitely.

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
