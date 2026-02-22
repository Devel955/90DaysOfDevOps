# Day 11 Challenge

## Files & Directories Created
- devops-file.txt
- team-notes.txt
- project-config.yaml
- app-logs/
- heist-project/
- heist-project/vault/gold.txt
- heist-project/plans/strategy.conf
- bank-heist/
- bank-heist/access-codes.txt
- bank-heist/blueprints.pdf
- bank-heist/escape-plan.txt

---

## Ownership Changes

- devops-file.txt: ubuntu:ubuntu → tokyo:ubuntu → berlin:ubuntu
- team-notes.txt: ubuntu:ubuntu → ubuntu:heist-team
- project-config.yaml: ubuntu:ubuntu → professor:heist-team
- app-logs/: ubuntu:ubuntu → berlin:heist-team
- heist-project/: ubuntu:ubuntu → professor:planners
- heist-project/vault/gold.txt: ubuntu:ubuntu → professor:planners
- heist-project/plans/strategy.conf: ubuntu:ubuntu → professor:planners
- access-codes.txt: ubuntu:ubuntu → tokyo:vault-team
- blueprints.pdf: ubuntu:ubuntu → berlin:tech-team
- escape-plan.txt: ubuntu:ubuntu → nairobi:vault-team

---

## Commands Used

```bash
ls -l

touch devops-file.txt
sudo chown tokyo devops-file.txt
sudo chown berlin devops-file.txt

touch team-notes.txt
sudo chgrp heist-team team-notes.txt

touch project-config.yaml
sudo chown professor:heist-team project-config.yaml

mkdir app-logs
sudo chown berlin:heist-team app-logs

mkdir -p heist-project/vault
mkdir -p heist-project/plans
touch heist-project/vault/gold.txt
touch heist-project/plans/strategy.conf
sudo chown -R professor:planners heist-project/

mkdir bank-heist
touch bank-heist/access-codes.txt
touch bank-heist/blueprints.pdf
touch bank-heist/escape-plan.txt

sudo chown tokyo:vault-team bank-heist/access-codes.txt
sudo chown berlin:tech-team bank-heist/blueprints.pdf
sudo chown nairobi:vault-team bank-heist/escape-plan.txt

What I Learned
Every file in Linux has an owner and a group.
The chown command is used to change file owner and group.
The -R option helps change ownership for entire directories.


---

# ✅ Step 3: Save & Exit
Press:
👉 `CTRL + X` → `Y` → `Enter`
---
# ✅ Step 4: Check File
```bash
cat day-11-file-ownership.md
