# 🚀 Hirunews.lk Website Monitoring System

A Docker-based monitoring solution using Grafana, Prometheus, and Blackbox Exporter to monitor hirunews.lk website health and performance.

---

## 📋 Overview

This project monitors:
- ✅ Website availability (UP/DOWN)
- ⏱️ Response times
- 📊 HTTP status codes
- 🔒 SSL certificate expiration
- 🌐 DNS resolution times

---

## 🏗️ Architecture
```
Docker Container
├── Grafana (Port 3000)       → Dashboard & Visualization
├── Prometheus (Port 9090)     → Metrics Collection & Storage
└── Blackbox Exporter (9115)   → HTTP/HTTPS Probes
```

**Data Flow:**
1. Blackbox Exporter probes hirunews.lk every 15 seconds
2. Prometheus collects and stores metrics
3. Grafana displays data in dashboards

---

## 📦 Components

### 1. Prometheus
- Collects metrics from Blackbox Exporter
- Stores time-series data
- Scrapes data every 15 seconds

### 2. Blackbox Exporter
- Performs HTTP probes to target URLs
- Measures response times, SSL status, DNS lookup
- Returns metrics to Prometheus

### 3. Grafana
- Visualizes metrics in dashboards
- Queries Prometheus for data
- Displays graphs and statistics

### 4. Supervisord
- Manages all three services
- Auto-restarts on failure
- Handles logging

---

## 🔧 Installation

### Prerequisites
- Docker installed
- 2GB RAM minimum
- Ports 3000, 9090, 9115 available

### Steps

1. **Create project directory:**
```bash
mkdir ~/hirunews-monitoring
cd ~/hirunews-monitoring
```

2. **Create configuration files:**

**prometheus.yml:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://hirunews.lk/
          - https://hirunews.lk/local-news
          - https://hirunews.lk/sports-news
          - https://hirunews.lk/entertainment
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
```

**blackbox.yml:**
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      method: GET
      follow_redirects: true
```

**grafana-datasource.yml:**
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://127.0.0.1:9090
    isDefault: true
```

**supervisord.conf:**
```ini
[supervisord]
nodaemon=true
user=root

[program:blackbox]
command=/bin/blackbox_exporter --config.file=/etc/blackbox_exporter/config.yml
autostart=true
autorestart=true

[program:prometheus]
command=/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus
autostart=true
autorestart=true

[program:grafana]
command=/run.sh
autostart=true
autorestart=true
```

**Dockerfile:**
```dockerfile
FROM grafana/grafana:latest

USER root

RUN apk add --no-cache wget tar supervisor

RUN wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz && \
    tar -xvf prometheus-2.45.0.linux-amd64.tar.gz && \
    cp prometheus-2.45.0.linux-amd64/prometheus /bin/ && \
    cp prometheus-2.45.0.linux-amd64/promtool /bin/ && \
    mkdir -p /etc/prometheus && \
    cp -r prometheus-2.45.0.linux-amd64/consoles /etc/prometheus/ && \
    cp -r prometheus-2.45.0.linux-amd64/console_libraries /etc/prometheus/ && \
    rm -rf prometheus-2.45.0.linux-amd64*

RUN wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-amd64.tar.gz && \
    tar -xvf blackbox_exporter-0.24.0.linux-amd64.tar.gz && \
    cp blackbox_exporter-0.24.0.linux-amd64/blackbox_exporter /bin/ && \
    mkdir -p /etc/blackbox_exporter && \
    rm -rf blackbox_exporter-0.24.0.linux-amd64*

RUN mkdir -p /prometheus /var/log

COPY prometheus.yml /etc/prometheus/prometheus.yml
COPY blackbox.yml /etc/blackbox_exporter/config.yml
COPY grafana-datasource.yml /etc/grafana/provisioning/datasources/datasource.yml
COPY supervisord.conf /etc/supervisord.conf

EXPOSE 3000 9090 9115

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```

3. **Build Docker image:**
```bash
sudo docker build -t hirunews-monitoring .
```

4. **Run container:**
```bash
sudo docker run -d \
  -p 3000:3000 \
  -p 9090:9090 \
  -p 9115:9115 \
  --name hirunews-monitor \
  --restart unless-stopped \
  hirunews-monitoring
```

5. **Access Grafana:**
- URL: http://localhost:3000
- Username: `admin`
- Password: `admin`

---

## 📊 Monitoring Metrics

### Available Metrics:

| Metric | Description |
|--------|-------------|
| `probe_success` | Website UP (1) or DOWN (0) |
| `probe_http_duration_seconds` | Response time |
| `probe_http_status_code` | HTTP status (200, 404, 500, etc.) |
| `probe_ssl_earliest_cert_expiry` | SSL certificate expiration |
| `probe_dns_lookup_time_seconds` | DNS resolution time |

### Sample Queries:
```promql
# Website status
probe_success{instance="https://hirunews.lk/"}

# Response time
probe_http_duration_seconds{instance="https://hirunews.lk/"}

# SSL days remaining
(probe_ssl_earliest_cert_expiry{instance="https://hirunews.lk/"} - time()) / 86400

# Uptime percentage (24h)
avg_over_time(probe_success{instance="https://hirunews.lk/"}[24h]) * 100
```

---

## 🔧 Configuration

### Add More URLs:
Edit `prometheus.yml`:
```yaml
- targets:
    - https://hirunews.lk/
    - https://hirunews.lk/YOUR-NEW-PAGE
```

Rebuild:
```bash
sudo docker build -t hirunews-monitoring .
sudo docker stop hirunews-monitor && sudo docker rm hirunews-monitor
sudo docker run -d -p 3000:3000 -p 9090:9090 -p 9115:9115 --name hirunews-monitor hirunews-monitoring
```

### Change Scrape Interval:
In `prometheus.yml`:
```yaml
global:
  scrape_interval: 30s  # Change from 15s
```

---

## 🛠️ Commands
```bash
# Start/Stop/Restart
sudo docker start hirunews-monitor
sudo docker stop hirunews-monitor
sudo docker restart hirunews-monitor

# View logs
sudo docker logs -f hirunews-monitor

# Check service status
sudo docker exec hirunews-monitor supervisorctl status

# Restart specific service
sudo docker exec hirunews-monitor supervisorctl restart prometheus

# Access container
sudo docker exec -it hirunews-monitor /bin/sh
```

---

## 🔍 Troubleshooting

### No data showing:
```bash
# Wait 2 minutes, then check:
curl http://localhost:9090/api/v1/targets
curl http://localhost:9115/metrics
```

### Port already in use:
```bash
# Use different ports:
sudo docker run -d \
  -p 4000:3000 \
  -p 9091:9090 \
  -p 9116:9115 \
  --name hirunews-monitor \
  hirunews-monitoring
```

### Check logs:
```bash
sudo docker logs hirunews-monitor
sudo docker exec hirunews-monitor cat /var/log/prometheus.err.log
sudo docker exec hirunews-monitor cat /var/log/blackbox.err.log
```

---

## 📁 Project Structure
```
hirunews-monitoring/
├── Dockerfile
├── prometheus.yml
├── blackbox.yml
├── grafana-datasource.yml
├── supervisord.conf
└── README.md
```

---

## 📊 Access Points

- **Grafana:** http://localhost:3000
- **Prometheus:** http://localhost:9090
- **Blackbox:** http://localhost:9115

---

## �� Notes

- Default credentials: `admin` / `admin`
- Data retained for 15 days
- Scrapes every 15 seconds
- Container size: ~800MB
- Memory usage: ~500MB

---

## 📄 License

MIT License

---

**Version:** 1.0.0  
**Created:** February 2026
