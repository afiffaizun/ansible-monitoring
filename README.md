# Ansible Monitoring Stack

Automated deployment of a full monitoring stack using Ansible, consisting of Prometheus, Grafana, Loki, Promtail, and Node Exporter on a remote server.

## Prerequisites

- Ansible installed on your local machine
- SSH access to target server (Ubuntu)
- Target server has internet access
- UFW firewall configured (Docker networks allowed)
- Ansible collections from `requirements.yml`
- Secret vars stored in an Ansible Vault (password file: `.vault_pass`, gitignored)

## Project Structure

```
ansible-monitoring/
├── ansible.cfg                   # Ansible configuration (inventory, SSH, callbacks)
├── inventory                     # Target server definition
├── site.yml                      # Main playbook
├── requirements.yml              # Ansible collections (community.docker, ansible.posix)
├── .ansible-lint                 # ansible-lint rules configuration
├── .gitignore                    # Excludes .vault_pass, retry files, etc.
├── group_vars/
│   └── all/
│       ├── vars.yml              # Variables (ports, retention, paths)
│       └── vault.yml             # Encrypted secrets (grafana_admin_password)
├── .github/workflows/
│   └── deploy.yml                # CI/CD: Lint → Dry Run → Deploy
├── dashboard/
│   └── vps-monitoring.json       # VPS monitoring dashboard
├── files/
│   ├── docker-compose.yml        # Docker Compose definition
│   ├── prometheus.yml            # Prometheus configuration
│   ├── grafana-datasources.yml   # Grafana datasource provisioning
│   └── loki-config.yml           # Loki configuration
└── roles/
    ├── docker/                   # Docker installation + Loki plugin
    ├── node_exporter/            # Node Exporter container
    └── monitoring_stack/         # Prometheus, Grafana, Loki, Promtail
        ├── tasks/main.yml
        ├── handlers/main.yml
        └── templates/
            ├── env.j2
            └── promtail-config.yml.j2
```

## Configuration

Edit `inventory` to set your server:

```
[monitoring]
vps ansible_host=<IP_ADDRESS> ansible_user=<USER> ansible_ssh_private_key_file=<SSH_KEY_PATH>
```

Edit `group_vars/all/vars.yml` to customize:

| Variable | Default | Description |
|----------|---------|-------------|
| `monitoring_dir` | `/opt/monitoring` | Installation directory on server |
| `grafana_admin_user` | `admin` | Grafana admin username |
| `grafana_port` | `3000` | Grafana web port |
| `prometheus_retention_time` | `15d` | Prometheus data retention |
| `prometheus_retention_size` | `8GB` | Prometheus max storage size |
| `loki_retention` | `336h` | Loki log retention (14 days) |
| `node_exporter_port` | `9100` | Node Exporter port |
| `monitoring_instance_name` | `monitoring-vps` | Instance label for metrics |

### Ansible Vault (Secrets)

The Grafana admin password is stored encrypted in `group_vars/all/vault.yml`:

```bash
# Edit encrypted secrets
ansible-vault edit group_vars/all/vault.yml

# Encrypt a new file
ansible-vault encrypt group_vars/all/vault.yml

# View decrypted content
ansible-vault view group_vars/all/vault.yml
```

The vault password is stored locally in `.vault_pass` (gitignored, `chmod 600`). The same file is replicated to the CI runner / server where needed. Do **not** commit `.vault_pass` or other plaintext secrets.

## How to Run

Install collections first:

```bash
ansible-galaxy collection install -r requirements.yml
```

Deploy:

```bash
# Deploy full stack (use .vault_pass if present)
ansible-playbook site.yml --vault-password-file .vault_pass

# Or prompt for the vault password interactively
ansible-playbook site.yml --ask-vault-pass

# Deploy with verbose output
ansible-playbook site.yml --vault-password-file .vault_pass -v
```

## CI/CD Pipeline

`.github/workflows/deploy.yml` runs on `main` push and pull requests. The `Lint` job is a **required status check** — PRs cannot be merged until `ansible-lint` passes.

| Job | Trigger | Description |
|-----|---------|-------------|
| **Lint** | push, PR, manual | Runs `ansible-lint` |
| **Dry Run** | push to main, manual | `ansible-playbook --check` against VPS |
| **Deploy** | push to main, manual | Full deploy to VPS |

Required GitHub Secrets:

- `SSH_PRIVATE_KEY` — private key to access the VPS
- `VAULT_PASSWORD` — password for `group_vars/all/vault.yml`

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

All services bind to `127.0.0.1`, so you need a single SSH tunnel forwarding all three ports:

### Option A — SSH config alias (recommended, already configured)

Add to `~/.ssh/config`:

```
Host monitoring-vps
  HostName <SERVER_IP>
  User <USER>
  IdentityFile ~/.ssh/id_ed25519
  LocalForward 3000 127.0.0.1:3000
  LocalForward 9090 127.0.0.1:9090
  LocalForward 3100 127.0.0.1:3100
```

Then a single command forwards all ports:

```bash
ssh monitoring-vps
```

### Option B — One-liner (no config)

```bash
ssh -N -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 -L 3100:127.0.0.1:3100 <USER>@<SERVER_IP>
```

### Service URLs

| Service | URL |
|---------|-----|
| Grafana | `http://localhost:3000` |
| Prometheus | `http://localhost:9090` |
| Loki | `http://localhost:3100` |

> Note: With aliases, once the tunnel is up, multiple services can be opened in parallel browser tabs. To close the tunnel run `pkill -f "ssh -N -o ExitOnForwardFailure"`.

## Grafana Dashboard

Dashboard **"VPS Monitoring Dashboard"** is auto-imported to folder **"VPS Monitoring"**.

| Dashboard | File | Description |
|-----------|------|-------------|
| VPS Monitoring Dashboard | `vps-monitoring.json` | CPU, Memory, Disk, Network, Logs |

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

### Vault Password Required to Deploy

**Error:** `Attempting to decrypt but no vault secrets found`

**Solution:** Ensure `.vault_pass` exists locally (gitignored) or pass it with `--vault-password-file`, or install vault ops running locally with collections installed (`pip install ansible ansible-lint`).