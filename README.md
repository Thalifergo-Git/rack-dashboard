# Rack Dashboard

**One pane of glass for a homelab rack** — every host's CPU, RAM, temperature and
*which process is actually burning it*, plus your latest alerts, on a single page.

*A multi-host configuration layer for [Homepage](https://github.com/gethomepage/homepage)
and [Glances](https://github.com/nicolargo/glances). Not a fork — see
[what this is](#what-this-is-and-what-it-is-not).*

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

## What this is, and what it is not

**This is not a new dashboard, and it is not a fork.** It contains no upstream source code.
It is a *configuration and deployment layer* that stands on three existing projects:

| Project | Licence | What it does here |
|---|---|---|
| **[Homepage](https://github.com/gethomepage/homepage)** | GPL-3.0 | The dashboard itself. All rendering, widgets and the service catalogue. |
| **[Glances](https://github.com/nicolargo/glances)** | LGPL-3.0 | The per-host agent. Supplies CPU, RAM, temperature and the process list. |
| **[docker-socket-proxy](https://github.com/Tecnativa/docker-socket-proxy)** | Apache-2.0 | Read-only Docker API, so the dashboard never touches the raw socket. |

All the credit for the hard parts belongs upstream. Homepage is doing the work you can see;
this repo is the opinionated wiring around it. If you want the dashboard on its own, go
straight to [gethomepage.dev](https://gethomepage.dev) — you do not need this project.

### What this project actually adds

The gap it fills is that Homepage, by default, describes **the machine it runs on**. A rack
is several machines, and some of them — an appliance with its own walled-garden dashboard, a
hypervisor with none — will never report into it on their own.

1. **A multi-host Performance layer.** One `info` tile and one `process` tile per machine,
   so the whole rack is on one page instead of one box being the star.
2. **A process-level activity monitor.** The reason the project exists. `node-exporter` has
   no per-process metrics and cAdvisor only sees containers, so anything running *outside* a
   container is invisible to a normal metrics stack. This shows what is actually burning
   each box, by process name.
3. **Agents for hosts without Docker.** A systemd unit alongside the container, because a
   Proxmox hypervisor should not have to install Docker just to be monitored.
4. **An onboarding script.** `setup.sh` asks what is in your rack, generates the config, and
   *prints* the commands for the other machines rather than reaching into them over SSH.
5. **A latest-alerts strip.** Reads an existing Alertmanager if you have one, skipped if not.
6. **Secret handling that survives being committed.** Config references
   `{{HOMEPAGE_VAR_*}}` placeholders; values live in a gitignored `.env`.
7. **Security defaults that are not the tutorial defaults.** No Docker socket mounted into
   the dashboard, Glances serving its API with the human web UI disabled, and the warning
   about port 61208 being unauthenticated stated where you will actually read it.

### Licence scope

The MIT licence in this repo covers **only the files in this repo** — the compose files,
`setup.sh`, the config templates and the docs. Homepage, Glances and docker-socket-proxy are
pulled as published images at run time and remain under their own licences above. Nothing
here redistributes or modifies their code.


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
