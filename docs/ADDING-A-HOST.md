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
| `fs:/rootfs` | free space on the host's root — `/rootfs`, not `/`, on a Docker agent (see below) |
| `network:eth0` | throughput on one interface |

## Troubleshooting

**Temperature shows nothing, but the host clearly has sensors.** The dashboard matches sensor
labels with a *prefix* list — `cpu_thermal`, `Core`, `Tctl`, `Temperature` — which covers
Intel and AMD. ARM boards name theirs differently: a Rockchip board reports `soc_thermal 0`,
`bigcore0_thermal 0`; a Raspberry Pi reports `cpu_thermal 0` (that one matches). Check what
the agent actually reports, then add the prefix to the header widget:

```bash
curl -s http://HOST:61208/api/4/sensors | grep -o '"label":"[^"]*"'
```

```yaml
- glances:
    url: http://HOST:61208
    version: 4
    cputemp: true
    cpuSensorLabel: soc_thermal     # the prefix your board uses
```

**Disk shows nothing on a host running the Docker agent.** A container only sees its own
mount namespace. The agent compose bind-mounts the host root read-only at `/rootfs`, and the
widget must be told that name — `disk: /rootfs` in the header, or `metric: fs:/rootfs` on a
tile. `disk: /` names the container's own overlay and matches nothing. Hosts running the
package agent see the real `/` and can use it directly.


**A tile shows an error badge.** The agent is not reachable. From the dashboard host,
`curl http://HOST:61208/api/4/cpu`. A timeout means a firewall or the agent is not running.

**The tile is missing entirely, with nothing in the logs.** Usually an unknown `type:` in the
widget block — Homepage drops the whole service silently. Check the spelling against
<https://gethomepage.dev/widgets/>.

**Do not diagnose by viewing source.** The dashboard is client-rendered, so a tile absent
from the served HTML may be perfectly present. Use `/api/services` to check presence.
