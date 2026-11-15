# Ansible Monitoring Stack Automation

Automate a basic monitoring stack on an Ubuntu EC2 instance using Ansible:

- **Prometheus** — collects metrics
- **Node Exporter** — exposes Linux host metrics
- **Grafana** — visualizes Prometheus metrics

## Architecture

```text
Your browser
  |--------------------> Grafana :3000
  |--------------------> Prometheus :9090
                              |
                              | scrapes
                              v
                       Node Exporter :9100
                              |
                         Ubuntu EC2
```

This starter project installs all three services on **one Ubuntu EC2 instance**. It is intended for learning and lab use, not a production high-availability deployment.

## Repository structure

```text
ansible-monitoring-stack-automation/
├── ansible.cfg
├── inventory/
│   ├── hosts.ini.example
│   └── group_vars/
│       └── all.yml
├── playbooks/
│   └── site.yml
├── templates/
│   └── prometheus.yml.j2
├── requirements.yml
├── .gitignore
└── README.md
```

## 1. Prerequisites

- Ansible installed on your local machine (control node)
- An Ubuntu 22.04 or 24.04 EC2 instance (managed node)
- SSH access to the EC2 instance
- An EC2 security group configured as described below

Do not upload your private SSH key, AWS credentials, or real inventory containing sensitive host details to GitHub.

## 2. Configure the EC2 security group

Allow inbound traffic only from trusted source IP addresses:

| Port | Purpose | Recommended source |
|---|---|---|
| 22 | SSH | Your public IP /32 |
| 3000 | Grafana | Your public IP /32 |
| 9090 | Prometheus | Your public IP /32, for lab access |
| 9100 | Node Exporter | Do not expose publicly; keep private |

For a single-instance lab, Prometheus and Grafana are accessed from your IP. Never use `0.0.0.0/0` for SSH or monitoring dashboards. Node Exporter does not need a public inbound rule when Prometheus runs on the same EC2 instance.

## 3. Prepare your inventory

Copy the example inventory:

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
```

Edit `inventory/hosts.ini` and replace the example IP with your EC2 public IP. Set the correct SSH username and the path to your local PEM key. Common Ubuntu AMI username: `ubuntu`.

Example:

```ini
[monitoring]
monitoring-ec2 ansible_host=YOUR_EC2_PUBLIC_IP ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/YOUR_KEY.pem
```

Keep `inventory/hosts.ini` untracked; it is ignored by Git.

## 4. Check SSH and Ansible connectivity

From the repository root:

```bash
ansible monitoring -m ping
```

A successful response includes `"ping": "pong"`.

## 5. Run the playbook

```bash
ansible-playbook playbooks/site.yml
```

The playbook installs and enables Node Exporter, Prometheus, and Grafana. It is designed to be re-runnable.

## 6. Verify services

On the EC2 instance:

```bash
sudo systemctl status node_exporter --no-pager
sudo systemctl status prometheus --no-pager
sudo systemctl status grafana-server --no-pager
```

Check local endpoints on EC2:

```bash
curl -I http://localhost:9100/metrics
curl -I http://localhost:9090/-/healthy
curl -I http://localhost:3000/api/health
```

Open these in your browser, replacing the placeholder:

- Grafana: `http://YOUR_EC2_PUBLIC_IP:3000`
- Prometheus: `http://YOUR_EC2_PUBLIC_IP:9090`

Grafana's initial login is usually `admin` / `admin`; change the password immediately when prompted. Prometheus is configured as Grafana's data source at `http://localhost:9090`.

## 7. Add a dashboard in Grafana

1. Sign in to Grafana and change the initial password.
2. Open **Dashboards → New → Import**.
3. Import dashboard ID **1860** (Node Exporter Full).
4. Select the Prometheus data source.

## Troubleshooting

- **SSH timeout:** check the EC2 security group, public IP, route to an Internet Gateway, and PEM key permissions.
- **Ansible unreachable:** verify `ansible_user`, `ansible_host`, and the key path in your local inventory.
- **Dashboard not loading:** confirm the service is active and port 3000/9090 is allowed from your IP.
- **Prometheus target down:** open **Status → Targets** in Prometheus and check the Node Exporter target.

## Cleanup

To avoid ongoing AWS charges, stop or terminate the EC2 instance when the lab is finished. Also remove temporary inbound security-group rules that are no longer needed.

## Security notes

- Never commit `.pem` files, AWS credentials, passwords, or the real inventory.
- Restrict SSH and dashboard ports to your own IP.
- Use private networking and TLS/authentication before adapting this lab for production.
