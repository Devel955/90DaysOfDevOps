# Day 28 – Revision Day

## Overview

Day 28 was dedicated to revising everything learned from Day 1 to Day 27. The goal was to evaluate understanding, identify weak areas, and strengthen concepts related to Linux, Shell Scripting, and Git & GitHub.

---

# Task 1 – Self Assessment

## Linux

* Navigate file system and manage files → **Can do confidently**
* Manage processes → **Need to revisit**
* Work with systemd services → **Need to revisit**
* Edit text files using vim/nano → **Can do confidently**
* Troubleshoot CPU, memory, disk issues → **Can do confidently**
* Explain Linux file system hierarchy → **Can do confidently**
* Create users and groups → **Can do confidently**
* Manage file permissions using chmod → **Can do confidently**
* Change ownership with chown/chgrp → **Can do confidently**
* Create and manage LVM volumes → **Need to revisit**
* Network troubleshooting commands → **Can do confidently**
* Explain DNS, IP, subnets and ports → **Need to revisit**

---

## Shell Scripting

* Write scripts with variables and arguments → **Can do confidently**
* Use if/elif/else statements → **Can do confidently**
* Use loops (for, while) → **Can do confidently**
* Write functions → **Need to revisit**
* Text processing tools (grep, awk, sed) → **Need to revisit**
* Error handling in scripts → **Need to revisit**
* Schedule scripts with crontab → **Can do confidently**

---

## Git & GitHub

* Initialize repository and commit changes → **Can do confidently**
* Branch creation and switching → **Can do confidently**
* Push and pull from GitHub → **Can do confidently**
* Explain clone vs fork → **Can do confidently**
* Merge branches → **Can do confidently**
* Git rebase → **Need to revisit**
* Git stash → **Can do confidently**
* Cherry pick → **Need to revisit**
* Reset vs revert → **Can do confidently**
* Branching strategies → **Need to revisit**
* GitHub CLI → **Need to revisit**

---

# Task 2 – Topics Revisited

## 1. Logical Volume Manager (LVM)

LVM provides flexible disk management in Linux. It allows resizing and managing storage without repartitioning disks.

Components:

* Physical Volume (PV)
* Volume Group (VG)
* Logical Volume (LV)

Example commands:

```
pvcreate /dev/sdb
vgcreate myvg /dev/sdb
lvcreate -L 5G -n mylv myvg
```

---

## 2. systemd Service Management

systemd manages services and system processes.

Common commands:

```
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl status nginx
systemctl enable nginx
systemctl disable nginx
```

---

## 3. Git Rebase

Git rebase is used to move or combine commits from one branch onto another branch.

Example:

```
git checkout feature
git rebase main
```

This keeps commit history clean and linear.

---

# Task 3 – Quick Fire Answers

### What does chmod 755 script.sh do?

It sets file permissions as:

Owner → read, write, execute
Group → read, execute
Others → read, execute

---

### Difference between Process and Service

Process → A running instance of a program.

Service → A background process managed by the system (example: nginx, ssh).

---

### How to find which process is using port 8080?

```
lsof -i :8080
```

or

```
netstat -tulnp | grep 8080
```

---

### What does set -euo pipefail do?

These options make shell scripts safer.

* `-e` exit if any command fails
* `-u` error if undefined variable is used
* `pipefail` pipeline fails if any command fails

---

### Difference between git reset --hard and git revert

`git reset --hard` removes commits and changes permanently.

`git revert` creates a new commit that reverses the previous commit.

---

### Recommended branching strategy for small teams

GitHub Flow is recommended for small teams because it is simple and supports fast development and deployments.

---

### What does git stash do?

git stash temporarily saves uncommitted changes so you can switch branches without committing unfinished work.

Example:

```
git stash
git stash pop
```

---

### How to schedule script at 3 AM daily?

```
crontab -e
```

Add:

```
0 3 * * * /home/script.sh
```

---

### Difference between git fetch and git pull

git fetch downloads remote changes without merging.

git pull downloads and merges remote changes.

---

### What is LVM?

Logical Volume Manager allows flexible storage management. It enables resizing disks and managing storage dynamically.

---

# Task 4 – Work Organization

Checklist completed:

* All submissions from Day 1 to Day 27 pushed to GitHub
* git-commands.md updated
* Shell scripting cheat sheet completed
* GitHub profile organized
* Repositories cleaned and structured

---

# Task 5 – Teach Back

## Explaining File Permissions in Linux

Linux file permissions control who can read, write, or execute files. There are three types of users: owner, group, and others.

Each file has three permissions:

* Read (r)
* Write (w)
* Execute (x)

Example:

```
chmod 755 script.sh
```

This gives full permissions to the owner and read/execute permissions to group and others.

Proper permissions help secure the system and prevent unauthorized access.
