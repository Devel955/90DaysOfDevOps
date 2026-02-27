# Day 19 – Shell Scripting Project: Log Rotation, Backup & Cron Automation

## 🔧 Features Implemented

### 1️⃣ Log Rotation Script
- Compresses `.log` files older than 7 days
- Deletes `.gz` files older than 30 days
- Validates directory existence
- Prints compressed & deleted file counts

### 2️⃣ Server Backup Script
- Creates timestamped `.tar.gz` archives
- Verifies archive creation
- Prints archive name & size
- Implements 14-day retention cleanup
- Proper error handling with strict mode

### 3️⃣ Cron Scheduling
- Daily log rotation (2 AM)
- Weekly server backup (Sunday 3 AM)
- Health check every 5 minutes
- Combined maintenance job (1 AM)

### 4️⃣ Combined Maintenance Script
- Calls log rotation + backup
- Logs output to `/var/log/maintenance.log`
- Uses structured timestamp logging
- Production-style safe scripting (`set -euo pipefail`)

---

## 🛡 Best Practices Used
- Strict mode: `set -euo pipefail`
- Functions with local variables
- Absolute paths in cron jobs
- Output redirection & logging
- Retention policy implementation
---
## 🚀 Learning Outcome
This project helped build production-ready scripting skills including automation, log management, backup strategies, cron scheduling, and safe error handling

## 📂 Folder Structure
