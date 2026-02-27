# day-17-scripting.md

# Day 17 – Shell Scripting: Loops, Arguments & Error Handling
## Objective
Level up Bash scripting by using loops, handling command-line arguments, installing packages via script, and adding basic error handling.
---
# Task 1: For Loop
##  for_loop.sh
```bash
#!/bin/bash
for fruit in Apple Mango Banana Orange Grapes
do
  echo $fruit
done
# Output
Apple
Mango
Banana
Orange
Grapes

# count.sh
#!/bin/bash
for number in {1..10}
do
  echo $number
done
Output
1
2
3
4
5
6
7
8
9
10

# Task 2: While Loop
countdown.sh
#!/bin/bash
echo "Enter a number:"
read num
while [ $num -ge 0 ]
do
  echo $num
  num=$((num - 1))
done
echo "Done!"
Output Example
Enter a number:
5
5
4
3
2
1
0
Done!

Task 3: Command-Line Arguments
greet.sh
#!/bin/bash
if [ -z "$1" ]
then
  echo "Usage: ./greet.sh <name>"
else
  echo "Hello, $1!"
fi
Output
Without argument:
Usage: ./greet.sh <name>
With argument:
Hello, Anand!
args_demo.sh
#!/bin/bash
echo "Script name: $0"
echo "Total arguments: $#"
echo "All arguments: $@"
Output Example
Script name: ./args_demo.sh
Total arguments: 3
All arguments: DevOps AWS Docker

Task 4: Install Packages via Script
install_packages.sh
#!/bin/bash
# Check if script is run as root
if [ "$EUID" -ne 0 ]
then
  echo "Please run this script as root (sudo)."
  exit 1
fi
packages=("nginx" "curl" "wget")
for pkg in "${packages[@]}"
do
  echo "Checking $pkg..."
  if dpkg -s "$pkg" &> /dev/null
  then
    echo "$pkg is already installed. Skipping..."
  else
    echo "$pkg is NOT installed. Installing..."
    apt update -y
    apt install -y "$pkg"
  fi
  echo "---------------------------"
done
Output Example
Checking nginx...
nginx is already installed. Skipping...
---------------------------
Task 5: Error Handling
safe_script.sh
#!/bin/bash
set -e
mkdir /tmp/devops-test || echo "Directory already exists"
cd /tmp/devops-test || { echo "Failed to enter directory"; exit 1; }
touch test-file.txt || echo "Failed to create file"
echo "Script completed successfully!"
Key Learnings
for and while loops automate repetitive tasks.
$1, $#, $@, $0 help handle command-line arguments dynamically.
set -e exits script immediately if a command fails.
|| helps print custom error messages.
Checking $EUID ensures the script runs with proper privileges.
utomation and error handling are essential DevOps skills.
Conclusion
Day 17 focused on practical Bash scripting skills.
These concepts are fundamental for DevOps automation, server management, and production scripting.

---
