# Ansible Monitoring Stack

Automated deployment of a full monitoring stack using Ansible, consisting of Prometheus, Grafana, Loki, Promtail, and Node Exporter on a remote server.

## Prerequisites

- Ansible installed on your local machine
- SSH access to target server (Ubuntu)
- Target server has internet access

## Project Structure

```
ansible-monitoring/
├── inventory                 # Target server definition
├── site.yml                  # Main playbook
├── group_vars/
│   └── all.yml               # Variables (credentials, ports, etc.)
├── roles/
│   ├── docker/               # Docker installation
│   ├── node_exporter/        # Node Exporter container
│   └── monitoring_stack/     # Prometheus, Grafana, Loki, Promtail
└── files/
    ├── docker-compose.yml    # Docker Compose definition
    ├── prometheus.yml        # Prometheus configuration
    ├── grafana-datasources.yml
    ├── loki-config.yml
    └── promtail-config.yml
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

## Pre-configured Grafana Datasources

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
