# Day 70 – Variables, Facts, Conditionals and Loops

## Overview

This document covers making Ansible playbooks dynamic and intelligent using variables, facts, conditionals, and loops. Static playbooks that do the same thing on every server are replaced with smart automation that adapts per host, group, and environment.

---

## Task 1: Variables in Playbooks

### `variables-demo.yml`

```yaml
---
- name: Variable demo
  hosts: all
  become: true

  vars:
    app_name: terraweek-app
    app_port: 8080
    app_dir: "/opt/{{ app_name }}"
    packages:
      - git
      - curl
      - wget

  tasks:
    - name: Print app details
      debug:
        msg: "Deploying {{ app_name }} on port {{ app_port }} to {{ app_dir }}"

    - name: Create application directory
      file:
        path: "{{ app_dir }}"
        state: directory
        mode: '0755'

    - name: Install required packages
      yum:
        name: "{{ packages }}"
        state: present
```

### CLI Variable Override

```bash
ansible-playbook variables-demo.yml -e "app_name=my-custom-app app_port=9090"
```

**Result:** The `-e` flag (extra vars) overrides the `app_name` and `app_port` defined inside the playbook. The directory becomes `/opt/my-custom-app` and the port becomes `9090`. This confirms that extra vars have the highest precedence.

---

## Task 2: `group_vars` and `host_vars`

### Directory Structure

```
ansible-practice/
  inventory.ini
  ansible.cfg
  group_vars/
    all.yml        # applies to every host
    web.yml        # applies only to [web] group
    db.yml         # applies only to [db] group
  host_vars/
    web-server.yml # applies only to web-server host
  playbooks/
    site.yml
```

### `group_vars/all.yml`

```yaml
---
ntp_server: pool.ntp.org
app_env: development
common_packages:
  - vim
  - htop
  - tree
```

### `group_vars/web.yml`

```yaml
---
http_port: 80
max_connections: 1000
web_packages:
  - nginx
```

### `group_vars/db.yml`

```yaml
---
db_port: 3306
db_packages:
  - mysql-server
```

### `host_vars/web-server.yml`

```yaml
---
max_connections: 2000
custom_message: "This is the primary web server"
```

### `playbooks/site.yml`

```yaml
---
- name: Apply common config
  hosts: all
  become: true
  tasks:
    - name: Install common packages
      yum:
        name: "{{ common_packages }}"
        state: present
    - name: Show environment
      debug:
        msg: "Environment: {{ app_env }}"

- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Show web config
      debug:
        msg: "HTTP port: {{ http_port }}, Max connections: {{ max_connections }}"
    - name: Show host-specific message
      debug:
        msg: "{{ custom_message }}"
```

### Variable Precedence (Low → High)

| Priority | Source | Example |
|----------|--------|---------|
| 1 (lowest) | Role defaults | `roles/myrole/defaults/main.yml` |
| 2 | `group_vars/all` | `group_vars/all.yml` |
| 3 | `group_vars/<group>` | `group_vars/web.yml` |
| 4 | `host_vars/<host>` | `host_vars/web-server.yml` |
| 5 | Playbook `vars:` | `vars: app_name: myapp` |
| 6 | Task vars / `set_fact` | `set_fact: foo=bar` |
| 7 (highest) | Extra vars (`-e`) | `-e "app_env=production"` |

**Key observation:** `host_vars/web-server.yml` sets `max_connections: 2000`, which overrides the `group_vars/web.yml` value of `1000` — only for `web-server`. All other web hosts still use `1000`.

---

## Task 3: Ansible Facts

### Useful Commands

```bash
# See all facts
ansible web-server -m setup

# Filter specific facts
ansible web-server -m setup -a "filter=ansible_os_family"
ansible web-server -m setup -a "filter=ansible_distribution*"
ansible web-server -m setup -a "filter=ansible_memtotal_mb"
ansible web-server -m setup -a "filter=ansible_default_ipv4"
```

### `facts-demo.yml`

```yaml
---
- name: Facts demo
  hosts: all
  tasks:
    - name: Show OS info
      debug:
        msg: >
          Hostname: {{ ansible_hostname }},
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }},
          RAM: {{ ansible_memtotal_mb }}MB,
          IP: {{ ansible_default_ipv4.address }}

    - name: Show all network interfaces
      debug:
        var: ansible_interfaces
```

### Five Useful Ansible Facts

| Fact | Use Case |
|------|----------|
| `ansible_distribution` | Conditionally use `yum` vs `apt` based on OS family (RHEL vs Ubuntu) |
| `ansible_memtotal_mb` | Skip memory-intensive tasks on hosts with < 1GB RAM; set JVM heap sizes dynamically |
| `ansible_default_ipv4.address` | Write the host's real IP into config files, firewall rules, or monitoring registrations |
| `ansible_processor_vcpus` | Configure worker thread counts in Nginx, Gunicorn, or Tomcat based on actual CPU count |
| `ansible_hostname` | Generate unique log file names, hostnames in config templates, or server labels in dashboards |

---

## Task 4: Conditionals with `when`

### `conditional-demo.yml`

```yaml
---
- name: Conditional tasks demo
  hosts: all
  become: true

  tasks:
    - name: Install Nginx (only on web servers)
      yum:
        name: nginx
        state: present
      when: "'web' in group_names"

    - name: Install MySQL (only on db servers)
      yum:
        name: mysql-server
        state: present
      when: "'db' in group_names"

    - name: Show warning on low memory hosts
      debug:
        msg: "WARNING: This host has less than 1GB RAM"
      when: ansible_memtotal_mb < 1024

    - name: Run only on Amazon Linux
      debug:
        msg: "This is an Amazon Linux machine"
      when: ansible_distribution == "Amazon"

    - name: Run only on Ubuntu
      debug:
        msg: "This is an Ubuntu machine"
      when: ansible_distribution == "Ubuntu"

    - name: Run only in production
      debug:
        msg: "Production settings applied"
      when: app_env == "production"

    - name: Multiple conditions (AND)
      debug:
        msg: "Web server with enough memory"
      when:
        - "'web' in group_names"
        - ansible_memtotal_mb >= 512

    - name: OR condition
      debug:
        msg: "Either web or app server"
      when: "'web' in group_names or 'app' in group_names"
```

### Sample Output

```
TASK [Install Nginx (only on web servers)]
ok: [web-server]
skipping: [db-server]   ← skipped because 'web' not in db-server's group_names

TASK [Install MySQL (only on db servers)]
skipping: [web-server]  ← skipped on web
ok: [db-server]

TASK [Run only in production]
skipping: [web-server]  ← app_env = "development" from group_vars/all.yml
skipping: [db-server]
```

**Verification:** Tasks correctly skip on hosts that do not match the condition. The `skipping` status is shown in the play output without errors.

---

## Task 5: Loops

### `loops-demo.yml`

```yaml
---
- name: Loops demo
  hosts: all
  become: true

  vars:
    users:
      - name: deploy
        groups: wheel
      - name: monitor
        groups: wheel
      - name: appuser
        groups: users

    directories:
      - /opt/app/logs
      - /opt/app/config
      - /opt/app/data
      - /opt/app/tmp

  tasks:
    - name: Create multiple users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        state: present
      loop: "{{ users }}"

    - name: Create multiple directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop: "{{ directories }}"

    - name: Install multiple packages
      yum:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - curl
        - unzip
        - jq

    - name: Print each user created
      debug:
        msg: "Created user {{ item.name }} in group {{ item.groups }}"
      loop: "{{ users }}"
```

### Sample Loop Output

```
TASK [Create multiple users]
changed: [web-server] => (item={'name': 'deploy', 'groups': 'wheel'})
changed: [web-server] => (item={'name': 'monitor', 'groups': 'wheel'})
changed: [web-server] => (item={'name': 'appuser', 'groups': 'users'})
```

Each iteration is shown separately with the current `item` value displayed.

### `loop` vs `with_items`

| Feature | `loop` | `with_items` |
|---------|--------|--------------|
| Introduced | Ansible 2.5+ | Ansible 1.x |
| Status | **Modern, recommended** | Legacy, deprecated |
| Syntax | `loop: [...]` | `with_items: [...]` |
| Flattening | Does NOT auto-flatten nested lists | Auto-flattens one level |
| Plugins | Replaces `with_dict`, `with_file`, `with_sequence`, etc. | Separate `with_*` plugins needed |

**Summary:** `loop` is the modern standard. It is consistent, plugin-agnostic, and supported going forward. Use `with_items` only in legacy playbooks where backward compatibility is required.

---

## Task 6: Server Health Report

### `server-report.yml`

```yaml
---
- name: Server Health Report
  hosts: all

  tasks:
    - name: Check disk space
      command: df -h /
      register: disk_result

    - name: Check memory
      command: free -m
      register: memory_result

    - name: Check running services
      shell: systemctl list-units --type=service --state=running | head -20
      register: services_result

    - name: Generate report
      debug:
        msg:
          - "========== {{ inventory_hostname }} =========="
          - "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
          - "IP: {{ ansible_default_ipv4.address }}"
          - "RAM: {{ ansible_memtotal_mb }}MB"
          - "Disk: {{ disk_result.stdout_lines[1] }}"
          - "Running services (first 20): {{ services_result.stdout_lines | length }}"

    - name: Flag if disk is critically low
      debug:
        msg: "ALERT: Check disk space on {{ inventory_hostname }}"
      when: "'9[0-9]%' in disk_result.stdout or '100%' in disk_result.stdout"

    - name: Save report to file
      copy:
        content: |
          Server: {{ inventory_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          IP: {{ ansible_default_ipv4.address }}
          RAM: {{ ansible_memtotal_mb }}MB
          Disk: {{ disk_result.stdout }}
          Checked at: {{ ansible_date_time.iso8601 }}
        dest: "/tmp/server-report-{{ inventory_hostname }}.txt"
      become: true
```

### Sample Report File (`/tmp/server-report-web-server.txt`)

```
Server: web-server
OS: Amazon Linux 2023
IP: 172.31.42.10
RAM: 1979MB
Disk: Filesystem      Size  Used Avail Use% Mounted on
      /dev/xvda1       20G  3.2G   17G  17% /
Checked at: 2026-04-14T08:23:11Z
```

**Verification:** SSH into the server and read the file:

```bash
ssh ec2-user@web-server
cat /tmp/server-report-web-server.txt
```

The report contains accurate OS, IP, RAM, disk, and timestamp information gathered dynamically using facts and registered variables.

---

## Key Takeaways

- **Variables** can come from playbook `vars:`, `group_vars/`, `host_vars/`, or CLI `-e` flags — with `-e` always winning
- **Facts** are auto-collected system metadata (OS, IP, RAM, CPU) gathered by the `setup` module before every play
- **`when`** conditionals use plain variable references (no `{{ }}`), and lists of conditions act as AND logic
- **`loop`** is the modern replacement for `with_items`, iterating cleanly over lists or lists of dicts using `{{ item }}`
- **`register`** captures command output into a variable with fields like `stdout`, `stdout_lines`, and `rc` for later use
