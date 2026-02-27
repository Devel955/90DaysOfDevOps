# 2026/day-18/day-18-scripting.md
# Day 18 – Shell Scripting: Functions & Intermediate Concepts
## Goals
- Write cleaner and reusable scripts using functions
- Understand strict mode: `set -euo pipefail`
- Use local variables to avoid variable leaks
- Build a real-world script: System Info Reporter
---
## Task 1: Basic Functions (functions.sh)
### Code
```bash
#!/bin/bash
# functions.sh
greet() {
  local name="$1"
  echo "Hello, $name!"
}
add() {
  local a="$1"
  local b="$2"
  echo $((a + b))
}
greet "Anand"
sum=$(add 10 20)
echo "Sum is: $sum"
Output
Hello, Anand!
Sum is: 30

# Task 2: Functions with Return Values (disk_check.sh)
#!/bin/bash
# disk_check.sh
check_disk() {
  echo "Disk usage for / :"
  df -h /
}
check_memory() {
  echo "Memory usage:"
  free -h
}
main() {
  echo "===== SYSTEM RESOURCE CHECK ====="
  echo
  check_disk
  echo
  check_memory
  echo
  echo "===== CHECK COMPLETE ====="
}
main
Output (example)
===== SYSTEM RESOURCE CHECK =====
Disk usage for / :
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1       30G  9.2G   21G  31% /
Memory usage:
               total        used        free      shared  buff/cache   available
Mem:           1.0Gi       200Mi       500Mi        10Mi       300Mi       700Mi
Swap:             0B          0B          0B

===== CHECK COMPLETE =====

Task 3: Strict Mode — set -euo pipefail (strict_demo.sh)
#!/bin/bash
# strict_demo.sh
set -euo pipefail

demo_u() {
  echo "Demo 1: set -u (Undefined Variable)"
  echo "$UNDEFINED_VAR"
}

demo_e() {
  echo "Demo 2: set -e (Command Failure)"
  ls /this/path/does/not/exist
}

demo_pipefail() {
  echo "Demo 3: set -o pipefail (Pipeline Failure)"
  cat /no/such/file | grep "hello"
}
main() {
  echo "===== STRICT MODE DEMO ====="
  echo
  # Uncomment ONE demo at a time:
  # demo_u
  demo_e
  # demo_pipefail
}
mainOutput (my run – Demo 2)
Demo 2: set -e (Command Failure)
ls: cannot access '/this/path/does/not/exist': No such file or directory

Task 4: Local Variables (local_demo.sh)
#!/bin/bash
# local_demo.sh
local_function() {
  local city="Kanpur"
  echo "Inside local_function: city=$city"
}
global_function() {
  city="Delhi"
  echo "Inside global_function: city=$city"
}
echo "Before: city=${city-Not Set}"
local_function
echo "After local_function: city=${city-Not Set}"
global_function
echo "After global_function: city=${city-Not Set}"
Output
Before: city=Not Set
Inside local_function: city=Kanpur
After local_function: city=Not Set
Inside global_function: city=Delhi
After global_function: city=Delhi

Task 5: System Info Reporter (system_info.sh)
#!/bin/bash
# system_info.sh
set -euo pipefail
section() {
  local title="$1"
  echo
  echo "========================================"
  echo "$title"
  echo "========================================"
}
print_host_os() {
  section "Hostname & OS Info"
  echo "Hostname : $(hostname)"
  if [ -f /etc/os-release ]; then
    . /etc/os-release
    echo "OS       : ${PRETTY_NAME}"
  else
    echo "OS       : $(uname -s) $(uname -r)"
  fi
}
print_uptime() {
  section "Uptime"
  uptime
}
print_disk_top5() {
  section "Disk Usage (Top 5 by size) - Current Directory"
  echo "Path     : $(pwd)"
  echo
  du -h --max-depth=1 2>/dev/null | sort -hr | head -n 5
}
print_memory() {
  section "Memory Usage"
  free -h
}
print_top_cpu() {
  section "Top 5 CPU-Consuming Processes"
  ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 6
}
main() {
  echo "SYSTEM INFO REPORT"
  echo "Generated at: $(date)"
  print_host_os
  print_uptime
  print_disk_top5
  print_memory
  print_top_cpu
  echo
  echo "Report complete ✅"
}
main
Output (example)
SYSTEM INFO REPORT
Generated at: Fri Feb 27 10:30:10 IST 2026
...
Report complete ✅

