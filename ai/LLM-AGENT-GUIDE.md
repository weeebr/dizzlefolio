## LLM Agent Guide – Dizzlefolio

Purpose: Enable an LLM coding agent in Cursor to implement changes with maximum accuracy, minimal risk, and strong maintainability.

### 1) Mindset & guardrails
- Be surgical: smallest change in the correct place; follow existing patterns.
- Push back if: intent may be misinterpreted; structural changes required; or critical details are missing. Ask 2–3 concise questions before editing.
- Never disable validation, authorization, queues, caching, or features to force a pass.

### 2) Architecture map (what matters most)
- Framework: Laravel 11 (PHP 8.3), Blade/Livewire UI, Vite frontend.
- Core domain models: `Portfolio`, `Holding`, `Transaction`, `Dividend`, `MarketData`, `Currency`, `CurrencyRate`, `DailyChange`.
- Market data abstraction: `Interfaces/MarketData/*` with `MarketDataInterface`; the app binds it to `FallbackInterface` which tries providers in `config/investbrain.php` (`provider` env order).
- Currency: Base-currency casting via `Casts/BaseCurrency.php` and historic FX via `CurrencyRate` (Frankfurter).
- Side effects are pipelined on model events (e.g., `Transaction` saving/saved), and heavy recompute lives behind Jobs/Artisan commands.
- API uses `Http/ApiControllers` + `Requests` for validation + `Resources` for shape; auth via `Policies`.

### 3) Critical contracts & side effects (treat as invariants)
- Market data
  - `AppServiceProvider` binds `MarketDataInterface` → `FallbackInterface` (do not change lightly).
  - `MarketData::getMarketData($symbol, $force=false)` hydrates and refreshes based on `config('investbrain.refresh')` and fills via provider `quote()`.
  - Providers implement: `exists`, `quote`, `dividends`, `splits`, `history` with consistent types in `Interfaces/MarketData/Types/*`.
- Transaction lifecycle (high-risk):
  - saving pipeline: `ConvertToMarketDataCurrency` → `EnsureCostBasisAddedToSale` → `CopyToBaseCurrency`.
  - saved: `syncToHolding()` then `EnsureDailyChangeIsSynced` (deferred daily change syncs).
  - `syncToHolding()` creates/updates holding, recalculates quantity, averages, realized gains.
- Currency & rates
  - `Currency::convert` uses `CurrencyRate::historic` for a date; base currency is `config('investbrain.base_currency')`.
  - Aliases (e.g., `GBX→GBP`) handled via `config('investbrain.currency_aliases')`.
  - `CurrencyRate::timeSeriesRates` populates ranges and uses `QueuedCurrencyRateInsertJob`.
- Holdings & performance
  - `Holding::withPerformance()` and `::withPortfolioMetrics()` compute display metrics using current market data and FX.
  - `Holding::dailyPerformance()` builds time series (CTE); SQL differs across drivers (SQLite/MySQL/PG) – be careful with grammar.
- Portfolio daily change
  - `Portfolio::syncDailyChanges()` is heavy: loops holdings, merges provider `history()` with positions and FX to write `daily_change` (composite key, see `HasCompositePrimaryKey`).
- Import/Export
  - Exports in `app/Exports/*`.
  - Imports in `app/Imports/*`, posting `BackupImport` job; `AfterImport` chains refresh, dividends, sync holdings, sync daily change. Also notifies user (email).
- Validation & auth
  - Requests in `Http/Requests/*`; `SymbolValidationRule` checks `MarketDataInterface::exists` (triggers provider call; consider rate-limits).
  - `PortfolioPolicy` governs access; controllers call `Gate::authorize`.
- Localization
  - `LocalizationMiddleware` sets locale and numeric formatting per user options.

### 4) High‑risk areas and how to avoid mistakes
- Do not short-circuit or rewrite `FallbackInterface` logic; confirm `config('investbrain.provider')` order instead.
- Avoid synchronous heavy recompute in web paths (e.g., calling `syncDailyChanges()` directly). Use jobs/Artisan commands.
- Preserve model event pipelines; if modifying save logic, keep the pipeline order and outputs intact (esp. currency casts).
- FX & dates: `CurrencyRate::historic` depends on date; weekend/holiday gaps are handled by nearest past date logic. Do not remove.
- Database grammar differences: in metrics/time-series SQL, code switches per driver; edits must keep all branches working (SQLite tests, MySQL, PG conditions).
- `DailyChange` uses composite PK; do not convert to single key or break `setKeysForSaveQuery` in `HasCompositePrimaryKey`.
- Validation/authorization must remain enforced in controllers; do not comment out `Gate::authorize` or relax request rules.
- Provider specifics:
  - Alpaca is USD-only (don’t assume multi-currency there).
  - AlphaVantage: symbol search semantics; ETF vs equity mapping; watch fields.
  - Yahoo uses caching and needs User-Agent headers (already set by client factory).

### 5) Surgical playbook (how to implement changes)
1) Clarify the task & pushback if risky/ambiguous.
2) Find the entry point:
   - Routes in `routes/{api,web}.php` → controller in `Http/…Controllers`.
   - Validate with `Http/Requests/*` and rules; check `Policies` usage.
   - Data shape in `Http/Resources/*`; domain logic in `Models/*` or `Actions/*`.
3) Plan minimal edits matching existing patterns; avoid new abstractions unless identical pattern exists.
4) Implement:
   - Adjust/extend request rules or small controller edits first.
   - Keep pipelines intact; if adding steps, ensure type and cast correctness.
   - For provider changes, implement in provider class without breaking interface contract; leave `FallbackInterface` untouched.
5) Tests:
   - Add or adjust PHPUnit tests mirroring existing style (`tests/*`).
   - Run `php artisan test` (SQLite in-memory; queue array driver).
6) Verify runtime aspects only if necessary:
   - Use targeted Artisan commands (e.g., `refresh:market-data`, `sync:holdings`) with narrow scope (`--user`, specific portfolio) to avoid heavy ops.
7) Review & cleanup:
   - Remove dead code if you changed approach in-branch.
   - Ensure API resource shapes remain backward-compatible unless instructed.

### 6) Verification checklist (pre-commit)
- Request rules enforce correct constraints; edge cases covered.
- Controllers still call `Gate::authorize` appropriately.
- Model pipelines and casts run in correct order (currency values correct in base and market currency). 
- No synchronous heavy work added to hot paths.
- Providers not hardwired; `config('investbrain.provider')` order respected.
- SQLite/MySQL/PG SQL branches still valid (especially date/CTE syntax).
- Tests pass locally: `php artisan test`.
- If UI changed, `npm run build` (Vite) and Blade compiles.
- No secrets or env assumptions leaked.

### 7) Observability & safe commands
- Confirm working directory before commands.
- Tests: `php artisan test`
- Refresh a single user’s data safely:
  - `php artisan refresh:market-data --user=<id> --force`
  - `php artisan refresh:dividend-data --user=<id> --force`
  - `php artisan sync:holdings --user=<id>`
  - For a single portfolio history: `php artisan sync:daily-change <portfolio_id> --force`
- Logs (Docker): `docker exec -it investbrain-app tail -f storage/logs/laravel.log`

### 8) Where to put what (file placement)
- Validation: `app/Http/Requests/*`, fine-grained rules in `app/Rules/*`.
- Authorization: `app/Policies/*` and controller `Gate::authorize`.
- Domain logic: `app/Models/*`; cross-cutting steps in `app/Actions/*`.
- API surface: `app/Http/Resources/*`.
- Market data providers: `app/Interfaces/MarketData/*`.
- Heavy/background: `app/Jobs/*`, `app/Console/Commands/*`.
- UI: `resources/views/*` and Livewire components.

### 9) Known pitfalls (quick reference)
- Breaking composite keys for `DailyChange`.
- Editing time-series SQL without updating all DB driver branches.
- Removing `unset($model->currency)` in `ConvertToMarketDataCurrency` (used to prevent persisting wrong field).
- Bypassing `SymbolValidationRule` or `PortfolioPolicy`.
- Calling provider endpoints in loops during tests (rate-limit risk); mock or keep to minimum.
- Running heavy sync commands without scoping (`--user`/portfolio) in development.

### 10) Small glossary of key functions
- `MarketData::getMarketData(symbol, force=false)` → returns/refreshes `MarketData` row.
- `FallbackInterface::__call(method, args)` → tries providers per config; logs failures; throws if none succeed.
- `Currency::convert(value, from, to?, date?)` → converts using historic rate; default `to` is base currency.
- `CurrencyRate::historic(currency, date?)` → returns single FX rate; fills DB if missing.
- `Portfolio::syncDailyChanges()` → recomputes time series of total market value in base currency.
- `Holding::syncTransactionsAndDividends()` → recomputes holding quantities/costs/realized gains.

Adhering to this guide maximizes success and minimizes risk of regressions.




### Addendum — Dizzlefolio operational specifics (only what's missing)

- **Env & secrets workflow**
  - Never commit secrets. `.env` and variants are ignored by git. When introducing a new env var, add it to `.env.example` with a safe placeholder and document its purpose and expected format.
  - Do not hardcode provider names or keys. Read from config, e.g., `config('investbrain.provider')`, and surface new configuration via `config/*.php` as needed.
  - If adding any secret-backed feature, include minimal setup notes in `README.md` or `ai/PROJECT.md` and ensure `.gitignore` already covers the file types involved.

- **Runtime & commands**
  - Prefer Docker Compose for running the app locally; tests run fine without Docker using SQLite in‑memory. Choose the least risky runtime for the task.
  - Use non‑interactive flags in commands (e.g., `--yes`, `--force`) and avoid long foreground processes. Keep retries bounded with short delays rather than long sleeps.
  - Before running commands, confirm the current working directory is the repository root and scope commands narrowly.

- **DB & schema discipline**
  - Tests rely on SQLite in‑memory; keep SQL branches valid for SQLite/MySQL/Postgres.
  - Do not introduce schema/migration changes without explicit approval. If required, add migrations, update casts/keys, and cover with tests.

- **API and naming preferences**
  - For new public endpoints, prefer verbs: use `get`/`fetch` for retrieval and `write`/`update` for modifications. Avoid `sync` in public API names (it is acceptable for internal Artisan commands where already established).

- **File placement and structure**
  - Match existing locations in “Where to put what.” When adding new files, keep structure flat and consistent; avoid introducing new nested folder patterns without precedent.

- **Style and commits**
  - Run Laravel Pint locally (`vendor/bin/pint`) to keep styling consistent with `pint.json`.
  - Use Conventional Commits (`feat:`, `fix:`, `refactor:`, `chore:`, `docs:`, `test:`, `perf:`). Keep commits small, with clear what/why/impact.

- **Final handover checklist additions**
  - Any new env vars added to `.env.example` and documented.
  - Commands are non‑interactive and bounded; long‑running work moved to Jobs/Artisan.
  - Public API names follow the verb conventions above.
