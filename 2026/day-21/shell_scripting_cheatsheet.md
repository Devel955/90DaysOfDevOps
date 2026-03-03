Task 1: Basics
Shebang
#!/bin/bash
Defines interpreter. Ensures script runs with Bash.
Run Script
chmod +x script.sht
./script.sh
bash script.sh

Comments
# Single line comment
echo "Hello"  # Inline comment
Variables
NAME="Anand"
echo $NAME
echo "$NAME"
echo '$NAME'
Use quotes to prevent word splitting.

Read Input
read -p "Enter name: " NAME
echo "Hello $NAME"
Command-Line Arguments
echo $0   # Script name
echo $1   # First argument
echo $#   # Number of arguments
echo $@   # All arguments
echo $?   # Exit status
Task 2: Operators & Conditionals
String Comparisons
[ "$a" = "$b" ]
[ "$a" != "$b" ]
[ -z "$a" ]
[ -n "$a" ]
Integer Comparisons
[ $a -eq $b ]
[ $a -ne $b ]
[ $a -lt $b ]
[ $a -gt $b ]
[ $a -le $b ]
[ $a -ge $b ]
File Tests
[ -f file ]
[ -d dir ]
[ -e file ]
[ -r file ]
[ -w file ]
[ -x file ]
[ -s file ]
If / Elif / Else
if [ condition ]; then
  echo "True"
elif [ condition ]; then
  echo "Else if"
else
  echo "False"
fi
Logical Operators
[ condition1 ] && echo "True"
[ condition1 ] || echo "False"
[ ! condition ]
Case Statement
case $1 in
  start) echo "Starting";;
  stop) echo "Stopping";;
  *) echo "Invalid option";;
esac
Task 3: Loops
For (List)
for fruit in apple mango banana; do
  echo $fruit
done
For (C Style)
for ((i=1; i<=5; i++)); do
  echo $i
done
While
i=1
while [ $i -le 5 ]; do
  echo $i
  ((i++))
done
Until
i=1
until [ $i -gt 5 ]; do
  echo $i
  ((i++))
done
Break / Continue
break
continue
Loop Files
for file in *.log; do
  echo $file
done
While Read
while read line; do
  echo $line
done < file.txt
🔹 Task 4: Functions
Define
greet() {
  echo "Hello"
}
Call
greet
Arguments
add() {
  echo $(($1 + $2))
}
add 5 3
Return vs Echo
myfunc() {
  return 1
}
echo $?
Local Variable
myfunc() {
  local var="Hello"
}
Task 5: Text Processing
Grep
grep -i "error" file.log
grep -r "text" .
grep -c "error" file.log
grep -n "error" file.log
grep -v "info" file.log
grep -E "error|failed" file.log
Awk
awk '{print $1}' file
awk -F: '{print $1}' /etc/passwd
awk 'BEGIN {print "Start"} {print $1} END {print "End"}' file
Sed
sed 's/old/new/g' file
sed -i 's/foo/bar/g' file
sed '/pattern/d' file
Cut
cut -d: -f1 /etc/passwd
Sort
sort file
sort -n file
sort -r file
sort -u file
Uniq
uniq file
uniq -c file
Tr
tr 'a-z' 'A-Z'
tr -d '0-9'
Wc
wc -l file
wc -w file
wc -c file
Head / Tail
head -n 5 file
tail -n 5 file
tail -f file
Task 6: Useful One-Liners
Delete files older than 7 days
find . -type f -mtime +7 -delete
Count lines in all logs
wc -l *.log
Replace text in multiple files
sed -i 's/old/new/g' *.txt
Check service status
systemctl is-active nginx
Disk usage alert
df -h | awk '$5+0 > 80 {print $0}'
Tail and filter errors
tail -f app.log | grep --line-buffered "ERROR"
Task 7: Error Handling & Debugging
Exit Codes
echo $?
exit 0
exit 1
Strict Mode
set -e
set -u
set -o pipefail
Debug Mode
set -x
Trap
trap 'echo "Cleaning up..."; rm -f temp.txt' EXIT



