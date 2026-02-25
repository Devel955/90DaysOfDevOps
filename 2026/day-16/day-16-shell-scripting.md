# Day 16 -- Shell Scripting Basics

## Introduction

Today I learned the basics of shell scripting including shebang,
variables, user input, and if-else conditions.

------------------------------------------------------------------------

## Task 1: First Script (hello.sh)

``` bash
#!/bin/bash
echo "Hello, DevOps!"
```

Commands:

    chmod +x hello.sh
    ./hello.sh

Output:

    Hello, DevOps!

If we remove shebang, script may not run properly with ./hello.sh.

------------------------------------------------------------------------

## Task 2: Variables (variables.sh)

``` bash
#!/bin/bash

NAME="Anand"
ROLE="DevOps Engineer"

echo "Hello, I am $NAME and I am a $ROLE"
```

Output:

    Hello, I am Anand and I am a DevOps Engineer

Single quotes do not expand variables, double quotes do.

------------------------------------------------------------------------

## Task 3: User Input (greet.sh)

``` bash
#!/bin/bash

read -p "Enter your name: " NAME
read -p "Enter your favourite tool: " TOOL

echo "Hello $NAME, your favourite tool is $TOOL"
```

------------------------------------------------------------------------

## Task 4: If-Else Conditions

### check_number.sh

``` bash
#!/bin/bash

read -p "Enter a number: " NUM

if [ $NUM -gt 0 ]; then
  echo "Positive"
elif [ $NUM -lt 0 ]; then
  echo "Negative"
else
  echo "Zero"
fi
```

### file_check.sh

``` bash
#!/bin/bash

read -p "Enter filename: " FILE

if [ -f "$FILE" ]; then
  echo "File exists"
else
  echo "File not found"
fi
```

------------------------------------------------------------------------

## Task 5: Server Check (server_check.sh)

``` bash
#!/bin/bash

SERVICE="ssh"

read -p "Do you want to check status? (y/n): " ANS

if [ "$ANS" == "y" ]; then
  systemctl is-active $SERVICE
elif [ "$ANS" == "n" ]; then
  echo "Skipped"
else
  echo "Invalid input"
fi
```

------------------------------------------------------------------------

## What I Learned

1.  Shebang defines interpreter
2.  Variables store data
3.  read takes user input
4.  if-else handles conditions
5.  Shell scripting helps automation

------------------------------------------------------------------------

## Conclusion

Shell scripting is very useful for DevOps automation.
