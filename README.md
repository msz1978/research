# research — market-research feed for the trading bot

Data bridge between ChatGPT and my paper-trading bot — bidirectional.

- **Who writes:** a ChatGPT scheduled task ("TradingResearcher", daily
  ~15:00 CET) commits `research.json` + `archive/<date>.json` via the
  GitHub plugin; data via StockLens, Financial Datasets, Alpha Vantage
  & friends
- **Who reads:** the bot fetches the raw file at session start (23:00),
  injects it into the strategy planner, archives every snapshot, and a
  15:30 daily job commits a copy to my Forgejo
- **Feedback loop:** after each session the bot writes
  `feedback/<date>.json` — which watchlist symbols its planner actually
  used vs ignored. The GPT reads it before writing the next research
- Public on purpose — the bot reads `raw.githubusercontent.com` no-auth

Setup details + the GPT prompt live in `docs/TRADING_RESEARCHER_GPT.md` in
the bot repo.
