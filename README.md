# vps-airflow-bootstrap

Provisions a fresh CentOS 9 VPS into a production-ready Airflow orchestration platform with a single command.

## Stack

| Component | Technology |
|---|---|
| OS | CentOS 9 Stream |
| Automation | Ansible |
| Container runtime | Docker CE + Compose v2 |
| Reverse proxy | Caddy 2 (automatic TLS) |
| Firewall | firewalld |
| Orchestrator | Apache Airflow 2.9.3 |

---

## Prerequisites

On the VPS:
- Fresh CentOS 9 Stream install
- Root SSH access on port 22 (will be changed during provisioning)
- Domain A record pointing to the VPS IP

On the operator machine:
- [uv](https://docs.astral.sh/uv/) installed

---

## Setup

### 1. Clone the repository

```bash
git clone git@github.com:rafael1985/vps-airflow-bootstrap.git
cd vps-airflow-bootstrap
```

### 2. Create the virtual environment

```bash
uv venv .venv
source .venv/bin/activate
uv pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

### 3. Configure secret variables

```bash
cp vars/secret.yml.example vars/secret.yml
```

Edit `vars/secret.yml` with your values:

```yaml
secret_vps_ip: "000.000.000.000"
secret_ssh_port: 2223
secret_deploy_user: "deploy"
secret_airflow_domain: "airflow.yourdomain.com"
secret_caddy_tls_email: "you@example.com"
secret_airflow_secret_key : #python3 -c "import secrets; print(secrets.token_hex(32))"
secret_airflow_admin_username: "admin"
secret_airflow_admin_password: "airflowpassword"
secret_postgres_password: #python3 -c "import secrets; print(secrets.token_hex(8))"
```
These values are injected into the `.env` file during provisioning and never touch the repository

> ⚠️ `vars/secret.yml` is in `.gitignore` and must **never** be committed.

### 4. Configure the inventory

```bash
cp inventories/production.ini.example inventories/production.ini
```

Edit `inventories/production.ini` with your VPS IP.

> ⚠️ `inventories/production.ini` is in `.gitignore` and must **never** be committed.

### 5. Add your SSH public key 

```bash
mkdir -p roles/security/files && cat ~/.ssh/id_ed25519.pub >> roles/security/files/deploy_authorized_keys
```
> ⚠️ This file is in `.gitignore`. Without it, the deploy user will have no SSH access after hardening.

## Run

```bash
# Activate the virtual environment (if not already active)
source .venv/bin/activate

# First run — fresh VPS, SSH still on port 22
ansible-playbook playbook.yml \
  -i inventories/production.ini \
  --private-key ~/.ssh/id_ed25519
```

After provisioning, SSH moves to `secret_ssh_port` and root login is blocked.

**Subsequent runs (idempotent):**
```bash
ansible-playbook playbook.yml \
  -i inventories/production.ini \
  -u deploy \
  -e ansible_port=2223 \
  --private-key ~/.ssh/id_ed25519
```

### Run by tag

```bash
ansible-playbook playbook.yml -i inventories/production.ini --tags security
ansible-playbook playbook.yml -i inventories/production.ini --tags docker
ansible-playbook playbook.yml -i inventories/production.ini --tags caddy
ansible-playbook playbook.yml -i inventories/production.ini --tags airflow
```

---

## Post-provisioning checklist

```bash
ssh -p 2223 deploy@<IP>           # SSH on new port
ssh root@<IP>                      # must be refused
docker info                        # Docker ok
docker compose version             # Compose v2
firewall-cmd --list-all            # Firewall active
systemctl status fail2ban          # Fail2ban active
systemctl status caddy             # Caddy active
curl -I https://<airflow_domain>   # Airflow accessible via HTTPS
```

---

## Repository structure

```
vps-airflow-bootstrap/
├── playbook.yml                        # Ansible entrypoint
├── requirements.txt                    # Python dependencies (uv)
├── requirements.yml                    # Ansible collections
├── .python-version                     # Python 3.12
├── vars/
│   ├── main.yml                        # Non-sensitive variables
│   ├── secret.yml.example              # Template — commit this
│   └── secret.yml                      # ⚠️ Local only, in .gitignore
├── inventories/
│   ├── production.ini.example          # Template — commit this
│   └── production.ini                  # ⚠️ Local only, in .gitignore
├── roles/
│   ├── security/                       # SSH hardening, firewall, fail2ban
│   ├── docker/                         # Docker CE + Compose v2
│   ├── caddy/                          # Reverse proxy + automatic TLS
│   └── airflow/                        # Directories, compose, health check
└── .github/workflows/lint.yml          # CI: ansible-lint + syntax check
```




