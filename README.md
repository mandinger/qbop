# qbop
A tool for maintaining a forwarded port from ProtonVPN, while optionally keeping OPNsense and qBittorrent in sync. The tool offers a simple web UI and API via `http://<host_ip>:4567/`.

qbop is built with Ruby and available as a Docker image.

## Installation
I recommend using the provided sample Docker Compose files to simplify the set up of qbop. This container must be routed through ProtonVPN due to the required `natpmpc` dependency.

You can ignore OPNsense and/or qBittorrent by using the `OPN_SKIP` and `QBIT_SKIP` environment variables.

The container image is available [here](https://github.com/clajiness/qbop/pkgs/container/qbop). The sample docker-compose.yml file is available [here](https://github.com/clajiness/qbop/blob/main/docker-compose/docker-compose.yml).

There is also an unmaintained (by me) [community compose directory](https://github.com/clajiness/qbop/blob/main/docker-compose/community/). Feel free to open a pull request to share your own compose files.

### Requirements
* AMD64 or ARM64/v8 architecture - If you need support for a different architecture, file an issue.
* [Docker Engine](https://docs.docker.com/engine/install/)
* [OPNsense](https://docs.opnsense.org/)
    * [Selective routing](https://docs.opnsense.org/manual/how-tos/wireguard-selective-routing.html)
    * [API](https://docs.opnsense.org/development/how-tos/api.html)
* [qBittorrent](https://www.qbittorrent.org/)
* [ProtonVPN](https://protonvpn.com/support/port-forwarding)

### ENV variables
| Variable | Default | Description |
| :--- | :--- | :--- |
| `UI_MODE` | `dark` | [`dark`/`light`] This value sets the UI mode of the web app. The default value is `dark`. |
| `LOOP_FREQ` | `45` | This value, in seconds, determines how often the job runs. It must be a positive integer. The default value is recommended by ProtonVPN. |
| `REQUIRED_ATTEMPTS` | `3` | The number of loops with a new forwarded port before updating OPNsense and qBit. The min is 1, and max is 10. |
| `LOG_LINES` | `50` | The number of log lines displayed on the "logs" page |
| `LOG_REVERSE` | `false` | Reverse the display order of log lines, showing newest logs at the top when enabled. |
| `LOG_TO_STDOUT` | `false` | Log to STDOUT instead of the default log directory |
| `PROTON_GATEWAY` | `10.2.0.1` | ProtonVPN provided gateway IP address. Do not use `http(s)://` or a trailing slash. |
| `OPN_SKIP` | `false` | [`true`/`false`] Skip OPNsense. If `true`, subsequent OPNsense environment variables are not required. |
| `OPN_INTERFACE_ADDR` | | OPNsense Interface Address. Requires `http(s)://` and no trailing slash. |
| `OPN_API_KEY` | | OPNsense API Key |
| `OPN_API_SECRET` | | OPNsense API Secret |
| `OPN_PROTON_ALIAS_NAME` | | The firewall alias that you use for ProtonVPN's forwarded port. For example, `proton_vpn_forwarded_port`. |
| `QBIT_SKIP` | `false` | [`true`/`false`] Skip qBittorrent. If `true`, subsequent qBit environment variables are not required. |
| `QBIT_ADDR` | | The IP address of your qBittorrent app. Requires `http(s)://` and no trailing slash. |
| `QBIT_USER` | | qBittorrent username |
| `QBIT_PASS` | | qBittorrent password |

## Local Development

The app is a Rack service (Sinatra UI + Grape API) with background jobs. You can run it either in Docker (closest to prod) or directly on your host for quick UI/API iteration.

### Option A — Docker (recommended)
- PowerShell
    - `$version = Get-Content -Raw version.yml`
    - `docker build --build-arg VERSION=$version -t qbop:dev .`
    - `docker compose -f docker-compose/docker-compose.yml up -d`
- UI: http://localhost:4567

### Option B — Host runtime (quick start)
Prereqs: Ruby 3.4.x, Bundler, SQLite3. On Windows, RubyInstaller + MSYS2 works well. For Proton/NAT-PMP tools, prefer WSL2 or Docker; host run will log errors but still serve the UI.

1) Install deps
- `bundle install`

2) Create data/log folders (SQLite DB and logs)
- `mkdir data`
- `mkdir log`

3) Set minimal env for local dev (PowerShell examples)
- `$env:OPN_SKIP = 'true'`  # skip OPNsense integration
- `$env:QBIT_SKIP = 'true'` # skip qBittorrent integration
- `$env:LOG_TO_STDOUT = 'true'`
- `$env:LOOP_FREQ = '60'`   # reduce background loop noise
- `$env:UI_MODE = 'dark'`
- `$env:PROTON_GATEWAY = '10.2.0.1'`

4) Run the app
- `bundle exec puma -p 4567`  # loads config.ru, runs migrations automatically
- UI: http://localhost:4567

Notes
- DB migrations auto-run at boot. You can also run `bundle exec rake db:migrate` manually.
- Background jobs start automatically; without `natpmpc` installed they’ll log errors but won’t crash the server.

### Dev Utilities
- Tests: `bundle exec rspec`
- Lint: `bundle exec rubocop`
- Logs: `log/qbop.log` or STDOUT if `LOG_TO_STDOUT=true`
