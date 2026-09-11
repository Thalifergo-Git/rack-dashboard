# Rack Dashboard

**One pane of glass for a homelab rack** — every host's CPU, RAM, temperature and
*which process is actually burning it*, plus your latest alerts, on a single page.

Built for the case where your machines don't agree with each other: a NAS appliance with
its own dashboard, a Proxmox hypervisor with none, and a Pi running everything else. Each
one shows you a slice. This shows you all of them.

```
                    ┌──────────────────────────────┐
   your browser ──> │        Homepage              │
                    │  ┌────────┬────────┬───────┐ │
                    │  │  NAS   │hypervsr│  Pi   │ │  cpu / ram / temp
                    │  ├────────┼────────┼───────┤ │
                    │  │ top    │ top    │ top   │ │  <- the activity monitor:
                    │  │ procs  │ procs  │ procs │ │     what is burning this box
                    │  └────────┴────────┴───────┘ │
                    │  [ latest alerts ]           │
                    └──────┬───────────┬───────────┘
                           │           │
                    Glances on      Alertmanager
                    every host      (optional)
                    :61208
```

## Why not just Prometheus + Grafana?

Use both — they answer different questions.

| Question | Tool |
|---|---|
| Was this host unhealthy last Tuesday? | Prometheus + Grafana |
| Is this host unhealthy *right now*? | either |
| **Which process is burning it right now?** | **only this** |

That last row is the gap this fills. `node-exporter` has no per-process metrics and
cAdvisor only sees containers — so anything running *outside* a container is invisible to a
normal metrics stack. The incident that motivated this project was a NAS pinned at 103 °C by
51 `mediainfo` processes spawned by the appliance's own file indexer. Not a container.
Invisible to every exporter running at the time.

## What you get

- **A tile per host** with CPU, RAM, load and temperature.
- **An activity monitor per host** — top processes by CPU, updated live.
- **A latest-alerts strip** if you already run Alertmanager (optional; skipped if not).
- **Service tiles** for whatever else you run, with live widgets for ~100 apps that Homepage
  supports out of the box.

## Requirements

- One host that will run the dashboard (Docker + Docker Compose).
- SSH access to each host you want to monitor.
- Docker on the monitored hosts, *or* the ability to install a package — both are supported.

## Quickstart

```bash
git clone https://github.com/Thalifergo-Git/rack-dashboard.git
cd rack-dashboard
./setup.sh
```

`setup.sh` asks for each host — name, address, and whether it runs Docker — then writes
`config/services.yaml` and brings the dashboard up. It does not touch the monitored hosts;
it prints the one command to run on each.

Then open `http://<dashboard-host>:3080`.

## Adding a host later

See [docs/ADDING-A-HOST.md](docs/ADDING-A-HOST.md). Short version: run the agent on the new
host, add six lines to `config/services.yaml`, done.

## Security notes, read them

- **Glances exposes a read-only REST API on port 61208 with no authentication.** It reveals
  process names, mounted filesystems and hostnames. **Bind it to your LAN or a VPN
  interface — never to the internet.** The provided units serve the API only, with the
  human web UI disabled.
- **This project never mounts `/var/run/docker.sock` into the dashboard.** That socket is
  root-equivalent on the host. Container stats come through a read-only socket proxy that
  publishes on loopback only.
- **API keys live in `.env`, which is gitignored.** `config/services.yaml` references them as
  `{{HOMEPAGE_VAR_*}}` placeholders, so the file you commit never contains a secret.

## Licence

MIT. See [LICENSE](LICENSE).
