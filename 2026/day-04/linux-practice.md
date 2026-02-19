# Day 04 Linux practice:
# Process Checks:
Command: ps aux | head
output:
  PID    PPID    PGID     WINPID   TTY         UID    STIME COMMAND
     1807    1806    1807       7468  pty0      197609 08:29:44 /usr/bin/bash
     1831    1807    1830      19816  pty0      197609 08:31:43 /usr/bin/head
     1830    1807    1830      11556  pty0      197609 08:31:43 /usr/bin/ps
     1806       1    1806      16368  ?         197609 08:29:44 /usr/bin/mintty

# Command: ps aux | grep bash
output:
        1807    1806    1807       7468  pty0      197609 08:29:44 /usr/bin/bash
         1837    1807    1836      11024  pty0      197609 08:33:06 /usr/bin/bash

# Command: ps -ef | head
output:  UID     PID    PPID  TTY        STIME COMMAND
    Asus    1843    1807 pty0     08:34:59 /usr/bin/bash --login -i
    Asus    1807    1806 pty0     08:29:44 /usr/bin/bash --login -i
    Asus    1806       1 ?        08:29:44 usr/bin/mintty --nodaemon -o AppID=GitForWindows.Bash -o AppLaunchCmd=C:\Program Files\Git\git-bash.exe -o AppName=Git Bash -i C:\Program Files\Git\git-bash.exe --store-taskbar-properties -- /usr/bin/bash --login -i
    Asus    1842    1807 pty0     08:34:59 ps -ef
-------

# Service / System Checks :
Command: systemctl status ssh
output: Feb 19 03:16:36 LAPTOP-C7SGCHL5 systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
Feb 19 03:16:36 LAPTOP-C7SGCHL5 sshd[1448]: Server listening on 0.0.0.0 port 22.
Feb 19 03:16:36 LAPTOP-C7SGCHL5 sshd[1448]: Server listening on :: port 22.
Feb 19 03:16:36 LAPTOP-C7SGCHL5 systemd[1]: Started ssh.service - OpenBSD Secure Shell server.

Command: systemctl list-units --type=service | head
output:  UNIT                                     LOAD   ACTIVE SUB     DESCRIPTION
  console-getty.service                    loaded active running Console Getty
  console-setup.service                    loaded active exited  Set console font and keymap
  cron.service                             loaded active running Regular background program processing daemon
  dbus.service                             loaded active running D-Bus System Message Bus
  getty@tty1.service                       loaded active running Getty on tty1
  jenkins.service                          loaded active running Jenkins Continuous Integration Server
  keyboard-setup.service                   loaded active exited  Set the console keyboard layout
  kmod-static-nodes.service                loaded active exited  Create List of Static Device Nodes
  polkit.service                           loaded active running Authorization Manager

## Log Checks:
Command: journalctl -u ssh -n 20
output: Feb 19 03:16:36 LAPTOP-C7SGCHL5 systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
Feb 19 03:16:36 LAPTOP-C7SGCHL5 sshd[1448]: Server listening on 0.0.0.0 port 22.
Feb 19 03:16:36 LAPTOP-C7SGCHL5 sshd[1448]: Server listening on :: port 22.
Feb 19 03:16:36 LAPTOP-C7SGCHL5 systemd[1]: Started ssh.service - OpenBSD Secure Shell server.

Command: tail -n 20 /var/log/syslog
Output:2026-02-19T03:24:37.643938+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[36mINFO#033[0m Reconnecting to Windows host in 60 seconds
2026-02-19T03:24:37.643984+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[37mDEBUG#033[0m Updated systemd status to "Not connected: waiting to retry"
2026-02-19T03:25:04.782060+00:00 LAPTOP-C7SGCHL5 systemd-resolved[112]: Clock change detected. Flushing caches.
2026-02-19T03:25:37.887511+00:00 LAPTOP-C7SGCHL5 systemd-resolved[112]: Clock change detected. Flushing caches.
2026-02-19T03:25:38.863088+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[36mINFO#033[0m Daemon: connecting to Windows Agent
2026-02-19T03:25:38.863252+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[37mDEBUG#033[0m Updated systemd status to "Connecting"
2026-02-19T03:25:38.870532+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[33mWARNING#033[0m Daemon: could not connect to Windows Agent: could not get address: could not read agent port file "/mnt/c/Users/Asus/.ubuntupro/.address": open /mnt/c/Users/Asus/.ubuntupro/.address: no such file or directory
2026-02-19T03:25:38.870656+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[36mINFO#033[0m Reconnecting to Windows host in 60 seconds
2026-02-19T03:25:38.870708+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[37mDEBUG#033[0m Updated systemd status to "Not connected: waiting to retry"
2026-02-19T03:26:10.758624+00:00 LAPTOP-C7SGCHL5 systemd-resolved[112]: Clock change detected. Flushing caches.
2026-02-19T03:26:10.770436+00:00 LAPTOP-C7SGCHL5 systemd[1]: man-db.service - Daily man-db regeneration was skipped because of an unmet condition check (ConditionACPower=true).
2026-02-19T03:26:39.354167+00:00 LAPTOP-C7SGCHL5 systemd[1]: Starting systemd-tmpfiles-clean.service - Cleanup of Temporary Directories...
2026-02-19T03:26:39.525080+00:00 LAPTOP-C7SGCHL5 systemd[1]: systemd-tmpfiles-clean.service: Deactivated successfully.
2026-02-19T03:26:39.526454+00:00 LAPTOP-C7SGCHL5 systemd[1]: Finished systemd-tmpfiles-clean.service - Cleanup of Temporary Directories.
2026-02-19T03:26:39.552984+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[36mINFO#033[0m Daemon: connecting to Windows Agent
2026-02-19T03:26:39.553169+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[37mDEBUG#033[0m Updated systemd status to "Connecting"
2026-02-19T03:26:39.557381+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[33mWARNING#033[0m Daemon: could not connect to Windows Agent: could not get address: could not read agent port file "/mnt/c/Users/Asus/.ubuntupro/.address": open /mnt/c/Users/Asus/.ubuntupro/.address: no such file or directory
2026-02-19T03:26:39.557448+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[36mINFO#033[0m Reconnecting to Windows host in 60 seconds
2026-02-19T03:26:39.558649+00:00 LAPTOP-C7SGCHL5 wsl-pro-service[182]: #033[37mDEBUG#033[0m Updated systemd status to "Not connected: waiting to retry"
2026-02-19T03:26:43.692464+00:00 LAPTOP-C7SGCHL5 systemd-resolved[112]: Clock change detected. Flushing caches.

## Mini Troubleshooting Steps

1. Checked system performance using top
2. Verified ssh service status
3. Installed and enabled openssh-server
4. Checked ssh logs using journalctl
5. Reviewed system logs using tail
6. Confirmed ssh is running on port 22



       


