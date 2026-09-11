# Adding a host

Two steps: run the agent on the new machine, then add it to the dashboard.

## 1. Run the agent

**If the host runs Docker:**

```bash
mkdir -p ~/rack-dashboard-agent && cd ~/rack-dashboard-agent
curl -fsSLO https://raw.githubusercontent.com/Thalifergo-Git/rack-dashboard/main/agent/docker-compose.yml
docker compose up -d
```

**If it does not** — a Proxmox hypervisor, a bare VM, anything where you would rather not
install Docker just for this:

```bash
sudo apt-get install -y glances
sudo curl -fsSL -o /etc/systemd/system/glances-web.service \
     https://raw.githubusercontent.com/Thalifergo-Git/rack-dashboard/main/agent/glances-web.service
sudo systemctl daemon-reload && sudo systemctl enable --now glances-web
```

Check it from the dashboard host — this must return JSON, not a timeout:

```bash
curl -s http://NEW_HOST_IP:61208/api/4/cpu
```

## 2. Add it to the dashboard

In `config/services.yaml`, under `- Performance:`:

```yaml
    - myhost:
        icon: mdi-server
        description: cpu / ram / load
        widget:
          type: glances
          url: http://NEW_HOST_IP:61208
          version: 4
          metric: info
    - myhost — activity:
        icon: mdi-chart-line
        description: what is burning it
        widget:
          type: glances
          url: http://NEW_HOST_IP:61208
          version: 4
          metric: process
```

Homepage hot-reloads the config; no restart needed.

## Other metrics

Swap `metric:` for any of these — one widget each:

| `metric:` | Shows |
|---|---|
| `info` | CPU, RAM, load — the compact block |
| `process` | top processes by CPU — the activity monitor |
| `cpu` / `memory` | that one resource, larger |
| `sensors` | temperatures |
| `fs:/mnt/tank` | free space on one filesystem |
| `network:eth0` | throughput on one interface |

## Troubleshooting

**A tile shows an error badge.** The agent is not reachable. From the dashboard host,
`curl http://HOST:61208/api/4/cpu`. A timeout means a firewall or the agent is not running.

**The tile is missing entirely, with nothing in the logs.** Usually an unknown `type:` in the
widget block — Homepage drops the whole service silently. Check the spelling against
<https://gethomepage.dev/widgets/>.

**Do not diagnose by viewing source.** The dashboard is client-rendered, so a tile absent
from the served HTML may be perfectly present. Use `/api/services` to check presence.
