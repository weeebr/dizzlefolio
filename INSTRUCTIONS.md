## Cursor Project Rules – Dizzlefolio

### Core mission
- Make surgical, minimal edits that preserve integrity, structure, and tests.
- Never disable or remove features to “force” a solution.

### Pushback protocol
Before coding, evaluate:
- Intent mismatch with the user’s actual goal
- Risky structural changes (routing, DB schema, provider abstractions)
- Missing critical details (inputs, validation, auth, data flow)
If any apply: pause, list 2–3 concrete risks, ask concise questions, proceed only after alignment.

### Context & environment
- Confirm context before commands:
  - Working dir: repository root
  - Runtime: Docker Compose vs local; prefer documented Docker flows for app runtime
  - DB: Tests use SQLite in-memory; don’t assume Postgres
- Never expose/alter secrets. Use `.env.example` patterns and provider configs safely.

### Codebase navigation
- Trace feature flow: Requests → Policies/Gates → Controllers → Models → Resources → Views.
- Check `AppServiceProvider` for key bindings (e.g., market data interface).
- Inspect model pipelines and events (`saving/saved/deleted`) for side-effects.
- Use tests in `tests/` to infer behavior and contracts.

### Scope & organization
- Match existing patterns:
  - Validation: `app/Http/Requests/*` and `app/Rules/*`
  - Business/domain logic: `app/Models/*` or `app/Actions/*`
  - API responses: `app/Http/Resources/*`
  - AuthZ: `Policies` + `Gate::authorize(...)`
  - Background work: `Jobs` and `Console/Commands`
  - Market data: respect `Interfaces/MarketData/*` and `FallbackInterface`
- If changing direction, remove unused code and dead imports in the same PR.

### Don’ts
- Don’t disable policies, validation, queues, or caching to pass tests.
- Don’t hardwire to a single market data provider; respect `config('investbrain.provider')`.
- Don’t alter DB schema/migrations without explicit approval.

### Implementation workflow
- Pre-flight: read adjacent implementations; plan minimal edits.
- Development:
  - Type-safe PHP, respect casts (`BaseCurrency`, etc.)
  - Maintain model pipelines and event side-effects
  - Keep changes small and focused
- Post-change:
  - Run tests; ensure they pass locally/Docker
  - If FE changed, ensure Vite build works
  - Review logs only as needed; avoid noisy commands

### Efficient diagnostics
- Prefer targeted file reads over broad logs.
- For behavior, run specific artisan commands with `--help` or narrow scopes.
- Provider issues: remember `FallbackInterface` handles sequencing and logging; validate provider names in config.

### Testing & quality
- Use existing tests as contracts; add new ones for new behavior.
- Keep SQLite + array queue assumptions for tests.

### Performance & jobs
- Heavy recomputations (daily changes, large imports) should be jobs/commands, not web requests.
- Avoid long synchronous loops; queue work where appropriate.

### API & UX
- Shape responses via `Resources/*`; don’t leak internals.
- Validation errors should be precise and consistent with existing style.

### Command usage
- Use non-interactive, safe commands (examples):
  - `php artisan test`
  - `php artisan refresh:market-data --force`
- Avoid long-running foreground tasks; background if necessary and tail logs briefly.

### Code removal
- When course-correcting, remove obsolete code introduced earlier in the branch.

### Commits
- Keep commits atomic with clear “what/why/impact” messages.


