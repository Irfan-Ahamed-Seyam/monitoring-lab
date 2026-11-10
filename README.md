# Monitoring Lab: Prometheus, Grafana, Alertmanager, Zabbix

A local NOC/SOC-style monitoring stack in Docker Compose. It watches a Linux host and HTTP endpoints, alerts on CPU, memory, disk and availability, and shows the same host in both Prometheus/Grafana and Zabbix.

## What's in it
| Component | Role | URL |
|---|---|---|
| Prometheus | Pull-based metrics collection + alert rules | http://localhost:9090 |
| node-exporter | Linux host metrics (CPU, RAM, disk, network) | scraped by Prometheus |
| blackbox-exporter | HTTP uptime and latency checks (SLA-style) | scraped by Prometheus |
| Alertmanager | Groups and routes alerts | http://localhost:9093 |
| Grafana | Dashboards (Prometheus + Zabbix data sources) | http://localhost:3000 (admin/admin, lab default) |
| Zabbix 7.0 LTS | Agent-based monitoring with triggers, PostgreSQL backend | http://localhost:8080 (Admin/zabbix) |
| zabbix-agent2 | Monitored host `lab-linux-01` | |

> Credentials in this repo are local lab defaults. Change them before exposing anything beyond localhost.

## Run
```bash
docker compose up -d
docker compose ps
```

## Alerts (prometheus/alerts.yml)
- HighCPU: > 80% for 1m
- HighMemory: > 90% for 2m
- DiskAlmostFull: > 85% for 5m
- EndpointDown: blackbox probe failing for 1m
- SlowResponse: response time > 2s
- ScrapeTargetDown: Prometheus can't reach an exporter

## Incident drills
1. **CPU spike:** `docker run --rm -it --cpus=8 alpine sh -c "apk add -q stress-ng && stress-ng --cpu 8 --timeout 180s"` → HighCPU fires in Prometheus/Alertmanager, and the Zabbix "High CPU utilization" trigger fires.
2. **Service down:** `docker compose stop zabbix-web` → EndpointDown fires within ~1 minute; `docker compose start zabbix-web` → it resolves.
3. **Exporter down:** `docker compose stop node-exporter` → ScrapeTargetDown fires.

## Useful PromQL
```promql
100 - avg by (instance)(rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100   # CPU %
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100        # RAM %
probe_success                                                                  # endpoint up (1) / down (0)
probe_duration_seconds                                                         # response time
```

## Drill results (2026-11-10)
**Drill 1: CPU spike.** I ran stress-ng on 8 cores for 4 minutes.
- Prometheus HighCPU went pending, then fired at 99%. Alertmanager received it.
- Zabbix's default trigger ("min over 5m > 90%") did not fire, because the spike was shorter than 5 minutes. I added a custom trigger, `min(/lab-linux-01/system.cpu.util,1m)>80`, and it fired.
- Both alerts resolved on their own when the load stopped.

**Drill 2: Website outage, recovered by GitOps.** I ran `kubectl scale deploy/my-web-app --replicas=0` on the k3d cluster.
- At +5s the site returned HTTP 503, and at +7s the blackbox probe showed it down.
- At +13s Argo CD (selfHeal) had restored 2/2 pods and the site was back to HTTP 200. At +22s the probe confirmed it was up.
- EndpointDown went no further than pending, because the outage (~15s) was shorter than the 1-minute alert threshold. That's intended: short blips don't page anyone.

**Tuning: false positive.** DiskAlmostFull fired on `/usr/lib/wsl/drivers`, which is the Windows C: drive mounted into WSL over 9p at 89%. It isn't part of the monitored Linux host, so I excluded `9p|squashfs|nsfs` filesystems from the disk rule and the dashboard.

**Cleanup:** I disabled the default "Zabbix server" host. It expects an agent on 127.0.0.1 inside the server container, which doesn't exist in this setup.
