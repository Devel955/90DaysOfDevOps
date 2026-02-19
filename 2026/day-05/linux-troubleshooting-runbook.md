# linux-troubleshooting-runbook.md
# Target Service / Process
Service: SSH
Process: sshd
PID: 1448
Host: Ubuntu (WSL / EC2 environment)
Verification Commands:
systemctl status ssh
ps -o pid,ppid,pcpu,pmem,stat,etime,comm -p 1448
ss -tulpn | grep -E ':22|ssh'
# Observation
SSH service is active and running
sshd process running under systemd (PPID 1)
CPU usage: 0.0%
Memory usage: 0.2%
Listening on port 22 (IPv4 + IPv6)
No zombie or stuck process detected

# Snapshot: CPU & Memory
Commands Used:top 
    free -h
  # Observation
  Load average near 0.00 (system idle)
  No abnormal CPU spikes
  Memory usage within safe range
  No swap pressure
 System performance stable

# Snapshot: Disk & IO
Commands Used: df -h
du -sh /var/log
# Observation
Root filesystem has sufficient free space
No partition near 100%
Log directory size normal
No disk exhaustion risk
Disk health good

# Snapshot: Network
  Commands Used:ss -tulpn | grep -E ':22|ssh'
          curl -I http://localhost
# Observation
SSH listening on port 22 (0.0.0.0 & ::)
No HTTP service running (connection refused on port 80 expected)
No unexpected open ports detected
Network behaving as expected

# Logs Reviewed
 Commands Used:journalctl -u ssh -n 50 --no-pager
  sudo tail -n 50 /var/log/auth.log 
# Observation
SSH service started successfully
No recent errors in logs
No failed login attempts detected
No suspicious authentication activity
System secure and clean

6️⃣ Quick Findings
SSH service healthy
No CPU bottleneck
No memory pressure
Disk space sufficient
Network properly configured
No security alerts
[Linux Troubleshooting Runbook -Day5 (1).pdf](https://github.com/user-attachments/files/25409999/Linux.Troubleshooting.Runbook.-Day5.1.pdf)
🔵 Overall System Status: Stable & Operational

