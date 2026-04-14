# Day 69 — Ansible Playbooks and Modules

## Overview

Ad-hoc commands are useful for quick checks, but real automation lives in **playbooks**. A playbook is a YAML file that describes the desired state of your servers — which packages to install, which services to run, which files to place where. Write it once, run it a hundred times, and get the same result every time.

---

## Task 1: My First Playbook — `install-nginx.yml`

### Annotated Playbook

```yaml
---                                         # YAML document start marker
- name: Install and start Nginx on web servers  # PLAY name — human-readable label
  hosts: web                                # Which inventory group to target
  become: true                              # Escalate privileges (sudo) for ALL tasks

  tasks:                                    # List of TASKS in this play
    - name: Install Nginx                   # TASK — one unit of work
      yum:                                  # MODULE — manages packages on RHEL/CentOS
        name: nginx                         # Package to install
        state: present                      # Desired state: install if absent

    - name: Start and enable Nginx          # TASK — manage service lifecycle
      service:                              # MODULE — controls system services
        name: nginx                         # Service name
        state: started                      # Start it if not running
        enabled: true                       # Enable at boot

    - name: Create a custom index page      # TASK — place file content on remote host
      copy:                                 # MODULE — copy content to managed node
        content: "<h1>Deployed by Ansible - TerraWeek Server</h1>"  # Inline content
        dest: /usr/share/nginx/html/index.html  # Destination path on remote
```

> **Note:** Use `apt` instead of `yum` for Ubuntu/Debian-based instances.

### Running the Playbook

```bash
ansible-playbook install-nginx.yml
```

**First run output (expected):**

```
PLAY [Install and start Nginx on web servers] **********************************

TASK [Gathering Facts] *********************************************************
ok: [web-server-1]

TASK [Install Nginx] ***********************************************************
changed: [web-server-1]

TASK [Start and enable Nginx] **************************************************
changed: [web-server-1]

TASK [Create a custom index page] **********************************************
changed: [web-server-1]

PLAY RECAP *********************************************************************
web-server-1               : ok=4    changed=3    unreachable=0    failed=0
```

**Second run output (idempotency in action):**

```
TASK [Install Nginx] ***********************************************************
ok: [web-server-1]

TASK [Start and enable Nginx] **************************************************
ok: [web-server-1]

TASK [Create a custom index page] **********************************************
ok: [web-server-1]

PLAY RECAP *********************************************************************
web-server-1               : ok=4    changed=0    unreachable=0    failed=0
```

> **Idempotency:** Ansible only makes changes when needed. Running the same playbook twice produces identical end-states. The second run shows all `ok` — zero changes — because the desired state already exists.

### Verification

```bash
curl http://<web-server-public-ip>
# Output: <h1>Deployed by Ansible - TerraWeek Server</h1>
```

---

## Task 2: Playbook Structure — Q&A

### What is the difference between a play and a task?

| Concept | Description |
|---------|-------------|
| **Play** | A top-level block that maps a set of tasks to a group of hosts. A play defines *who* the work is done on (`hosts:`) and global settings like `become:`. |
| **Task** | A single unit of work within a play. Each task calls exactly one module and defines the desired state for one specific thing (install a package, start a service, write a file). |

### Can you have multiple plays in one playbook?

**Yes.** A single YAML playbook file can contain multiple plays (each starts with `- name:`). Each play can target a different host group, use different privilege settings, and run a different list of tasks. Plays run sequentially from top to bottom.

### What does `become: true` do at the play level vs the task level?

- **Play level** (`become: true` under the play header): All tasks in that play run with elevated privileges (sudo). This is the most common pattern.
- **Task level** (`become: true` on a specific task): Only that single task runs with elevated privileges. Useful when most tasks run as a regular user but one task requires root (e.g., installing a system package).

### What happens if a task fails — do remaining tasks still run?

By default, **Ansible stops all remaining tasks on the failed host** and marks it as failed in the PLAY RECAP. Other hosts in the play that didn't fail continue running. You can override this with:
- `ignore_errors: true` on a task — continue despite failure
- `any_errors_fatal: true` on a play — abort all hosts if any one fails

---

## Task 3: Essential Modules — `essential-modules.yml`

```yaml
---
- name: Practice essential Ansible modules
  hosts: all
  become: true

  tasks:

    # ── yum/apt ─────────────────────────────────────────────────────────────
    - name: Install multiple packages
      yum:
        name:
          - git
          - curl
          - wget
          - tree
        state: present

    # ── service ─────────────────────────────────────────────────────────────
    - name: Ensure Nginx is running and enabled at boot
      service:
        name: nginx
        state: started
        enabled: true

    # ── copy ────────────────────────────────────────────────────────────────
    - name: Copy application config file to remote host
      copy:
        src: files/app.conf
        dest: /etc/app.conf
        owner: root
        group: root
        mode: '0644'

    # ── file ────────────────────────────────────────────────────────────────
    - name: Create application directory with correct permissions
      file:
        path: /opt/myapp
        state: directory
        owner: ec2-user
        mode: '0755'

    # ── command ─────────────────────────────────────────────────────────────
    - name: Check disk space (no shell features needed)
      command: df -h
      register: disk_output

    - name: Print disk space output
      debug:
        var: disk_output.stdout_lines

    # ── shell ────────────────────────────────────────────────────────────────
    - name: Count running processes (needs pipe — use shell)
      shell: ps aux | wc -l
      register: process_count

    - name: Show total process count
      debug:
        msg: "Total processes: {{ process_count.stdout }}"

    # ── lineinfile ───────────────────────────────────────────────────────────
    - name: Set timezone in /etc/environment
      lineinfile:
        path: /etc/environment
        line: 'TZ=Asia/Kolkata'
        create: true
```

### Module Reference Table

| Module | What it does | Key arguments |
|--------|-------------|---------------|
| `yum` / `apt` | Install or remove packages | `name`, `state` (present/absent/latest) |
| `service` | Manage systemd/init services | `name`, `state`, `enabled` |
| `copy` | Copy a file or inline content to the remote host | `src`/`content`, `dest`, `owner`, `mode` |
| `file` | Create/delete files and directories; set permissions | `path`, `state` (file/directory/absent), `owner`, `mode` |
| `command` | Execute a command without a shell | `cmd` or free-form |
| `shell` | Execute a command through `/bin/sh` (pipes, redirects) | free-form |
| `lineinfile` | Ensure a specific line exists (or doesn't) in a file | `path`, `line`, `regexp`, `create` |
| `debug` | Print a variable or message during playbook run | `var` or `msg` |

### `command` vs `shell` — When to Use Each

| | `command` | `shell` |
|-|-----------|---------|
| **Runs via** | Direct exec (no shell) | `/bin/sh -c ...` |
| **Supports pipes** | ❌ No | ✅ Yes (`ps aux \| wc -l`) |
| **Supports redirects** | ❌ No | ✅ Yes (`echo foo >> file`) |
| **Supports env vars** | Limited (`$HOME` won't expand) | ✅ Yes |
| **Safer** | ✅ More predictable | ⚠️ Shell injection risk |
| **Use when** | Simple commands with no shell features | You need pipes, redirects, globs, or env expansion |

> **Best practice:** Prefer `command` by default. Use `shell` only when you genuinely need shell features.

---

## Task 4: Handlers — `nginx-config.yml`

Handlers are special tasks that only run when **notified** by another task. If the triggering task reports `changed`, the handler is queued; if it reports `ok` (no change), the handler never runs. Handlers always execute at the **end** of all tasks, and only once even if notified multiple times.

```yaml
---
- name: Configure Nginx with a custom config
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Deploy Nginx config               # If this file changes → notify handler
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'
      notify: Restart Nginx                   # Queues the handler (only if changed)

    - name: Deploy custom index page
      copy:
        content: "<h1>Managed by Ansible</h1><p>Server: {{ inventory_hostname }}</p>"
        dest: /usr/share/nginx/html/index.html

    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart Nginx                     # Only runs if notified AND something changed
      service:
        name: nginx
        state: restarted
```

### Handler Behavior — Before vs After

**First run** (config file is new → `changed`):

```
TASK [Deploy Nginx config] *****************************************************
changed: [web-server-1]

RUNNING HANDLER [Restart Nginx] ************************************************
changed: [web-server-1]          ← Handler triggered!
```

**Second run** (config already in place → `ok`):

```
TASK [Deploy Nginx config] *****************************************************
ok: [web-server-1]

                                  ← Handler NOT triggered (no change)
PLAY RECAP *********************************************************************
web-server-1               : ok=4    changed=0    unreachable=0    failed=0
```

> **Why handlers matter:** Without handlers, you'd restart Nginx on every playbook run whether or not the config changed. Handlers make restarts conditional and safe.

---

## Task 5: Dry Run, Diff, and Verbosity

### `--check` — Dry Run (Preview Mode)

```bash
ansible-playbook install-nginx.yml --check
```

Simulates the playbook run without making any real changes. Tasks report `changed` or `ok` as if they ran, but nothing is actually modified on the managed hosts. Essential for previewing impact before touching production.

### `--diff` — Show File Differences

```bash
ansible-playbook nginx-config.yml --check --diff
```

Outputs a unified diff for any file that would be created or modified. Combined with `--check`, this shows exactly what lines would change in config files — without applying the change.

### `-v` / `-vv` / `-vvv` — Verbosity Levels

```bash
ansible-playbook install-nginx.yml -v     # Show task results (module return values)
ansible-playbook install-nginx.yml -vv    # Also show input arguments
ansible-playbook install-nginx.yml -vvv   # Also show SSH connection details
```

### `--limit` — Target Specific Hosts

```bash
ansible-playbook install-nginx.yml --limit web-server-1
ansible-playbook install-nginx.yml --limit "web-server-1,web-server-2"
```

### `--list-hosts` / `--list-tasks` — Inspect Without Running

```bash
ansible-playbook install-nginx.yml --list-hosts   # Which hosts would be targeted?
ansible-playbook install-nginx.yml --list-tasks   # Which tasks would run?
```

### Why `--check --diff` is the Most Important Flag Combination for Production

Running `--check --diff` together gives you a **zero-risk preview** of every change:

- `--check` prevents any modifications from being applied
- `--diff` shows the exact before/after content for every file that would be touched

This lets you catch misconfigurations (wrong indentation in nginx.conf, wrong file permissions, unexpected content) **before** they cause downtime. It is the Ansible equivalent of a `git diff` before a merge — you see what changes and decide whether to proceed. For production environments, this should be a mandatory step in any change-control process.

---

## Task 6: Multiple Plays in One Playbook — `multi-play.yml`

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present
    - name: Start Nginx
      service:
        name: nginx
        state: started
        enabled: true

- name: Configure app servers
  hosts: app
  become: true
  tasks:
    - name: Install Node.js build dependencies
      yum:
        name:
          - gcc
          - make
        state: present
    - name: Create app directory
      file:
        path: /opt/app
        state: directory
        mode: '0755'

- name: Configure database servers
  hosts: db
  become: true
  tasks:
    - name: Install MySQL client
      yum:
        name: mysql
        state: present
    - name: Create secure data directory
      file:
        path: /var/lib/appdata
        state: directory
        mode: '0700'
```

### Running Multi-Play Playbooks

```bash
ansible-playbook multi-play.yml
```

Ansible runs each play in order. Each play only contacts the hosts in its `hosts:` group:

- `web` group → gets Nginx installed and started
- `app` group → gets gcc, make, and `/opt/app` directory
- `db` group → gets MySQL client and `/var/lib/appdata` directory

**Verification:**

```bash
# SSH into a web server
nginx -v          # Should succeed

# SSH into a db server
mysql --version   # Should succeed
nginx -v          # Should fail — Nginx was NOT installed here
```

This separation is a core Ansible pattern: one playbook file can express the complete desired state of your entire infrastructure, while each role's configuration stays targeted and independent.

---

## Key Concepts Summary

| Concept | Description |
|---------|-------------|
| **Playbook** | A YAML file containing one or more plays |
| **Play** | Maps a group of hosts to a list of tasks; sets global options like `become` |
| **Task** | A single call to an Ansible module |
| **Module** | The actual worker (yum, service, copy, file, etc.) |
| **Handler** | A task that runs only when notified by another task reporting `changed` |
| **Idempotency** | Running the same playbook N times produces the same end result |
| `register` | Saves a task's return value into a variable |
| `debug` | Prints variables or messages during a run |
| `notify` | Queues a named handler to run at the end of the play |
| `become` | Enables privilege escalation (sudo) |
| `--check` | Dry-run mode — no real changes |
| `--diff` | Shows file diffs for changes that would be made |

---

*Day 69 of #90DaysOfDevOps — Ansible Playbooks and Modules*
