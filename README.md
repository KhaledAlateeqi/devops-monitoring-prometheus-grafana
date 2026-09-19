# Prometheus & Grafana Monitoring Stack

![Prometheus](https://img.shields.io/badge/Prometheus-monitoring-E6522C)
![Grafana](https://img.shields.io/badge/Grafana-dashboards-F46800)
![Docker Compose](https://img.shields.io/badge/Docker%20Compose-ready-2496ED)

A ready-to-run **observability stack** using **Prometheus** for metrics collection, **Node Exporter** for host-level metrics, and **Grafana** for visualization and alerting — all orchestrated with a single `docker-compose.yml`.

---

## 1. Architecture & Metrics Flow

```
┌────────────────┐   scrapes every 15s    ┌──────────────┐   queries    ┌──────────┐
│  Node Exporter  │ ◄──────────────────────│  Prometheus   │◄─────────────│ Grafana  │
│  (host metrics: │      :9100/metrics     │  :9090        │   PromQL     │  :3000   │
│  CPU, memory,   │                        │               │              │          │
│  disk, network) │                        │  - Stores     │              │ Dashboards
└────────────────┘                        │    time-series│              │ & Alerts │
                                            │  - Evaluates  │              └──────────┘
                                            │    alert.rules│
                                            │    .yml       │
                                            └──────────────┘
```

**Flow explained:**
1. **Node Exporter** runs on the host (or as a container with host access) and exposes hardware/OS metrics (CPU, memory, disk, network) at `/metrics`.
2. **Prometheus** scrapes Node Exporter's `/metrics` endpoint every **15 seconds** (configured in `prometheus.yml`) and stores the time-series data. It also continuously evaluates the rules in `alert.rules.yml` (e.g., CPU > 80%).
3. **Grafana** connects to Prometheus as a data source (auto-provisioned) and renders the metrics as dashboards, letting you visualize trends and drill into specific time ranges.

---

## 2. Repository Structure

```
prometheus-grafana-monitoring/
├── docker-compose.yml                        # Prometheus + Node Exporter + Grafana
├── prometheus.yml                             # Scrape configuration (15s interval)
├── alert.rules.yml                             # CPU / memory / disk alert rules
└── grafana/
    └── provisioning/
        └── datasources/
            └── datasource.yml                  # Auto-provisions Prometheus in Grafana
```

---

## 3. Quickstart Guide

### Prerequisites
- Docker & Docker Compose installed

### Launch the stack
```bash
git clone https://github.com/YOUR_USERNAME/prometheus-grafana-monitoring.git
cd prometheus-grafana-monitoring

docker compose up -d
```

This starts three containers:
| Service        | Port  | Purpose                                  |
|----------------|-------|--------------------------------------------|
| `prometheus`   | 9090  | Metrics storage, scraping, and alert evaluation |
| `node_exporter`| 9100  | Exposes host-level metrics                |
| `grafana`      | 3000  | Dashboards and visualization              |

Check everything is running:
```bash
docker compose ps
```

Stop the stack:
```bash
docker compose down
```
(Add `-v` to also remove the persisted Prometheus/Grafana volumes.)

---

## 4. Accessing Grafana & Importing Dashboards

1. Open **http://localhost:3000** in your browser.
2. Log in with the default credentials:
   - **Username:** `admin`
   - **Password:** `admin`
   (You'll be prompted to change this on first login — recommended for anything beyond local testing.)
3. The **Prometheus data source is already auto-provisioned** (via `grafana/provisioning/datasources/datasource.yml`), so you don't need to add it manually.
4. Import a pre-built dashboard:
   - Go to **Dashboards → New → Import**
   - Enter dashboard ID **`1860`** ("Node Exporter Full" — one of the most widely used community dashboards) and click **Load**
   - Select **Prometheus** as the data source and click **Import**
5. You should immediately see live CPU, memory, disk, and network graphs for the host.

Other useful community dashboard IDs to try the same way:
| ID     | Dashboard                        |
|--------|-----------------------------------|
| 1860   | Node Exporter Full                |
| 405    | Node Exporter Server Metrics      |
| 11074  | Node Exporter for Prometheus Dashboard (simplified) |

---

## 5. Alerting Rules

`alert.rules.yml` defines the following alerts, evaluated every 15 seconds:

| Alert            | Condition                                      | Duration before firing |
|-------------------|--------------------------------------------------|--------------------------|
| `HighCpuUsage`    | CPU usage > 80%                                 | 2 minutes                |
| `HighMemoryUsage` | Memory usage > 80%                              | 2 minutes                |
| `HighDiskUsage`   | Disk usage > 85% on any real filesystem         | 5 minutes                |
| `InstanceDown`    | A scrape target is unreachable (`up == 0`)      | 1 minute                 |

View active/pending alerts in the Prometheus UI at **http://localhost:9090/alerts**.

> To actually receive notifications (Slack, email, PagerDuty, etc.), add an **Alertmanager** service to `docker-compose.yml` and point the `alerting.alertmanagers` section in `prometheus.yml` to it. This stack ships with the rules and evaluation logic; wiring up a notification channel is a natural next step.

---

## 6. Verification & Troubleshooting

### Verify Prometheus is scraping successfully
Go to **http://localhost:9090/targets** — both `prometheus` and `node_exporter` jobs should show state **UP**.

### Verify metrics are flowing
In the Prometheus UI (**http://localhost:9090/graph**), run a test query:
```
node_cpu_seconds_total
```
You should see time-series data returned.

### Common issues

| Symptom                               | Likely cause & fix |
|----------------------------------------|----------------------|
| `node_exporter` target shows **DOWN**  | Container failed to start — check `docker compose logs node_exporter`. On some systems you may need to adjust the `pid: host` / volume mounts for your OS. |
| Grafana shows "No data"                | Confirm the Prometheus data source URL is `http://prometheus:9090` (container-to-container DNS name, not `localhost`) — already set correctly in `datasource.yml`. |
| Port already in use (3000/9090/9100)   | Another process is using the port. Change the left-hand side of the port mapping in `docker-compose.yml`, e.g. `"3001:3000"`. |
| Alerts never fire                      | Check **http://localhost:9090/rules** to confirm rules loaded without syntax errors, and generate load (e.g. `stress --cpu 4`) to actually trigger the CPU alert. |
| Grafana admin password forgotten       | Reset via `docker compose exec grafana grafana-cli admin reset-admin-password newpassword` |

### Generate test load to trigger the CPU alert (Linux/macOS)
```bash
# Requires the `stress` package (apt install stress / brew install stress)
stress --cpu 4 --timeout 180s
```
Watch the `HighCpuUsage` alert transition from **Inactive → Pending → Firing** in the Prometheus UI as sustained load crosses the 80% threshold for 2+ minutes.

---

## 7. Possible Extensions

- Add **Alertmanager** for real notification routing (Slack/email/PagerDuty)
- Add **cAdvisor** to monitor Docker container-level metrics alongside host metrics
- Persist Grafana dashboards as code (JSON provisioning) instead of manual import
- Point this stack at the `fastapi-cicd-pipeline` project by adding a `/metrics` endpoint (e.g. via `prometheus-fastapi-instrumentator`) and scraping it as an additional job

---

## License
MIT — feel free to fork and adapt for your own portfolio.
