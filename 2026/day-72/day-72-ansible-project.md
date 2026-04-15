# Day 72 — Ansible Project: Automate Docker and Nginx Deployment

## Overview

This project brings together everything from the five-day Ansible block (Days 68–72) into a single, production-style automation pipeline. One command goes from a fresh server to a fully running Docker container fronted by an Nginx reverse proxy, with secrets managed by Ansible Vault.

**Architecture:**
```
Ansible Control Node
        │
        ▼
  Managed Server
  ┌─────────────────────────────┐
  │  Nginx (port 80)            │
  │      │ reverse proxy        │
  │      ▼                      │
  │  Docker Container (8080)    │
  └─────────────────────────────┘
```

---

## Project Directory Structure

```
ansible-docker-project/
├── ansible.cfg
├── inventory.ini
├── site.yml                          # Master playbook
├── .vault_pass                       # Gitignored vault password file
├── .gitignore
├── group_vars/
│   ├── all.yml                       # Common variables (timezone, packages, etc.)
│   └── web/
│       ├── vars.yml                  # Nginx-specific variables
│       └── vault.yml                 # Vault-encrypted Docker Hub credentials
└── roles/
    ├── common/                       # Baseline setup for all servers
    │   └── tasks/
    │       └── main.yml
    ├── docker/                       # Docker install + container lifecycle
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   └── main.yml
    │   ├── templates/
    │   │   └── docker-compose.yml.j2
    │   └── handlers/
    │       └── main.yml
    └── nginx/                        # Nginx reverse proxy
        ├── defaults/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        ├── templates/
        │   ├── nginx.conf.j2
        │   └── app-proxy.conf.j2
        └── handlers/
            └── main.yml
```

---

## Configuration Files

### `ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
vault_password_file = .vault_pass
remote_user = ec2-user
private_key_file = ~/.ssh/my-key.pem
```

### `inventory.ini`

```ini
[web]
web1 ansible_host=<EC2_PUBLIC_IP> ansible_user=ec2-user

[web:vars]
ansible_ssh_private_key_file=~/.ssh/my-key.pem
```

### `group_vars/all.yml`

```yaml
---
timezone: Asia/Kolkata
project_name: devops-app
app_env: development
common_packages:
  - vim
  - curl
  - wget
  - git
  - htop
  - tree
  - jq
  - unzip
```

### `group_vars/web/vars.yml`

```yaml
---
nginx_http_port: 80
nginx_upstream_port: 8080
nginx_server_name: "_"
```

---

## Key Role Files

### Master Playbook — `site.yml`

```yaml
---
- name: Apply common configuration
  hosts: all
  become: true
  roles:
    - common
  tags: common

- name: Install Docker and run containers
  hosts: web
  become: true
  roles:
    - docker
  tags: docker

- name: Configure Nginx reverse proxy
  hosts: web
  become: true
  roles:
    - nginx
  tags: nginx
```

### Common Role — `roles/common/tasks/main.yml`

```yaml
---
- name: Update package cache
  yum:
    update_cache: true
  tags: common

- name: Install common packages
  yum:
    name: "{{ common_packages }}"
    state: present
  tags: common

- name: Set hostname
  hostname:
    name: "{{ inventory_hostname }}"
  tags: common

- name: Set timezone
  timezone:
    name: "{{ timezone }}"
  tags: common

- name: Create deploy user
  user:
    name: deploy
    groups: wheel
    shell: /bin/bash
    state: present
  tags: common
```

### Docker Role — `roles/docker/defaults/main.yml`

```yaml
---
docker_app_image: nginx
docker_app_tag: latest
docker_app_name: myapp
docker_app_port: 8080
docker_container_port: 80
```

### Docker Role — `roles/docker/tasks/main.yml`

```yaml
---
- name: Install Docker dependencies
  yum:
    name:
      - yum-utils
      - device-mapper-persistent-data
      - lvm2
    state: present
  tags: docker

- name: Add Docker CE repository
  command: yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  args:
    creates: /etc/yum.repos.d/docker-ce.repo
  tags: docker

- name: Install Docker CE
  yum:
    name: docker-ce
    state: present
  tags: docker

- name: Start and enable Docker service
  service:
    name: docker
    state: started
    enabled: true
  tags: docker

- name: Add deploy user to docker group
  user:
    name: deploy
    groups: docker
    append: true
  tags: docker

- name: Install Docker SDK for Python (required for community.docker modules)
  pip:
    name: docker
    state: present
  tags: docker

- name: Log in to Docker Hub
  community.docker.docker_login:
    username: "{{ vault_docker_username }}"
    password: "{{ vault_docker_password }}"
  become_user: deploy
  when: vault_docker_username is defined
  tags: docker

- name: Pull application image
  community.docker.docker_image:
    name: "{{ docker_app_image }}"
    tag: "{{ docker_app_tag }}"
    source: pull
  tags: docker

- name: Run application container
  community.docker.docker_container:
    name: "{{ docker_app_name }}"
    image: "{{ docker_app_image }}:{{ docker_app_tag }}"
    state: started
    restart_policy: always
    ports:
      - "{{ docker_app_port }}:{{ docker_container_port }}"
  tags: docker

- name: Wait for container to be healthy
  uri:
    url: "http://localhost:{{ docker_app_port }}"
    status_code: 200
  retries: 5
  delay: 3
  register: health_check
  until: health_check.status == 200
  tags: docker
```

### Docker Role — `roles/docker/handlers/main.yml`

```yaml
---
- name: Restart Docker
  service:
    name: docker
    state: restarted
```

### Nginx Role — `roles/nginx/tasks/main.yml`

```yaml
---
- name: Install Nginx
  yum:
    name: nginx
    state: present
  tags: nginx

- name: Remove default Nginx site config
  file:
    path: /etc/nginx/conf.d/default.conf
    state: absent
  notify: Reload Nginx
  tags: nginx

- name: Deploy reverse proxy config from template
  template:
    src: app-proxy.conf.j2
    dest: /etc/nginx/conf.d/{{ project_name }}.conf
    owner: root
    group: root
    mode: '0644'
  notify: Reload Nginx
  tags: nginx

- name: Test Nginx configuration
  command: nginx -t
  changed_when: false
  tags: nginx

- name: Start and enable Nginx
  service:
    name: nginx
    state: started
    enabled: true
  tags: nginx
```

### Nginx Role — `roles/nginx/handlers/main.yml`

```yaml
---
- name: Reload Nginx
  service:
    name: nginx
    state: reloaded

- name: Restart Nginx
  service:
    name: nginx
    state: restarted
```

### Nginx Reverse Proxy Template — `roles/nginx/templates/app-proxy.conf.j2`

```nginx
# Reverse Proxy to Docker Container -- Managed by Ansible
upstream docker_app {
    server 127.0.0.1:{{ nginx_upstream_port }};
}

server {
    listen {{ nginx_http_port }};
    server_name {{ nginx_server_name }};

    location / {
        proxy_pass http://docker_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /health {
        access_log off;
        return 200 'OK';
        add_header Content-Type text/plain;
    }

{% if app_env == 'production' %}
    access_log /var/log/nginx/{{ project_name }}_access.log;
    error_log /var/log/nginx/{{ project_name }}_error.log;
{% else %}
    access_log /var/log/nginx/{{ project_name }}_access.log;
    error_log /var/log/nginx/{{ project_name }}_error.log debug;
{% endif %}
}
```

---

## Vault: Encrypting Docker Hub Credentials

Create the vault file interactively:
```bash
ansible-vault create group_vars/web/vault.yml
```

Contents of `group_vars/web/vault.yml` (before encryption):
```yaml
vault_docker_username: your-dockerhub-username
vault_docker_password: your-dockerhub-access-token
```

Create a vault password file for CI/local convenience:
```bash
echo "YourStrongVaultPassword" > .vault_pass
chmod 600 .vault_pass
echo ".vault_pass" >> .gitignore
```

Reference in `ansible.cfg`:
```ini
vault_password_file = .vault_pass
```

Verify the file is encrypted:
```bash
cat group_vars/web/vault.yml
# Output should show: $ANSIBLE_VAULT;1.1;AES256 ...
```

---

## Deployment Commands

### Prerequisites
```bash
# Install required Ansible collection
ansible-galaxy collection install community.docker
```

### Dry Run (Always First)
```bash
ansible-playbook site.yml --check --diff
```

### Full Deployment
```bash
ansible-playbook site.yml
```

### Selective Execution with Tags
```bash
# Only install Docker and run containers
ansible-playbook site.yml --tags docker

# Only update Nginx config
ansible-playbook site.yml --tags nginx

# Skip common setup (baseline already applied)
ansible-playbook site.yml --skip-tags common
```

### Deploy a Different App (Extra Vars)
```bash
ansible-playbook site.yml --tags docker \
  -e "docker_app_image=httpd docker_app_tag=latest docker_app_name=apache-app"
```

---

## Verification Steps

### 1. Check Container is Running (on managed node)
```bash
# Expected: container listed with 0.0.0.0:8080->80/tcp
docker ps
```

**Expected output:**
```
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                   NAMES
a1b2c3d4e5f6   nginx:latest   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp   myapp
```

### 2. Direct Container Access (port 8080)
```bash
curl http://<SERVER_IP>:8080
# Expected: nginx welcome page HTML
```

### 3. Nginx Reverse Proxy Access (port 80)
```bash
curl http://<SERVER_IP>:80
# Expected: same nginx welcome page, proxied through Nginx
```

### 4. Health Endpoint
```bash
curl http://<SERVER_IP>/health
# Expected: OK
```

---

## Idempotency Proof

Run the playbook a second time immediately after a successful deploy:

```bash
ansible-playbook site.yml
```

**Expected second-run output summary:**
```
PLAY RECAP *****************************************************
web1  : ok=18   changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

`changed=0` confirms the entire setup is idempotent — Ansible detected no drift and made no changes.

---

## How Tags Were Used

| Tag      | What it runs                                | When to use                          |
|----------|---------------------------------------------|--------------------------------------|
| `common` | Package install, hostname, timezone, user   | First-time server setup              |
| `docker` | Docker install, image pull, container run   | Updating the app image/container     |
| `nginx`  | Nginx install, config deploy, service start | Updating proxy config only           |

Example: to roll out a new Docker image without touching Nginx config:
```bash
ansible-playbook site.yml --tags docker -e "docker_app_tag=1.25.3"
```

---

## Ansible Concept Mapping

| Day | Concept Used in This Project |
|-----|------------------------------|
| 68  | Inventory (`inventory.ini`), SSH setup, `ansible.cfg`, ad-hoc commands for smoke testing |
| 69  | Playbooks (`site.yml`), modules (`yum`, `service`, `user`, `uri`), handlers (`Reload Nginx`) |
| 70  | Variables (`group_vars/all.yml`, `defaults/main.yml`), facts (`inventory_hostname`), conditionals (`when: vault_docker_username is defined`), loops (package lists) |
| 71  | Roles (`common`, `docker`, `nginx`), Jinja2 templates (`app-proxy.conf.j2`), Galaxy (`community.docker`), Vault (encrypted credentials) |
| 72  | Everything combined — one `ansible-playbook site.yml` deploys the full stack |

---

## What I Would Add for Production

| Enhancement | Tool / Approach |
|-------------|-----------------|
| HTTPS / TLS termination | Certbot + `community.crypto` role, auto-renew via cron |
| Multi-container app | Docker Compose via `docker-compose.yml.j2` template |
| Log rotation | `logrotate` config deployed via Ansible template |
| Monitoring | Prometheus Node Exporter role + Grafana dashboard |
| Zero-downtime deploys | Rolling updates with `serial: 1` in the playbook |
| Firewall rules | `firewalld` or `ufw` role to restrict ports |
| Health-check alerting | Ansible `uri` module in a scheduled AWX/Tower job |
| CI/CD integration | GitHub Actions triggering `ansible-playbook` on push to main |

---

## Cleanup

If using Terraform:
```bash
terraform destroy
```

If instances were created manually, terminate them from the EC2 console to avoid ongoing charges.

---

## Key Takeaways

- **Roles** make the project modular — Docker and Nginx can be updated independently using tags.
- **Vault** keeps secrets out of version control while keeping the playbook fully automated.
- **Templates** let a single Jinja2 file handle both development (debug logging) and production (standard logging) via `app_env`.
- **Handlers** ensure Nginx only reloads when config actually changes — no unnecessary restarts.
- **`nginx -t` before reload** prevents bad config from taking down the proxy — safe deployment by default.
- **Idempotency** means this playbook can be run on a schedule or in CI without causing disruption.
