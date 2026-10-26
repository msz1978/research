# research — external market-research feed for the trading bot

This repo is a **data bridge**, not code. It carries curated market research
from a ChatGPT scheduled task ("TradingResearcher") into the
self-improving-trading-bot paper-trading bot.

## Why it exists

ChatGPT Go can't talk to the bot directly, and the bot can't call ChatGPT.
GitHub is the neutral middle layer: the GPT **commits** `research.json` here,
the bot **fetches** it over `raw.githubusercontent.com` (no auth — repo is
public). Durable, versioned, zero extra services.

## Structure

research.json            # latest research (what the bot consumes)
archive/YYYY-MM-DD.json  # per-day snapshots (research history)

## Pipeline

ChatGPT scheduled task (daily ~15:00 CET, GitHub plugin)
    → commits research.json + archive/<date>.json to main
    → bot's improvement session (23:00) GETs the raw URL
    → saved to workspace/research.json (+ workspace/research_history/)
    → injected into the planner prompt as market context

Bot-side config (self-improving-trading-bot/.env):

RESEARCH_URL=https://raw.githubusercontent.com/msz1978/research/main/research.json

## Schema

{
  "generated_at": "YYYY-MM-DD",
  "watchlist": [{"symbol": "...", "thesis": "one sentence"}],
  "sector_notes": "2-4 sentences on sector momentum/breakdowns",
  "earnings_flags": [{"symbol": "...", "date": "YYYY-MM-DD", "note": "..."}],
  "universe_suggestions": ["SYM1", "SYM2"],
  "avoid": [{"symbol": "...", "reason": "..."}]
}

## Rules

- The bot ignores research older than **7 days** (`generated_at`)
- Every run overwrites `research.json` and adds `archive/<date>.json`
- Full setup + paste-ready prompt: [docs/TRADING_RESEARCHER_GPT.md](cci:7://file:///home/marcin/self-improving-trading-bot/docs/TRADING_RESEARCHER_GPT.md:0:0-0:0) in the
  bot repo
- Manual fallback if the scheduled task can't write: trigger "update
  research" in a regular ChatGPT chat with the GitHub plugin enabled
