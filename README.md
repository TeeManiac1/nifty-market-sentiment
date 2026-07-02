# Nifty Market Sentiment Pipeline Cheatsheet

## Project Goal
Build a research pipeline that converts Indian stock-market YouTube commentary into structured transcript, chunk, and feature data, then compares those features with NIFTY and India VIX market moves. The longer-term direction is to filter relevant stock/index chunks cheaply first, and only then use LLM extraction for richer market and stock-level signal capture.

## Current Pipeline Order
1. Discover raw YouTube candidates.
2. Review and keep filtered candidate videos.
3. Collect transcripts for candidate videos.
4. Build timestamp-aware transcript chunks.
5. Build descriptive and forward-looking text features.
6. Aggregate features daily and weekly against NIFTY and India VIX.
7. Maintain stock-universe reference data.
8. Next planned step: LLM-based alias/transliteration enrichment, then stock/index matching, then filtered LLM extraction.

## Daily Command Sequence
```bash
python src/discovery/discover_youtube_candidates.py --start-date YYYY-MM-DD --end-date YYYY-MM-DD
python src/pipeline/collect_youtube_transcripts.py --limit 300 --sleep-seconds 1
python src/pipeline/build_transcript_chunks.py --chunk-words 500 --overlap-words 75
python src/pipeline/build_text_features.py --ignore-leading-overlap-words 75
python src/analysis/analyze_sentiment_market_relationship.py
```

## Important Scripts
`src/discovery/discover_youtube_candidates.py`
- Finds candidate Indian stock-market videos for target dates.
- Main raw outputs:
  - `data/raw/video_sources/youtube_candidates_raw.csv`
  - `data/raw/video_sources/youtube_candidates_filtered.csv`
- Typical parameters:
  - `--start-date`
  - `--end-date`
  - `--backfill-days`

`src/pipeline/collect_youtube_transcripts.py`
- Fetches transcripts for filtered candidates.
- Main output:
  - `data/processed/youtube_transcripts.csv`
- Default behavior:
  - append only new `video_id` values
- Important parameters:
  - `--limit`
  - `--sleep-seconds`
  - `--provider`
  - `--replace-output`
- Important note:
  - needs `TRANSCRIPTAPI_API_KEY`

`src/pipeline/build_transcript_chunks.py`
- Splits transcript text into timestamp-aware chunks.
- Main output:
  - `data/processed/transcript_chunks.csv`
- Default behavior:
  - append/incremental by `video_id`
- Important parameters:
  - `--chunk-words 500`
  - `--overlap-words 75`
  - `--rebuild`
  - `--force-video-id`
- Important note:
  - chunk IDs are deterministic: `video_id_chunk_XXXX`
  - do not mix chunking versions

`src/pipeline/build_text_features.py`
- Builds descriptive and forward-looking phrase features from chunks.
- Main output:
  - `data/processed/transcript_features.csv`
- Default behavior:
  - append/incremental by `chunk_id`
- Important parameters:
  - `--ignore-leading-overlap-words 75`
  - `--rebuild`
  - `--force-video-id`
  - `--force-chunk-id`
- Important note:
  - overlap ignore must stay aligned with chunk overlap

`src/analysis/analyze_sentiment_market_relationship.py`
- Aggregates features daily and weekly and merges them with NIFTY and India VIX data.
- Main outputs:
  - `data/processed/daily_sentiment_features.csv`
  - `data/processed/daily_sentiment_market_features.csv`
  - `data/processed/sentiment_market_correlations.csv`
  - `data/processed/weekly_sentiment_features.csv`
  - `data/processed/weekly_sentiment_market_features.csv`
  - `data/processed/weekly_sentiment_market_correlations.csv`
- Default behavior:
  - rebuild summary outputs from feature files

`src/reference/build_indian_stock_universe.py`
- Builds the stock-universe reference from the local NSE equity list.
- Main output:
  - `data/reference/indian_stock_universe.csv`

`src/reference/add_hindi_transliterations.py`
- Tested local transliteration approaches.
- Output:
  - `data/reference/indian_stock_universe_with_transliteration.csv`
- Current decision:
  - do not use this output for matching

## Append vs Rebuild Rule
Default rule:
- daily scripts should increasingly work in append/incremental mode by default
- use `--rebuild` only when logic or version settings change

Use append/incremental mode for:
- transcript collection by `video_id`
- chunk building by `video_id`
- feature building by `chunk_id`

Use rebuild mode for:
- chunking logic changes
- feature dictionary or feature logic changes
- version changes such as `chunking_version` or `feature_version`
- small analysis outputs, which are safe to recompute

Force modes:
- `--force-video-id` for chunks and features
- `--force-chunk-id` for features only

## Important Parameters
`discover_youtube_candidates.py`
- `--start-date`: start of discovery range
- `--end-date`: end of discovery range
- `--backfill-days`: recent weekday collection mode

`collect_youtube_transcripts.py`
- `--limit 300`: cap how many new candidates to fetch in one run
- `--sleep-seconds 1`: polite pause between fetch attempts

`build_transcript_chunks.py`
- `--chunk-words 500`: chunk size
- `--overlap-words 75`: overlap between adjacent chunks
- `--rebuild`: full chunk rebuild with backup
- `--force-video-id`: regenerate one or more videos

`build_text_features.py`
- `--ignore-leading-overlap-words 75`: avoids double-counting overlap words
- `--rebuild`: full feature rebuild with backup
- `--force-video-id`: regenerate feature rows for one or more videos
- `--force-chunk-id`: regenerate specific chunks only

## Main Files And What They Mean
`data/raw/video_sources/youtube_candidates_raw.csv`
- raw discovery output

`data/raw/video_sources/youtube_candidates_filtered.csv`
- filtered candidates ready for transcript collection

`data/processed/youtube_transcripts.csv`
- transcript master store
- append by `video_id`
- contains status, transcript text, and timestamp JSON

`data/processed/transcript_chunks.csv`
- chunk master store
- append by `video_id`
- rebuild if chunking logic changes

`data/processed/transcript_features.csv`
- feature master store
- append by `chunk_id`
- rebuild if dictionary or feature logic changes

`data/processed/market_data.csv`
- market benchmark data
- must cover the sentiment date range

`data/reference/indian_stock_universe.csv`
- core NSE stock-universe reference

`data/reference/indian_stock_universe_with_transliteration.csv`
- tested transliteration output
- not trusted for matching

## Current Interpretation Of Results
Current approximate scale:
- about 665 successful transcript videos
- about 38.5k chunks
- about 16.3M feature words
- about 28 sentiment dates in the current daily analysis span

Current read on the pipeline:
- same-day dictionary signal behaves like a market mood thermometer
- future aggregate predictive signal is still noisy and weak
- stock/index separation is needed before LLM extraction
- cheap filtering should happen before any expensive LLM step

## Transliteration / Alias Decision
Current decision:
- `simple_rules` and `indic_transliteration` local attempts produced poor Hindi market-style transliteration
- do not use `indian_stock_universe_with_transliteration.csv` for stock matching
- next plan is LLM-based alias/transliteration enrichment

Strict rule:
- transliterate, do not translate

Bad example:
- `Power Grid -> शक्ति ग्रिड`

Good example:
- `Power Grid -> पावर ग्रिड`

Also avoid:
- overbroad aliases like `Bank`, `Power`, `India`, `Steel`, `Capital`

## Scaling Decision
Current storage style:
- CSV is okay for now

Current plan:
- extend CSV life by append/incremental chunk and feature processing
- later consider Parquet + DuckDB if scale or query complexity grows

Hard rule:
- never run LLM on all chunks
- first filter by stock/index matching and other cheap heuristics

## Rebuild Commands
```bash
python src/pipeline/build_transcript_chunks.py --chunk-words 500 --overlap-words 75 --rebuild
python src/pipeline/build_text_features.py --ignore-leading-overlap-words 75 --rebuild
python src/analysis/analyze_sentiment_market_relationship.py
```

## Common Checks
Check transcript status:
```bash
python -c "import pandas as pd; df=pd.read_csv('data/processed/youtube_transcripts.csv'); print(df.shape); print(df['status'].value_counts(dropna=False)); print(df[['video_id','status','language','channel_title','target_date']].tail(20).to_string(index=False))"
```

Check chunks:
```bash
python -c "import pandas as pd; df=pd.read_csv('data/processed/transcript_chunks.csv'); print(df.shape); print(df[['chunk_id','target_date','video_id','language','chunk_start_time','chunk_end_time','chunk_word_count','channel_title']].head(20).to_string(index=False)); print(df['chunk_word_count'].describe())"
```

Check features:
```bash
python -c "import pandas as pd; df=pd.read_csv('data/processed/transcript_features.csv'); print(df.shape); print(df[['chunk_id','target_date','video_id']].head(20).to_string(index=False))"
```

## Next Steps
1. Finish and trust the incremental chunk/features workflow as the daily default.
2. Create `stock_alias_llm_input.csv` from the clean stock universe.
3. Run a small LLM alias enrichment test on the first 100 stocks.
4. Run full LLM alias enrichment and build `indian_stock_universe_enriched.csv`.
5. Build stock mention matching against chunks.
6. Build index mention matching for NIFTY, Bank Nifty, and Sensex.
7. Test LLM extraction only on matched, relevant chunks.

## Update Note: Current Source Of Truth
The older sections above are still useful for the early pipeline, but the current stock-level research branch is now much larger than the original cheatsheet. Use the sections below as the current source of truth for the post-matcher workflow.

## Current Expanded Workflow (2026-07-01)
1. Discover raw YouTube candidates.
2. Filter candidate videos manually or with current review rules.
3. Collect transcripts into the transcript master.
4. Build transcript chunks.
5. Build transcript text features.
6. Run market-level sentiment analysis.
7. Build the Indian stock universe.
8. Run the completed LLM alias enrichment reference workflow.
9. Merge safe aliases into `data/reference/indian_stock_universe_enriched.csv`.
10. Match stock mentions against transcript chunks with `src/features/match_stock_mentions.py`.
11. Build stock-centered context windows with `src/features/build_stock_mention_contexts_v2.py`.
12. Run the strict v2.1 context-builder mode and keep only LLM-worthy rows.
13. Use `data/processed/stock_mention_contexts_v2_1_kept.csv` as the current main LLM input file.
14. Run `src/llm/extract_stock_contexts.py` in dry-run mode first.
15. Reuse or append to `data/processed/llm_extractions/llm_extractions_master.csv` so already-extracted rows are not re-called.
16. Write a run-specific extraction CSV for the current pilot or batch.
17. Flatten extracted price levels with `src/analysis/flatten_extracted_price_levels.py`.
18. Update extracted-level OHLC coverage with `src/market/update_stock_ohlc_for_extracted_levels.py`.
19. Track price-level hits locally with `src/analysis/track_extracted_price_level_hits.py`.
20. Build Track A price-level paper-trading summaries with `src/reporting/build_price_level_paper_trading_summary.py`.
21. Track directional outcomes locally with `src/analysis/track_directional_call_outcomes.py`.
22. Build Track B directional toy paper-trading summaries with `src/reporting/build_paper_trading_summary.py`.
23. Run `src/reporting/audit_llm_trade_run.py` as the final sanity-check command before scaling a batch.

## Current Context Builder Files
`data/processed/stock_mention_contexts_v2.csv`
- Base Context Builder v2 output.
- Current size seen locally: about `183,725` rows.
- This is broader than the strict LLM-ready subset.

`data/processed/stock_mention_contexts_v2_1.csv`
- Strict Context Builder v2.1 full output.
- Current size seen locally: about `183,725` rows.
- Adds stricter quality and mismatch diagnostics.

`data/processed/stock_mention_contexts_v2_1_test.csv`
- Limited strict v2.1 test output.
- Current size seen locally: about `5,539` rows.
- Useful only for smaller QA checks.

`data/processed/stock_mention_contexts_v2_1_kept.csv`
- Current main LLM input file.
- Current size seen locally: about `5,020` rows.
- Current bucket mix:
  - `A_strong_trade_context`: `4,632`
  - `B_likely_trade_context`: `388`
- This is the file to use for current LLM extraction pilots unless you intentionally rerun the context-builder and regenerate a newer kept subset.

## Current Normal LLM Flow
1. Build or refresh stock matches:
```bash
python src/features/match_stock_mentions.py --overwrite --progress-every 1000 --match-engine candidate
```
2. Build base v2 stock contexts:
```bash
python src/features/build_stock_mention_contexts_v2.py --force-overwrite --output-path data/processed/stock_mention_contexts_v2.csv --diagnostics-dir data/processed/diagnostics/context_builder_v2
```
3. Build strict v2.1 test:
```bash
python src/features/build_stock_mention_contexts_v2.py --strict-llm-filter --limit 500 --force-overwrite --output-path data/processed/stock_mention_contexts_v2_1_test.csv --diagnostics-dir data/processed/diagnostics/context_builder_v2_1_test
```
4. Build strict v2.1 full:
```bash
python src/features/build_stock_mention_contexts_v2.py --strict-llm-filter --force-overwrite --output-path data/processed/stock_mention_contexts_v2_1.csv --diagnostics-dir data/processed/diagnostics/context_builder_v2_1
```
5. Keep only the rows approved for LLM use:
```bash
python - <<'PY'
import pandas as pd
inp = "data/processed/stock_mention_contexts_v2_1.csv"
out = "data/processed/stock_mention_contexts_v2_1_kept.csv"
df = pd.read_csv(inp)
kept = df[df["v2_keep_for_llm"] == True].copy()
kept.to_csv(out, index=False)
print("saved", out, kept.shape)
PY
```
6. Dry-run the LLM extractor:
```bash
python src/llm/extract_stock_contexts.py --input-path data/processed/stock_mention_contexts_v2_1_kept.csv --provider openrouter --model openai/gpt-4o-mini --schema-version v3 --limit 3 --random-sample --dry-run --output-path data/processed/llm_extractions/dry_run_preview.csv
```
7. Run a real cached extraction batch only after dry-run QA passes.

## LLM Extraction Cache / Master Store
Current extraction behavior is now append-only and cache-aware.

Main master store:
- `data/processed/llm_extractions/llm_extractions_master.csv`

Purpose:
- Do not pay again for rows already extracted with the same LLM setup.
- Do not lose older successful extraction rows when downstream logic changes.
- Keep one append-only historical extraction store plus run-specific output files.

Cache identity:
- The extractor now treats a row as already done when a successful matching cache key exists for:
  - `stock_context_id_v2`
  - `provider`
  - `model`
  - `temperature`
  - `schema_version`
  - `prompt_version`

Important extractor cache parameters:
- `--master-output-path`
- `--use-cache`
- `--no-cache`
- `--append-to-master`
- `--no-append-to-master`
- `--force-reextract`
- `--import-existing-output`
- `--checkpoint-every`
- `--disable-checkpoints`
- `--max-api-retries`
- `--api-retry-backoff-seconds`

Important rule:
- Matching successful rows are reused from the master by default.
- Failed rows are not treated as completed by default.
- Run-specific output files still include both cache hits and new API rows, so downstream scripts can use the run file normally.

Checkpoint persistence rule:
- Live extraction now checkpoint-writes the current run output periodically.
- At each checkpoint the script rewrites the full current run CSV safely using a temp-file replace.
- If `--output-jsonl` is enabled, the JSONL companion file is also refreshed safely at checkpoints.
- New API rows completed since the last checkpoint are appended safely to the master store at checkpoint time when `--append-to-master` is enabled.
- Cache-hit rows are written into the run output file but are never re-appended to the master store.
- If a long run dies late, restarting with cache enabled should reuse the already checkpointed successful master rows instead of re-calling the API for them.

OpenRouter retry/backoff rule:
- The OpenRouter client now retries only transient failures:
  - request timeout or connection/request error
  - HTTP `429`
  - HTTP `500`
  - HTTP `502`
  - HTTP `503`
  - HTTP `504`
- It does not retry obvious permanent failures such as missing API key, `400`, `401`, `403`, or `404`.
- Current default retry plan:
  - max retries: `3`
  - backoff seconds: `5,15,45`
- Retry attempts are logged in the terminal during live runs so you can see when a row is being retried instead of failing immediately.

Current visible cache metadata columns include:
- `extraction_run_id`
- `extracted_at`
- `cache_key`
- `cache_status`
- `cache_hit`
- `reused_from_master`

Safe dry-run example:
```bash
python src/llm/extract_stock_contexts.py --input-path data/processed/stock_mention_contexts_v2_1_kept.csv --output-path data/processed/llm_extractions/cache_check.csv --provider openrouter --model openai/gpt-4o-mini --schema-version v3 --limit 250 --random-sample --seed 42 --dry-run
```

Safer long-run example with checkpointing:
```bash
python src/llm/extract_stock_contexts.py --input-path data/processed/stock_mention_contexts_v2_1_kept.csv --output-path data/processed/llm_extractions/openrouter_gpt4omini_v3_next_batch.csv --provider openrouter --model openai/gpt-4o-mini --schema-version v3 --limit 1000 --random-sample --seed 42 --checkpoint-every 25
```

Long-run example with both checkpointing and explicit retry controls:
```bash
python src/llm/extract_stock_contexts.py --input-path data/processed/stock_mention_contexts_v2_1_kept.csv --output-path data/processed/llm_extractions/openrouter_gpt4omini_v3_next_batch.csv --provider openrouter --model openai/gpt-4o-mini --schema-version v3 --limit 1000 --random-sample --seed 42 --checkpoint-every 25 --max-api-retries 3 --api-retry-backoff-seconds 5,15,45
```

Safe import example for an old output:
```bash
python src/llm/extract_stock_contexts.py --import-existing-output data/processed/llm_extractions/openrouter_gpt4omini_v3_patch_2_anomaly_250.csv
```

## Current Extraction Versions
Current extractor status:
- `schema_version v3` is now active for the current trade-evaluation branch.
- Current live prompt family includes the safety-patched v3 trade-evaluation prompt.

Important operational rule:
- Dry-run first.
- Then a tiny live or cache-aware batch.
- Then flatten, evaluate, and audit.

## Track A: Explicit Price-Level Branch
Main scripts:
- `src/analysis/flatten_extracted_price_levels.py`
- `src/market/update_stock_ohlc_for_extracted_levels.py`
- `src/analysis/track_extracted_price_level_hits.py`
- `src/reporting/build_price_level_paper_trading_summary.py`
- `src/reporting/analyze_price_level_alpha_vs_nifty.py`

Normal command order:
```bash
python src/analysis/flatten_extracted_price_levels.py --input-path data/processed/llm_extractions/openrouter_gpt4omini_v3_patch_2_anomaly_250.csv --output-dir data/processed/llm_extractions/analysis
python src/market/update_stock_ohlc_for_extracted_levels.py --levels-path data/processed/llm_extractions/analysis/extracted_price_levels_flat_v3_patch_2_anomaly_250.csv --output-dir data/processed/market
python src/analysis/track_extracted_price_level_hits.py --levels-path data/processed/llm_extractions/analysis/extracted_price_levels_flat_v3_patch_2_anomaly_250.csv --mapping-path data/processed/market/symbol_yfinance_map.csv --ohlc-path data/processed/market/stock_ohlc_daily.csv --output-dir data/processed/llm_extractions/analysis --output-file extracted_price_level_hits_v3_patch_2_anomaly_250.csv --summary-file extracted_price_level_hits_v3_patch_2_anomaly_250_summary.json
python src/reporting/build_price_level_paper_trading_summary.py --levels-path data/processed/llm_extractions/analysis/extracted_price_level_hits_v3_patch_2_anomaly_250.csv --ohlc-path data/processed/market/stock_ohlc_daily.csv --output-dir data/processed/llm_extractions/analysis --run-label v3_patch_2_anomaly_250_from_hits --force-overwrite
```

Important Track A input contract:
- `build_price_level_paper_trading_summary.py` expects the enriched hit-tracking file from `track_extracted_price_level_hits.py`.
- Do not pass the raw flattened levels file directly into the Track A paper-trading summary script.

Important Track A safety rule:
- Implausible or likely wrong-symbol levels should be blocked at hit-tracking level, not only later in paper trading.

Track A alpha-vs-NIFTY report:
- `src/reporting/analyze_price_level_alpha_vs_nifty.py`
- Purpose:
  - read the existing Track A paper-trading trades file
  - align each trade horizon to NIFTY over the same trading-day window
  - compute `alpha_vs_nifty_*d_pct`
  - report the main benchmark as:
    - long-only
    - complete 10d window
    - dedup by `target_date + symbol`
    - alpha versus NIFTY
- Important rule:
  - this script does not mutate the original Track A trade file
  - it writes a separate enriched CSV, summary JSON, markdown report, and group-summary CSV

Typical alpha-vs-NIFTY command:
```bash
python src/reporting/analyze_price_level_alpha_vs_nifty.py --trades-path data/processed/llm_extractions/analysis/price_level_paper_trading_trades_v3_patch_2_anomaly_1000_next_open.csv --market-path data/processed/market_data.csv --output-dir data/processed/reports --run-label v3_patch_2_anomaly_1000_next_open
```

## Track B: Directional No-Price-Level Branch
Main scripts:
- `src/analysis/track_directional_call_outcomes.py`
- `src/reporting/build_paper_trading_summary.py`

Track B purpose:
- Evaluate forward-looking bullish or bearish stock calls even when the extraction does not contain a full explicit target-stop setup.
- Use local OHLC only.
- Do not call another LLM to decide whether the directional call worked.

Directional paper-trading script:
- `src/reporting/build_paper_trading_summary.py`

Important new capability:
- Supports both equal-notional and weighted-notional toy sizing modes.
- Weighted sizing uses extraction-time metadata only.
- This is still a toy paper-trading report, not a real backtest.

Important weighted sizing parameters:
- `--sizing-mode equal|weighted|both`
- `--weighted-sizing-method budget_normalized|raw_multiplier`
- `--min-weighted-notional`
- `--max-weighted-notional`
- `--disable-weighted-caps`
- `--write-sizing-diagnostics`

Typical weighted report command:
```bash
python src/reporting/build_paper_trading_summary.py --directional-outcomes-path data/processed/llm_extractions/analysis/directional_call_outcomes_20.csv --price-level-hits-path data/processed/llm_extractions/analysis/extracted_price_levels_hit_tracking_20.csv --output-dir data/processed/reports --run-label pilot_20_weighted --notional 10000 --round-trip-cost-pct 0.0 --sizing-mode both --weighted-sizing-method budget_normalized --write-sizing-diagnostics
```

## Audit Script
New script:
- `src/reporting/audit_llm_trade_run.py`

Purpose:
- One quick post-run sanity check across:
  - extraction output
  - flattened levels
  - hit tracking
  - paper trades
  - paper skipped groups

Main outputs:
- `audit_summary_<run_label>.json`
- `audit_suspicious_levels_<run_label>.csv`
- `audit_blocked_levels_<run_label>.csv`
- `audit_top_winners_<run_label>.csv`
- `audit_top_losers_<run_label>.csv`
- `audit_repeated_symbols_<run_label>.csv`
- `audit_manual_review_candidates_<run_label>.csv`

Normal audit command:
```bash
python src/reporting/audit_llm_trade_run.py --extractions-path data/processed/llm_extractions/openrouter_gpt4omini_v3_patch_2_anomaly_250.csv --flat-levels-path data/processed/llm_extractions/analysis/extracted_price_levels_flat_v3_patch_2_anomaly_250.csv --hit-tracking-path data/processed/llm_extractions/analysis/extracted_price_level_hits_v3_patch_2_anomaly_250.csv --paper-trades-path data/processed/llm_extractions/analysis/price_level_paper_trading_trades_v3_patch_2_anomaly_250_from_hits.csv --paper-skipped-path data/processed/llm_extractions/analysis/price_level_paper_trading_skipped_v3_patch_2_anomaly_250_from_hits.csv --output-dir data/processed/reports/audits --run-label v3_patch_2_anomaly_250 --top-n 30
```

What the audit is meant to catch quickly:
- parse success unexpectedly low
- cache misses unexpectedly high
- too many suspicious price levels
- possible wrong-symbol rows still remaining eligible
- too few or zero paper trades
- repeated-symbol concentration
- skipped-group reason spikes

## Current Important Files To Use
Use:
- `data/reference/indian_stock_universe_enriched.csv`
- `aliases_final`
- `data/processed/stock_mention_contexts_v2_1_kept.csv`
- `data/processed/llm_extractions/llm_extractions_master.csv`
- `data/processed/llm_extractions/analysis/extracted_price_level_hits_*.csv` for Track A paper trading
- `data/processed/llm_extractions/analysis/directional_call_outcomes_*.csv` for Track B paper trading

Do not use:
- `data/reference/indian_stock_universe_with_transliteration.csv`
- raw flattened levels as direct input to `build_price_level_paper_trading_summary.py`
- all chunk rows as direct LLM input

## Current Practical End-To-End Command Block
```bash
python src/features/match_stock_mentions.py --overwrite --progress-every 1000 --match-engine candidate
python src/features/build_stock_mention_contexts_v2.py --strict-llm-filter --force-overwrite --output-path data/processed/stock_mention_contexts_v2_1.csv --diagnostics-dir data/processed/diagnostics/context_builder_v2_1
python - <<'PY'
import pandas as pd
df = pd.read_csv("data/processed/stock_mention_contexts_v2_1.csv")
df[df["v2_keep_for_llm"] == True].copy().to_csv("data/processed/stock_mention_contexts_v2_1_kept.csv", index=False)
print("saved kept file")
PY
python src/llm/extract_stock_contexts.py --input-path data/processed/stock_mention_contexts_v2_1_kept.csv --provider openrouter --model openai/gpt-4o-mini --schema-version v3 --limit 250 --random-sample --seed 42 --dry-run --output-path data/processed/llm_extractions/dry_run_preview.csv
python src/analysis/flatten_extracted_price_levels.py --input-path data/processed/llm_extractions/openrouter_gpt4omini_v3_patch_2_anomaly_250.csv --output-dir data/processed/llm_extractions/analysis
python src/market/update_stock_ohlc_for_extracted_levels.py --levels-path data/processed/llm_extractions/analysis/extracted_price_levels_flat_v3_patch_2_anomaly_250.csv --output-dir data/processed/market
python src/analysis/track_extracted_price_level_hits.py --levels-path data/processed/llm_extractions/analysis/extracted_price_levels_flat_v3_patch_2_anomaly_250.csv --mapping-path data/processed/market/symbol_yfinance_map.csv --ohlc-path data/processed/market/stock_ohlc_daily.csv --output-dir data/processed/llm_extractions/analysis --output-file extracted_price_level_hits_v3_patch_2_anomaly_250.csv --summary-file extracted_price_level_hits_v3_patch_2_anomaly_250_summary.json
python src/reporting/build_price_level_paper_trading_summary.py --levels-path data/processed/llm_extractions/analysis/extracted_price_level_hits_v3_patch_2_anomaly_250.csv --ohlc-path data/processed/market/stock_ohlc_daily.csv --output-dir data/processed/llm_extractions/analysis --run-label v3_patch_2_anomaly_250_from_hits --force-overwrite
python src/reporting/audit_llm_trade_run.py --extractions-path data/processed/llm_extractions/openrouter_gpt4omini_v3_patch_2_anomaly_250.csv --flat-levels-path data/processed/llm_extractions/analysis/extracted_price_levels_flat_v3_patch_2_anomaly_250.csv --hit-tracking-path data/processed/llm_extractions/analysis/extracted_price_level_hits_v3_patch_2_anomaly_250.csv --paper-trades-path data/processed/llm_extractions/analysis/price_level_paper_trading_trades_v3_patch_2_anomaly_250_from_hits.csv --paper-skipped-path data/processed/llm_extractions/analysis/price_level_paper_trading_skipped_v3_patch_2_anomaly_250_from_hits.csv --output-dir data/processed/reports/audits --run-label v3_patch_2_anomaly_250 --top-n 30
```

## Parallel LLM Extraction Upgrade
The extractor now supports safe single-process parallel API execution while preserving:
- append-only master cache reuse
- checkpoint safety
- deterministic final row ordering
- one central writer/checkpointer only

Hard rule:
- do not run multiple extractor processes against the same master cache at the same time
- one extractor process should own:
  - the API worker threads
  - the run output CSV
  - the optional JSONL output
  - the recovery journal
  - the append-only master cache writes

New `extract_stock_contexts.py` controls:
- `--max-concurrent-api-calls`
  - default `1`
  - use `5` as the first serious larger-run setting
- `--progress-every`
  - prints completed rows, cache hits, API completions, failures, cost, speed, ETA, pending master rows, and checkpoint count
- `--max-run-cost-usd`
  - optional hard budget cap for a live run
- `--request-start-spacing-seconds`
  - optional spacing between API call starts to reduce 429 bursts
- `--input-manifest-path`
  - replay an exact selected manifest
- `--export-selection-manifest`
  - freeze the exact selected rows before a paid run
- `--recovery-jsonl-path`
  - live recovery journal written row by row by the main thread

What did not change:
- cache identity still uses:
  - `stock_context_id_v2`
  - `provider`
  - `model`
  - `temperature`
  - `schema_version`
  - `prompt_version`
- master store remains:
  - `data/processed/llm_extractions/llm_extractions_master.csv`
- cache hits still show up in the run-specific output
- only new API rows are appended to the master store

Why this matters:
- long paid runs are no longer forced to be fully sequential
- progress is visible during the run
- checkpoints persist the current run output and master-cache progress
- reruns after interruption remain cost-safe because cache reuse skips already completed successful rows

## Manifest + Parallel Run Workflow
Recommended safe workflow for the remaining large extraction:

1. Build or refresh `stock_mention_contexts_v2_1_kept.csv`.
2. Run a dry-run with manifest export.
3. Inspect the manifest row count.
4. Run the real extraction from that exact manifest.
5. If the run is interrupted, rerun the same manifest with cache enabled.
6. Continue with:
   - `flatten_extracted_price_levels.py`
   - `track_extracted_price_level_hits.py`
   - `build_price_level_paper_trading_summary.py`
   - `analyze_price_level_alpha_vs_nifty.py`

Recommended dry-run manifest command:
```bash
python src/llm/extract_stock_contexts.py \
  --input-path data/processed/stock_mention_contexts_v2_1_kept.csv \
  --output-path data/processed/llm_extractions/parallel_plan_preview.csv \
  --provider openrouter \
  --model openai/gpt-4o-mini \
  --schema-version v3 \
  --max-concurrent-api-calls 5 \
  --checkpoint-every 25 \
  --progress-every 25 \
  --max-api-retries 3 \
  --api-retry-backoff-seconds 5,15,45 \
  --export-selection-manifest data/processed/llm_extractions/remaining_selection_manifest.csv \
  --dry-run \
  --force-overwrite
```

Recommended real manifest run:
```bash
python src/llm/extract_stock_contexts.py \
  --input-path data/processed/stock_mention_contexts_v2_1_kept.csv \
  --input-manifest-path data/processed/llm_extractions/remaining_selection_manifest.csv \
  --output-path data/processed/llm_extractions/openrouter_gpt4omini_v3_remaining_parallel.csv \
  --provider openrouter \
  --model openai/gpt-4o-mini \
  --schema-version v3 \
  --max-concurrent-api-calls 5 \
  --checkpoint-every 25 \
  --progress-every 25 \
  --max-api-retries 3 \
  --api-retry-backoff-seconds 5,15,45 \
  --force-overwrite
```

## Verified Parallel Tests
Safe dry-run with concurrency:
```bash
python src/llm/extract_stock_contexts.py \
  --input-path data/processed/stock_mention_contexts_v2_1_kept.csv \
  --output-path tmp/parallel_dry_run_test.csv \
  --provider openrouter \
  --model openai/gpt-4o-mini \
  --schema-version v3 \
  --limit 20 \
  --random-sample \
  --seed 123 \
  --max-concurrent-api-calls 5 \
  --checkpoint-every 5 \
  --dry-run \
  --force-overwrite
```

Tiny real parallel validation:
```bash
python src/llm/extract_stock_contexts.py \
  --input-path data/processed/stock_mention_contexts_v2_1_kept.csv \
  --output-path tmp/parallel_real_api_test.csv \
  --provider openrouter \
  --model openai/gpt-4o-mini \
  --schema-version v3 \
  --limit 10 \
  --random-sample \
  --seed 202606 \
  --max-concurrent-api-calls 3 \
  --checkpoint-every 5 \
  --progress-every 2 \
  --max-api-retries 3 \
  --api-retry-backoff-seconds 5,15,45 \
  --max-run-cost-usd 1 \
  --force-overwrite
```

Cache-hit rerun validation:
```bash
python src/llm/extract_stock_contexts.py \
  --input-path data/processed/stock_mention_contexts_v2_1_kept.csv \
  --output-path tmp/parallel_real_api_test_rerun.csv \
  --provider openrouter \
  --model openai/gpt-4o-mini \
  --schema-version v3 \
  --limit 10 \
  --random-sample \
  --seed 202606 \
  --max-concurrent-api-calls 3 \
  --checkpoint-every 5 \
  --progress-every 2 \
  --max-api-retries 3 \
  --api-retry-backoff-seconds 5,15,45 \
  --max-run-cost-usd 1 \
  --force-overwrite
```

Expected rerun behavior:
- cache hits should cover the already extracted rows
- API calls planned should fall to `0`
- total new cost should stay `0`
- rows appended to master should stay `0`
