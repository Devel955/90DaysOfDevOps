# Day 71 - Roles, Templates, Galaxy and Vault

## 🔹 Webserver Role Structure

roles/webserver/
├── defaults/main.yml
├── handlers/main.yml
├── tasks/main.yml
├── templates/
│   ├── index.html.j2
│   └── vhost.conf.j2

---

## 🔹 Jinja2 Templates

Used Jinja2 templates to dynamically generate Nginx configuration files.

Example:
- {{ app_name }}
- {{ ansible_hostname }}
- {{ ansible_default_ipv4.address }}

Rendered Output:
<h1>terraweek</h1>
<p>Server: DESKTOP-IE9QTP1</p>

---

## 🔹 Role Execution

Command:
ansible-playbook -i inventory site.yml

Result:
- Nginx installed
- Configuration applied
- Web page deployed successfully

---

## 🔹 Ansible Galaxy

Installed role:
ansible-galaxy install geerlingguy.docker

Used in playbook to install Docker automatically.

---

## 🔹 Docker Verification

Commands:
docker ps
docker run hello-world

Output:
Hello from Docker!

---

## 🔹 Ansible Vault

Created encrypted file:
ansible-vault create group_vars/db/vault.yml

Stored secrets securely:
- DB password
- API keys

Used vault in playbook with:
--ask-vault-pass
--vault-password-file

---

## 🔹 Key Learnings

- Templates allow dynamic configuration
- Roles make automation reusable
- Galaxy provides ready-made roles
- Vault secures sensitive data

---

## 🔹 When to Use

Ad-hoc:
- Quick tasks

Playbooks:
- Automation workflows

Roles:
- Reusable and scalable automation

---

## 🔹 Final Outcome

Successfully built production-level automation using:
- Ansible Roles
- Jinja2 Templates
- Galaxy roles
- Vault encryption
