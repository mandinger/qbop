# Copilot Instructions for qbop

Purpose: Help AI coding agents work productively in this repo by capturing architecture, workflows, and project-specific patterns. Keep changes minimal and aligned with current code.

## Big Picture
- App type: Rack-based Ruby app exposing a Sinatra web UI and a Grape JSON API, with background loops via SuckerPunch.
- Goal: Track ProtonVPN forwarded port (via `natpmpc`) and keep OPNsense alias and qBittorrent listen port in sync.
- State: SQLite via Sequel; auto-migrated on boot; simple stats/counters and notifications persisted under `data/qbop.sqlite3`.
- Entry: `config.ru` wires everything, runs DB migrations/seeding, mounts apps, and starts jobs.

## Architecture Layout
- `framework/`:
  - `web.rb` (Sinatra UI): routes `/`, `/logs`, `/about`, `/api-docs`, `/tools` (WireGuard pubkey + public IP helpers).
  - `api.rb` (Grape API under `/api`): `GET /stats`, `/about`, `/notifications`, `/health`.
- `jobs/`:
  - `qbop.rb`: main loop; gets Proton forwarded port; reconciles OPNsense alias and qBittorrent port with backoff (`REQUIRED_ATTEMPTS`).
  - `check_for_new_release.rb`: polls GitHub tags and stores `Notification` for UI/health.
- `service/`:
  - `proton.rb`: runs `natpmpc`, parses forwarded port.
  - `opnsense.rb`: Faraday client against OPNsense API; updates alias and applies changes.
  - `qbit.rb`: qBittorrent Web API (login, read/update `listen_port`).
  - `helpers.rb`: env parsing, time math, logging, file helpers (log/Gemfile), GitHub tag check, shell utilities.
  - `seed.rb`: initializes baseline rows for each `Source`.
- `models/` (Sequel): `Source` (has `Stat`, `Counter`), `Stat`, `Counter`, `Notification`.
- `db/migrate/`:
  - `001_start.rb`: tables for `sources`, `stats`, `counters`, `notifications`.

## Boot Flow (important)
- `config.ru` order is intentional:
  1) require frameworks/jobs/services
  2) enable Sequel plugin `update_or_create`
  3) `rake db:migrate` (runs all migrations)
  4) connect DB, load models, seed default rows
  5) mount `Framework::Web` and `Framework::API` via `Rack::Cascade`
  6) kick off jobs: `Qbop.perform_async`, `CheckForNewReleases.perform_async`
- Do not block the event loop in web requests; all long-running work goes to SuckerPunch jobs.

## Configuration & Env
- Preferred envs (see `.env.example`): `UI_MODE`, `LOOP_FREQ`, `REQUIRED_ATTEMPTS`, `PROTON_GATEWAY`, `OPN_*`, `QBIT_*`, `LOG_*`.
- Booleans are string-based; use `Service::Helpers#true?` when interpreting env values.
- Logging uses `Service::Helpers#logger_instance`; defaults to `log/qbop.log`, or STDOUT if `LOG_TO_STDOUT=true`.

## Developer Workflows
- Install deps: `bundle install`
- Run (local): `bundle exec puma -p 4567` (picks up `config.ru`; migrations run automatically)
- Migrate manually: `bundle exec rake db:migrate`
- Tests: `bundle exec rspec` (note: current specs are minimal and may be stale)
- Lint: `bundle exec rubocop`
- Docker (PowerShell):
  - `$version = Get-Content -Raw version.yml`
  - `docker build --build-arg VERSION=$version -t qbop:dev .`
  - `docker compose -f docker-compose/docker-compose.yml up -d`

## Patterns & Conventions
- Use `Service::Helpers.env_variables` once per job/flow and pass config around; do not scatter `ENV` lookups.
- Use `Service::Helpers#logger_instance` for consistent logging and to honor `LOG_TO_STDOUT`.
- Data access via Sequel models; prefer existing helper methods on `Source` for updating stats/counters.
- API/UI read stats via `Stat.as_hash` and `Notification` lookups; keep shapes stable when adding fields.
- External HTTP via Faraday; disable TLS verify only where already done (OPNsense/qBit), and keep headers consistent.
- Shelling out: use `Open3.capture3` as in `helpers.rb` and `proton.rb`; ensure timeouts are respected.

## Extending Safely
- New background task: add job in `jobs/`, require it via `config.ru`, and start with `perform_async` after apps mount.
- New DB fields: create a new migration under `db/migrate/`, rely on boot-time migrate, and extend models minimally.
- New integration: mirror `service/*` pattern (Faraday client + small surface), wire via `jobs/qbop.rb` with backoff and counters.
- UI/API changes: keep Sinatra templates in `views/` and JSON under Grape routes prefixed with `/api`.

## Gotchas
- Proton parsing expects `natpmpc` output containing "Mapped public port".
- Port updates gate on `REQUIRED_ATTEMPTS` and valid range 1024–65535.
- `VERSION` is shown in UI/API; in Docker it’s injected via build arg/env; locally it may be unset.
- Tests under `spec/` do not match some current helper names; update or add new specs if you change helpers.
