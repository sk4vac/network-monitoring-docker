# Docker Network Monitoring

A simple network and infrastructure monitoring stack built with Docker Compose.

## Stack

- Prometheus - collects and stores metrics
- Node Exporter - monitors Linux host metrics
- cAdvisor - monitors Docker containers
- Grafana - displays metrics in dashboards

## Project Structure

```text
network-monitoring/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
└── grafana/
```

## Stage 1 - Create the Project

```bash
mkdir -p ~/Projects/docker-projects/network-monitoring
cd ~/Projects/docker-projects/network-monitoring
mkdir -p prometheus grafana
```

## Stage 2 - Configure Prometheus

Create:

```text
prometheus/prometheus.yml
```

Content:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]
```

## Stage 3 - Add Node Exporter

Node Exporter collects Linux host metrics such as:

- CPU
- RAM
- Disk
- Network
- Uptime

Add this service to `docker-compose.yml`:

```yaml
node-exporter:
  image: prom/node-exporter:latest
  container_name: node-exporter
  ports:
    - "9100:9100"
  command:
    - "--path.rootfs=/host"
  volumes:
    - "/:/host:ro,rslave"
```

Test it:

```bash
curl -s http://localhost:9100/metrics | head
```

Add Node Exporter to `prometheus/prometheus.yml`:

```yaml
  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

## Stage 4 - Add cAdvisor

cAdvisor collects Docker container metrics.

Add:

```yaml
cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  container_name: cadvisor
  ports:
    - "8080:8080"
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker:/var/lib/docker:ro
```

Test:

```bash
curl -s http://localhost:8080/metrics | head
```

Add to Prometheus:

```yaml
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```

## Stage 5 - Add Grafana

Grafana displays the metrics collected by Prometheus.

Add:

```yaml
grafana:
  image: grafana/grafana:latest
  container_name: grafana
  ports:
    - "3000:3000"
  volumes:
    - grafana_data:/var/lib/grafana

volumes:
  grafana_data:
```

Start the stack:

```bash
docker compose config
docker compose up -d
```

Check:

```bash
docker compose ps
```

Expected containers:

```text
prometheus
node-exporter
cadvisor
grafana
```

Open Grafana:

```text
http://localhost:3000
```

Add Prometheus as a Grafana data source.

Use:

```text
http://prometheus:9090
```

Important: use `prometheus:9090`, not `localhost:9090`, because Grafana is running inside Docker.

Test with:

```promql
up
```

`1` means the target is reachable.

## Stage 6 - Create the Grafana Dashboard

Create a dashboard and add panels using these PromQL queries.

### CPU Usage

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Unit:

```text
Percent (0-100)
```

### Memory Usage

```promql
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

Unit:

```text
Percent (0-100)
```

### Root Disk Usage

```promql
100 * (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})
```

Unit:

```text
Percent (0-100)
```

### Network Receive

```promql
sum by (device) (rate(node_network_receive_bytes_total{device!="lo"}[5m])) * 8
```

Unit:

```text
bits/sec
```

### Network Transmit

```promql
sum by (device) (rate(node_network_transmit_bytes_total{device!="lo"}[5m])) * 8
```

Unit:

```text
bits/sec
```

### System Uptime

```promql
time() - node_boot_time_seconds
```

Use a `Stat` visualization.

### Target Status

```promql
up
```

`1` = UP

`0` = DOWN

### Container CPU

```promql
sum by (name) (rate(container_cpu_usage_seconds_total{name!=""}[5m])) * 100
```

### Container Memory

```promql
sum by (name) (container_memory_working_set_bytes{name!=""})
```

Unit:

```text
bytes
```

## Enable Grafana Auto Refresh

For near-real-time monitoring:

1. Open the dashboard.
2. Set the dashboard time range to something like `Last 15 minutes`.
3. Set the refresh interval to `10s`.

Prometheus currently collects metrics every:

```text
15 seconds
```

Grafana can refresh the dashboard every:

```text
10 seconds
```

## Useful Commands

Check containers:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs
```

View logs for one service:

```bash
docker compose logs prometheus
docker compose logs grafana
```

Restart services:

```bash
docker compose restart
```

Stop the project:

```bash
docker compose stop
```

Start it again:

```bash
docker compose start
```

Stop and remove the containers/network:

```bash
docker compose down
```

Remove containers, network, and named volumes:

```bash
docker compose down -v
```

Start the project:

```bash
docker compose up -d
```

## Access

```text
Grafana:       http://localhost:3000
Prometheus:    http://localhost:9090
cAdvisor:      http://localhost:8080
Node Exporter: http://localhost:9100/metrics
```

## Current Architecture

```text
                    Browser
                       |
                       v
                 Grafana :3000
                       |
                       v
                Prometheus :9090
                 /             \
                v               v
       Node Exporter         cAdvisor
            :9100               :8080
                |                 |
                v                 v
          Linux Host          Docker
            Metrics           Metrics
```

## Future Improvements

Possible next stages:

- Monitor the Debian Server VM
- Monitor network connectivity with ICMP
- Add Blackbox Exporter
- Monitor routers and switches with SNMP
- Monitor Wi-Fi access points
- Create network-specific Grafana dashboards
- Add alerts for high CPU, RAM, disk usage, or host downtime
