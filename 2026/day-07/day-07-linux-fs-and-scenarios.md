# Day 07 – Linux File System Hierarchy

## Part 1: File System Hierarchy

# Purpose: Root directory. All files start here.

Command:
ls -l /
I would use this when checking system structure.
---
### /home
Purpose: Stores user files.
Command:
ls -l /home
I would use this when accessing documents.
---
### /root
Purpose: Root user home directory.
Command:
ls -l /root
I would use this for admin work.
---
### /etc
Purpose: Configuration files.
Command:
ls -l /etc
I would use this to edit settings.
---
### /var/log
Purpose: System logs.
Command:
ls -l /var/log
I would use this to troubleshoot.
---
### /tmp
Purpose: Temporary files.
Command:
ls -l /tmp
I would use this for temp data.
---
### /bin
Purpose: Basic commands.
Command:
ls -l /bin
I would use this for shell commands.
---
### /usr/bin
Purpose: User programs.
Command:
ls -l /usr/bin
I would use this for tools.
---
### /opt
Purpose: External software.
Command:
ls -l /opt
---

## Part 2: Scenario-Based Practice

---

### Solved Example: Check Service Status (nginx)

# Step 1:
systemctl status nginx  
Why: Check if service is running or failed.

# Step 2:
systemctl list-units --type=service  
Why: List available services if nginx is not found.

# Step 3:
systemctl is-enabled nginx  
Why: Check if service starts on boot.

What I learned: Always check status first, then investigate.

---

### Scenario 1: Service Not Starting (myapp)

# Step 1:
systemctl status myapp  
Result: Service not found (example service).

# Step 2:
systemctl list-units --type=service  
Why: List running services.

# Step 3:
systemctl status ssh  
Why: Check existing service.

# Step 4:
systemctl is-enabled ssh  
Why: Check auto start.
What I learned: If service is not found, test with another running service.

---

### Scenario 2: High CPU Usage

Step 1:
top  
Why: Check live CPU usage.

Step 2:
ps aux --sort=-%cpu | head -10  
Why: Find top CPU processes.

Step 3:
ps -p <PID> -f  
Why: Check process details.

Step 4:
kill -9 <PID>  
Why: Stop heavy process if needed.

---

### Scenario 3: Finding Service Logs (Docker)

Step 1:
systemctl status docker  
Why: Check service status.

Step 2:
journalctl -u docker -n 50  
Why: View recent logs.

Step 3:
journalctl -u docker -f  
Why: Follow logs live.

Note: Docker not installed, so tested with ssh.

systemctl status ssh  
journalctl -u ssh -n 20  
journalctl -u ssh -f

---

### Scenario 4: File Permission Issue

Step 1:
ls -l ~/backup.sh  
Why: Check file permission.

Step 2:
chmod +x ~/backup.sh  
Why: Add execute permission.

Step 3:
ls -l ~/backup.sh  
Why: Verify permission.

Step 4:
./backup.sh  
Why: Run script.
