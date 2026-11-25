# 🍓 Raspberry Pi 5 Observability Stack

[![Docker Compose](https://img.shields.io/badge/Docker_Compose-v2-blue?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![Prometheus](https://img.shields.io/badge/Prometheus-v2-orange?logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-v10-F46800?logo=grafana&logoColor=white)](https://grafana.com/)
[![Alertmanager](https://img.shields.io/badge/Alertmanager-Telegram-0088cc?logo=telegram&logoColor=white)](https://prometheus.io/docs/alerting/latest/alertmanager/)
[![Platform](https://img.shields.io/badge/Platform-Raspberry_Pi_5_(aarch64)-C51A4A?logo=raspberrypi&logoColor=white)](https://www.raspberrypi.com/)

A production-grade, declarative **Observability-as-Code** stack designed specifically for edge Linux servers and **Raspberry Pi 5** home labs.

Provides end-to-end monitoring for host telemetry (including SoC temperature and active cooling fan RPM), Docker container resource usage via **cAdvisor**, synthetic HTTP probing via **Blackbox Exporter**, automated **Telegram** alerting, and pre-provisioned **Grafana** dashboards.

---

## 📐 Architecture Overview

```
                          ┌────────────────────────────┐
                          │   Raspberry Pi 5 (Host)    │
                          │  /sys /proc kernel hwmon   │
                          └─────────────┬──────────────┘
                                        │
                                        ▼
┌───────────────────────┐    ┌─────────────────────┐    ┌─────────────────────────┐
│     Docker Engine     │    │    Node Exporter    │    │    Target Services      │
│  cgroups v2 & metrics │    │  (Host net, :9100)  │    │  (Gitea, Apps, APIs...) │
└───────────┬───────────┘    └──────────┬──────────┘    └────────────┬────────────┘
            │                           │                            │
            ▼                           │                            ▼
┌───────────────────────┐               │               ┌─────────────────────────┐
│        cAdvisor       │               │               │    Blackbox Exporter    │
│  (Container metrics)  │               │               │    (HTTP/ICMP probes)   │
└───────────┬───────────┘               │               └────────────┬────────────┘
            │                           │                            │
            └───────────────────┐       │       ┌────────────────────┘
                                │       │       │
                                ▼       ▼       ▼
                        ┌───────────────────────────────┐
                        │      Prometheus Server        │
                        │   (Pull Model, Scrape: 15s)   │
                        └───────┬───────────────┬───────┘
                                │               │
                Rule Evaluation │               │ Metrics Query
                                ▼               ▼
                    ┌───────────────────┐   ┌───────────────────────────┐
                    │   Alertmanager    │   │          Grafana          │
                    │ (Deduplication)   │   │  (Dashboard Provisioning) │
                    └─────────┬─────────┘   └─────────────┬─────────────┘
                              │                           │
                              ▼                           ▼
                    ┌───────────────────┐   ┌───────────────────────────┐
                    │   Telegram Bot    │   │  Web UI (Port 3001)       │
                    │  (Incident Alerts)│   │  • RPi 5 Hardware Telemetry│
                    └───────────────────┘   │  • Docker Container Usage │
                                            └───────────────────────────┘
```

---

## ✨ Key Highlights

* **Observability as Code:** Dashboards, datasources, and alerting rules are fully defined in code. Running `docker compose up -d` brings up a complete, zero-touch configured environment.
* **Hardware-Specific Telemetry:** Scrapes Raspberry Pi 5 SoC thermal zones (`node_thermal_zone_temp`) and PWM cooling fan RPM (`node_hwmon_fan_rpm`).
* **Zero Hardcoded Host IPs:** Services communicate through Docker internal DNS and `host.docker.internal:host-gateway`. The stack works seamlessly whether the host is on Wi-Fi, Ethernet, DHCP, or Mesh VPNs (Tailscale).
* **Container Metrics via cAdvisor:** Detailed per-container CPU percentage, memory working sets, network I/O, and crash/restart detection.
* **Instant Incident Alerting:** Rich HTML alert templates routed to a Telegram channel/chat with automatic resolution notices (`RESOLVED`).

---

## 📂 Repository Structure

```
.
├── docker-compose.yml          # Container stack orchestration definition
├── .env.example                # Sample environment variables template
├── .gitignore                  # Prevents secrets & database mounts from leaking
├── alertmanager/
│   ├── alertmanager.yml        # Notification routes & Telegram formatting
│   └── telegram_token.example  # Bot token template
├── blackbox/
│   └── blackbox.yml            # HTTP 2xx and ICMP prober modules
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── prometheus.yml  # Auto-configures Prometheus as default datasource
│   │   └── dashboards/
│   │       └── dashboards.yml  # Auto-loads dashboard JSONs on startup
│   └── dashboards/
│       ├── rpi-hardware.json   # Raspberry Pi 5 CPU, Temp, Fan, Disk & RAM dashboard
│       └── docker-containers.json # cAdvisor container resource consumption
└── prometheus/
    ├── prometheus.yml          # Scrape configurations & service discovery targets
    └── rules/
        └── alerts.yml          # Alerting rules (CPU overload, Disk full, Thermal throttle)
```

---

## 🚀 Quick Start

### 1. Clone the repository
```bash
git clone https://github.com/<your-username>/rpi-observability-stack.git
cd rpi-observability-stack
```

### 2. Configure Environment & Secrets
Copy the sample environment file:
```bash
cp .env.example .env
```

If you wish to enable Telegram notifications:
1. Create a bot with [@BotFather](https://t.me/BotFather) and get your `BOT_TOKEN`.
2. Get your numeric `CHAT_ID` from [@userinfobot](https://t.me/userinfobot).
3. Save your token into `alertmanager/telegram_token`:
   ```bash
   echo "YOUR_BOT_TOKEN_HERE" > alertmanager/telegram_token
   chmod 600 alertmanager/telegram_token
   ```
4. Update `chat_id` inside `alertmanager/alertmanager.yml`.

### 3. Launch the Stack
```bash
docker compose up -d
```

### 4. Access the Dashboards
* **Grafana:** `http://<PI_IP>:3001` (Default login: `admin` / `admin`)
* **Prometheus:** `http://<PI_IP>:9090`
* **Alertmanager:** `http://<PI_IP>:9093`
* **cAdvisor:** `http://<PI_IP>:8080`
* **Uptime Kuma:** `http://<PI_IP>:3002`

---

## 📊 Pre-Provisioned Dashboards

1. **Raspberry Pi 5 - Hardware & OS Overview** (`rpi-hardware.json`):
   * **SoC Temperature Gauge:** Real-time BCM2712 temperature with dynamic color thresholds (<60°C Green, 60-75°C Yellow, >75°C Red).
   * **Cooling Fan RPM:** Speed sensor monitoring for Raspberry Pi active cooler.
   * **CPU & RAM Breakdown:** CPU usage history, RAM utilization (Active, Buffers, Cache, Free).
   * **Disk Storage & Network:** Real-time root disk space breakdown and network RX/TX bandwidth across `wlan0` / `eth0`.

2. **Docker Containers & cAdvisor Overview** (`docker-containers.json`):
   * Total active containers count.
   * Per-container CPU load (%) and memory usage in bytes.
   * Container network ingress/egress throughput.

---

## 🚨 Included Alerting Rules

| Alert Name | Condition | Severity | Description |
| :--- | :--- | :--- | :--- |
| **`RPiHighCpuTemperature`** | `temp > 75°C for 2m` | `warning` | SoC temperature is high. Active cooler should engage. |
| **`RPiCriticalCpuTemperature`**| `temp > 82°C for 1m` | `critical`| CPU thermal throttling threshold approached. |
| **`HostHighCpuLoad`** | `cpu > 85% for 5m` | `warning` | Sustained high CPU load across all cores. |
| **`HostLowMemory`** | `available RAM < 10%` | `warning` | Host running out of memory. OOM killer risk. |
| **`HostLowDiskSpace`** | `root disk free < 15%`| `critical`| Storage on root partition almost exhausted. |
| **`ServiceHttpProbeFailed`** | `probe_success == 0` | `critical`| Blackbox HTTP health-check failed for a monitored target. |
| **`ContainerKilledOrRestarted`**| `frequent restarts` | `warning` | Container restarting repeatedly in crash-loop. |

---

## 🛠️ Customization & Adding New Targets

To monitor a new web service, add it under `blackbox-http` in `prometheus/prometheus.yml`:

```yaml
  - job_name: "blackbox-http"
    static_configs:
      - targets:
          - "http://my-service:8080/health"
```

Reload configuration dynamically without downtime:
```bash
curl -X POST http://localhost:9090/-/reload
```

---

## 📄 License
MIT License. Feel free to use, modify, and extend this stack for your own home labs and production clusters!
