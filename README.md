# Ansible Monitoring Stack

Automated deployment of a full monitoring stack using Ansible, consisting of Prometheus, Grafana, Loki, Promtail, and Node Exporter on a remote server.

## Prerequisites

- Ansible installed on your local machine
- SSH access to target server (Ubuntu)
- Target server has internet access
- UFW firewall configured (Docker networks allowed)

## Project Structure

```
ansible-monitoring/
├── inventory                     # Target server definition
├── inventory.cfg                 # Ansible configuration
├── site.yml                      # Main playbook
├── group_vars/
│   └── all.yml                   # Variables (credentials, ports, etc.)
├── roles/
│   ├── docker/                   # Docker installation + Loki plugin
│   ├── node_exporter/            # Node Exporter container
│   └── monitoring_stack/         # Prometheus, Grafana, Loki, Promtail
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       └── templates/
│           ├── env.j2
│           └── promtail-config.yml.j2
└── files/
    ├── docker-compose.yml        # Docker Compose definition
    ├── prometheus.yml            # Prometheus configuration
    ├── grafana-datasources.yml   # Grafana datasource provisioning
    ├── grafana-dashboard.json    # Grafana dashboard template
    └── loki-config.yml           # Loki configuration
```

## Configuration

Edit `inventory` to set your server:

```
[monitoring]
vps ansible_host=<IP_ADDRESS> ansible_user=<USER> ansible_ssh_private_key_file=<SSH_KEY_PATH>
```

Edit `group_vars/all.yml` to customize:

| Variable | Default | Description |
|----------|---------|-------------|
| `monitoring_dir` | `/opt/monitoring` | Installation directory on server |
| `grafana_admin_user` | `admin` | Grafana admin username |
| `grafana_admin_password` | `admin` | Grafana admin password |
| `grafana_port` | `3000` | Grafana web port |
| `prometheus_retention_time` | `15d` | Prometheus data retention |
| `prometheus_retention_size` | `8GB` | Prometheus max storage size |
| `loki_retention` | `336h` | Loki log retention (14 days) |
| `node_exporter_port` | `9100` | Node Exporter port |
| `monitoring_instance_name` | `monitoring-vps` | Instance label for metrics |

## How to Run

```bash
# Deploy full stack
ansible-playbook -i inventory site.yml

# Deploy with verbose output
ansible-playbook -i inventory site.yml -v

# Deploy with extra variables
ansible-playbook -i inventory site.yml -e "grafana_admin_password=mysecret"
```

## What Gets Deployed

| Service | Port | Purpose |
|---------|------|---------|
| Prometheus | `127.0.0.1:9090` | Metrics collection |
| Grafana | `127.0.0.1:3000` | Dashboards & visualization |
| Loki | `127.0.0.1:3100` | Log aggregation |
| Promtail | - | Log shipping to Loki |
| Node Exporter | `127.0.0.1:9100` | System metrics |
| Loki Docker Driver | - | Docker log driver plugin |

> All services bind to `127.0.0.1` by default. Use SSH tunnel or reverse proxy to access remotely.

## Firewall Configuration

UFW must allow Docker networks to access Node Exporter (port 9100):

```bash
# Allow Docker bridge network
sudo ufw allow from 172.17.0.0/16 to any port 9100

# Allow monitoring network
sudo ufw allow from 172.19.0.0/16 to any port 9100
```

> Without these rules, Prometheus cannot scrape Node Exporter metrics.

## Access Services via SSH Tunnel

```bash
# Grafana
ssh -L 3000:127.0.0.1:3000 <user>@<server_ip>

# Prometheus
ssh -L 9090:127.0.0.1:9090 <user>@<server_ip>

# Loki
ssh -L 3100:127.0.0.1:3100 <user>@<server_ip>
```

Then open in browser:
- Grafana: `http://localhost:3000`
- Prometheus: `http://localhost:9090`

## Grafana Dashboard

Dashboard **"VPS Monitoring Dashboard"** is auto-imported to folder **"VPS Monitoring"**.

### Dashboard Panels

| Row | Panel | Type | Description |
|-----|-------|------|-------------|
| System Overview | CPU Usage | Gauge | Current CPU utilization |
| System Overview | Memory Usage | Gauge | Current memory utilization |
| System Overview | Disk Usage (/) | Gauge | Root filesystem usage |
| System Overview | Uptime | Stat | System uptime |
| CPU & Memory | CPU Usage Over Time | Time series | CPU user/system/iowait |
| CPU & Memory | Memory Usage Over Time | Time series | Used/cached/buffers/available |
| Disk & Network | Network Traffic | Time series | RX/TX per interface |
| Disk & Network | Disk I/O | Time series | Read/write bytes per device |
| Logs | Docker Container Logs | Logs | All container logs (Loki) |
| Logs | System Logs | Logs | syslog/authlog/journal (Loki) |

### Datasources

- **Prometheus** — for metrics queries
- **Loki** — for log queries

## Useful Commands on Server

```bash
# Check service status
docker ps

# View logs
docker logs prometheus
docker logs grafana
docker logs loki
docker logs promtail
docker logs node_exporter

# Restart stack
cd /opt/monitoring && docker compose restart

# Stop stack
cd /opt/monitoring && docker compose down

# Update containers
cd /opt/monitoring && docker compose pull && docker compose up -d
```

## Troubleshooting

### Loki DNS Resolution Failed

**Error:** `dial tcp: lookup loki on 127.0.0.11:53: server misbehaving`

**Solution:**
```bash
# Restart Docker daemon
sudo systemctl restart docker

# Recreate monitoring stack
cd /opt/monitoring && docker compose down && docker compose up -d
```

### Node Exporter Not Accessible from Prometheus

**Error:** Prometheus targets showing `down` status

**Solution:**
```bash
# Check UFW rules
sudo ufw status

# Allow Docker networks
sudo ufw allow from 172.17.0.0/16 to any port 9100
sudo ufw allow from 172.19.0.0/16 to any port 9100
```

### Loki Config Parse Error

**Error:** `yaml: unmarshal errors: field X not found`

**Solution:** Check `files/loki-config.yml` structure matches Loki 3.x format.

### Grafana Dashboard Not Imported

**Solution:**
```bash
# Check provisioning logs
docker logs grafana | grep -i provisioning

# Restart Grafana
docker restart grafana
```
