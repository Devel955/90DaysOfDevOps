# Day2 linux-architecture-notes.md

# Computer program machine samajhta hain.
# the Linux kernel is the core of the operating system, acting as a bridge between software and the physical hardware.
# And kernel is also (core of linux )-linux kernel.
# Implementing network protocols (like TCP/IP) and managing network communication.
# User Space
# The user space is the environment where all non-kernel processes and applications run. 
# These processes operate in a non-privileged user mode, meaning they cannot access or modify kernel memory directly and must use system calls to request kernel services.
# All user-facing software, such as web browsers, office software, and development tools.

# init/systemd
# Systemd is the modern default init system and service manager for most Linux distributions (Ubuntu, Fedora, Debian), running as Process ID 1 to manage system boot,services, and daemons.

# How processes are created and managed.
# Processes are created by the Operating System (OS) or existing processes using system calls like fork() (Unix/Linux) or CreateProcess() (Windows), which allocate resources and initialize a Process Control Block (PCB)>>. Management involves scheduling, 
# tracking, and coordinating processes through their lifecycle (New, Ready, Running, Waiting, Terminated) to ensure CPU efficiency and data consistency.

# What systemd does and why it matters.
# Systemd is a comprehensive system and service manager for Linux operating systems, acting as the default initialization system (PID 1) 
  # for the vast majority of modern Linux distributions. 
# responsible for booting the system, managing services (daemons), and controlling resources. 
