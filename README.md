# research — market-research feed for the trading bot

Data bridge between ChatGPT and my paper-trading bot, nothing more.

- **Who writes:** a ChatGPT scheduled task ("TradingResearcher") commits
  `research.json` (+ `archive/<date>.json`) via the GitHub plugin
- **Who reads:** the bot fetches the raw file at session start and injects
  it into the strategy planner as market context
- Repo is public on purpose — the bot reads `raw.githubusercontent.com`
  with no auth

Setup details + the GPT prompt live in `docs/TRADING_RESEARCHER_GPT.md` in
the bot repo.
