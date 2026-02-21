# Day 10 – Linux File Handling & Permissions Practice

## Task 1 – Create Files

### Files Created
- devops.txt (empty file)
- notes.txt (with content)
- script.sh (shell script)

### Commands Used
```bash
# Create empty file
touch devops.txt

# Create file with content
echo "This is my DevOps" > notes.txt

# Create script
vim script.sh
# inside file:
echo "Hello DevOps"

# Make executable and run
chmod +x script.sh
./script.sh

#Task 2 – Read Files
Commands Used
# Read notes file
cat notes.txt
# Open script in read-only mode
vim -R script.sh
# First 5 lines of passwd
head -n 5 /etc/passwd

# Last 5 lines of passwd
tail -n 5 /etc/passwd
Output Verification
notes.txt displayed correctly
script.sh opened in read-only mode
/etc/passwd first and last lines verified

# Task 3 – Understand Permissions
Permission Format
rwxrwxrwx
Owner | Group | Others
Permission Values
r = read (4)
w = write (2)
x = execute (1)
Check Command
ls -l devops.txt notes.txt script.sh

Task 4 – Modify Permissions
Commands Used:# Make script executable
chmod +x script.sh

# Make devops.txt read-only
chmod a-w devops.txt

# Set notes.txt to 640
chmod 640 notes.txt

# Create project directory
mkdir project
chmod 755 project
Verification:
ls -l devops.txt notes.txt script.sh
ls -ld project.

Task 5 – Test Permissions
Write Test (Read-only File)
echo "test line" >> devops.txt
Output:
Permission denied
xecute Test (Without Permission)
chmod -x script.sh
./script.sh
Output:
Permission denied
Restore Permission
chmod +x script.sh

Learning Outcome
Learned how to create and manage files.
Understood Linux permission structure.
Practiced modifying permissions using chmod.
Tested security using permission errors.
Improved command-line troubleshooting skills.

Conclusion<img width="745" height="449" alt="Screenshot 2026-02-21 201707" src="https://github.com/user-attachments/assets/99683ae3-de47-4e5f-a022-dc4e42e168b9" />

This practice helped me understand real-world Linux file handling and permission management, which is important for DevOps and system administration.





