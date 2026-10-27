# TradingResearcher — production guardrails

These rules prevent provider-specific failures from being misclassified as global data-source failures and prevent unverified research from being persisted.

## 1. Provider availability is endpoint-specific

Never infer `provider_unavailable` from one failed endpoint.

For every provider, maintain three states independently:

- `available`: the provider is responding successfully;
- `endpoint_unavailable`: the provider works, but this endpoint/metric is not entitled, rate-limited, unsupported, or otherwise unavailable;
- `provider_unavailable`: repeated basic calls fail across otherwise independent endpoints or the provider is unreachable.

Example: Massive returning `NOT_ENTITLED` for a snapshot endpoint means **that endpoint is unavailable**, not that Massive is unavailable. A successful `/v2/aggs/ticker/{ticker}/prev` call proves the provider itself is available.

## 2. Capability discovery before research

At the beginning of every run:

1. Read `feedback/` first.
2. Probe each primary provider with one low-cost, known-safe capability.
3. Record internally which capabilities succeeded, failed, were rate-limited, or were not entitled.
4. Use only successful capabilities.
5. Do not downgrade the entire research run because one capability failed.

Do not persist provider diagnostics, internal scores, API errors, or capability matrices into `research.json`.

## 3. Fallback policy

Fallback is metric-level, not provider-level.

For each required metric:

`primary endpoint -> another entitled endpoint from same provider -> designated secondary provider -> web/IR verification -> omit metric`

Never silently replace a missing value with a guessed value.

If a metric cannot be verified, reduce confidence internally and compensate by requiring stronger evidence elsewhere. If evidence is still insufficient, exclude the candidate rather than inventing data.

## 4. Rate-limit protection

Respect provider limits immediately.

- After `RATE_LIMIT`, stop hammering that endpoint.
- Reduce request volume and use already retrieved data.
- Prefer batch/grouped endpoints when actually entitled.
- Do not interpret a rate-limit response as bad market data.
- Do not interpret a rate-limit response as provider outage.
- Do not retry the same request in a tight loop.

## 5. Entitlement protection

`NOT_ENTITLED`, `FORBIDDEN`, or equivalent means the requested data product is outside the current plan unless the provider explicitly says otherwise.

The bot must:

- mark only that capability unavailable;
- test a lower-tier/known-entitled alternative when practical;
- continue with other available data;
- never claim that the whole provider is unavailable.

## 6. Cross-source validation

For every final watchlist candidate, require enough independent evidence to support the thesis:

- StockLens for candidate discovery and technical/quality ranking;
- Massive for current/most-recent market price action when available;
- Financial Datasets for fundamentals/SEC/ownership when available;
- secondary providers or web for missing specialist metrics;
- company IR / authoritative calendars for earnings dates when a date affects inclusion.

If a source is unavailable, do not fabricate its evidence. The thesis must be written only from evidence that was actually retrieved.

## 7. Earnings-date hard gate

An earnings date inside the intended holding period is an event-risk flag, not a minor detail.

Before persisting an earnings date:

1. Prefer company investor relations or an authoritative filing/calendar.
2. Cross-check when the date materially changes inclusion.
3. If two sources disagree, do not choose the earlier date by guesswork.
4. Resolve the conflict or omit the date and exclude the candidate if the event risk cannot be bounded.

The current run demonstrated why this matters: EOG was initially treated as November 5 by secondary sources, while EOG's own investor-relations calendar specifies November 6. The authoritative date wins.

## 8. No stale-data masquerading as current data

Every market-sensitive claim must be tied internally to a retrieval date/time or clearly identified as historical.

Never describe yesterday's data as real-time data.
Never use an old provider response as evidence that a current endpoint works.
Never carry yesterday's earnings calendar into today's run without revalidation.

## 9. Candidate rejection safety

A high internal score is never sufficient by itself.

Reject or flag candidates when there is:

- imminent earnings/event risk;
- unresolved data conflict;
- poor liquidity/execution quality;
- extreme gap risk;
- contradictory primary evidence;
- missing critical data needed to support the thesis.

Do not use a high score to override a hard data-quality or event-risk gate.

## 10. Persistence gate

Before writing `research.json`:

- validate JSON syntax;
- validate the exact top-level schema;
- validate `generated_at` equals today's research date;
- validate every ticker is a valid symbol returned by the research universe;
- validate earnings dates that materially affect candidates;
- validate that no internal score/confidence/source/provider diagnostic is persisted;
- validate that every thesis is supported by retrieved evidence;
- validate that no citation markup or tool-internal metadata is present in JSON;
- validate `research.json` and `archive/<date>.json` will be byte-for-byte identical.

After writing:

1. Read both files back from GitHub.
2. Parse both as JSON.
3. Compare their normalized content and SHA/content hash.
4. Only report success if both reads succeed and the contents are identical.

If any write or verification fails, report the research as **completed but persistence failed**. Never claim successful persistence based only on a write response.

## 11. Failure classification

Use this internal decision tree:

```text
Did the provider respond?
  no -> test one independent basic endpoint
       if that also fails -> provider_unavailable
       otherwise -> endpoint_unavailable
  yes -> did the requested endpoint return entitlement/rate-limit/error?
        yes -> endpoint_unavailable
        no -> capability available
```

Then continue at the metric level.

## 12. Never repeat the original failure mode

The specific prohibited behavior is:

> `Massive snapshot endpoint -> NOT_ENTITLED -> Massive unavailable -> abandon all Massive data`

Correct behavior is:

> `Massive snapshot endpoint -> NOT_ENTITLED -> snapshot unavailable -> test entitled aggregates/previous bars -> use successful Massive data -> fallback only for missing metrics`

This rule applies to every provider, not just Massive.
