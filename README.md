# Briefing Builder

Create personalized news briefings from 40+ curated sources. No code required.

## Quick Start

1. Run `/setup-briefing` in Claude Code
2. Answer the wizard questions (focus, sources, style)
3. Run `/briefing` anytime to generate your briefing

That's it.

---

## What It Does

- Pulls from 40+ curated RSS feeds (wire services, international papers, regional outlets)
- Falls back to live homepage scraping for sources without RSS
- Generates briefings in your preferred style
- Deduplicates and clusters stories across sources

---

## Presets

The wizard offers these starting points:

| Preset | Sources |
|--------|---------|
| **General International** | Reuters, AP, AFP, BBC, Guardian, Al Jazeera, FT, Economist |
| **Asia Focus** | SCMP, Japan Times, Nikkei, Yomiuri, Asahi + wires |
| **Europe Focus** | Guardian, Le Monde, El País, Bild, Corriere + wires |
| **Middle East Focus** | Al Jazeera, Times of Israel, Haaretz + wires |
| **Latin America Focus** | Folha, O Globo, Estadão, El País + wires |
| **Business/Finance** | FT, Bloomberg, WSJ, Economist, Nikkei, Reuters |

You can add/remove sources from any preset.

---

## Adding Custom Sources

During setup (or anytime), just name the outlet:

> "Add Axios to my briefing"

Claude will:
1. Check if it's in the database
2. If not, search for an RSS feed
3. If no RSS exists, offer to scrape the homepage live

---

## Output Styles

**Conversational** (default)
> Good morning. Here's the state of play: [narrative paragraphs with embedded links]

**Bullet Summary**
> Quick one-liners, one story per bullet

**Detailed**
> Full excerpts from each story for deep research

---

## Your Config

Preferences are saved in:
```
~/.claude/skills/briefing-builder/configs/{your-name}.json
```

Optionally also in your `~/.claude/CLAUDE.md` for redundancy.

---

## Commands

| Command | What it does |
|---------|--------------|
| `/setup-briefing` | Run the setup wizard |
| `/briefing` | Generate your briefing |
| "Add [source]" | Add a source to your config |
| "Remove [source]" | Remove a source |
| "Show my config" | Display current settings |
| "Change style to bullets" | Switch output format |

---

## Troubleshooting

**"Source not found"**
Provide the full URL (e.g., axios.com) and Claude will search for an RSS feed or set up live scraping.

**"Stale data"**
RSS feeds cache at the source. For truly fresh data, add the site to your live_scrape list.

**"Too many stories"**
Ask Claude to adjust `max_stories` in your config, or focus on a specific region/topic.

**"Config not loading"**
Check that your config file exists at `~/.claude/skills/briefing-builder/configs/`. Run `/setup-briefing` to recreate.

---

## For Developers

This skill is pure instructions (no custom code). It uses:
- `WebFetch` for RSS feeds
- Chrome MCP for live scraping
- `sources.json` for the curated feed database

To add new sources to the database, edit `sources.json` directly.

---

## Credits

Curated sources extracted from Adam Pasick's morning briefing system. Writing rules adapted from The World newsletter style guide.
