# DAY 08 – CLOUD DEPLOYMENT
# (AWS EC2 + Nginx + Docker)

# Aim
# EC2 server launch karna, SSH se connect karna, Docker & Nginx install karna, aur logs nikalna.
# Step 1: Launch EC2
AWS → EC2 → Launch Instance
Ubuntu 22.04 select
Instance: t2.micro
Key pair create
Security Group:
SSH (22) → My IP
HTTP (80) → Anywhere
Launch 
EC2 = Cloud ka virtual computer

# Step 2: SSH Connect
command:ssh -i 90days-devops-day08-key.pem ubuntu@<IP>
# erver me login karne ke liye

# Step 3: System Update
command:sudo apt update && sudo apt upgrade -y
# System fresh karta hai.

# Step 4: Install Docker
command:sudo apt install docker.io -y
sudo systemctl enable --now docker
# Docker = Container tool

# Step 5: Install Nginx
 command:sudo apt install nginx -y
 command:sudo systemctl enable --now nginx
 # “Welcome to nginx” = Success ✅

 # Step 6: View Logs
 command:sudo tail -n 20 /var/log/nginx/access.log
 # Website traffic dekhne ke liye

 # Step 7: Save Logs
 command:sudo cp /var/log/nginx/access.log ~/nginx-logs.txt
 command:sudo chown ubuntu:ubuntu nginx-logs.txt
# Log file ban gayi

# Step 8: Download Logs
command:scp -i 90days-devops-day08-key.pem ubuntu@<IP>:~/nginx-logs.txt .
# Server se laptop me file

# Challenges Faced
SSH permission denied
Key file path problem
HTTP port blocked
File permission issue
# Sab fix kiya step by step

# What I Learned
EC2 launch karna
SSH use karna
Docker install karna
Nginx deploy karna
Logs handle karna

# Conclusion
Is task se mujhe real cloud deployment, security setup, aur troubleshooting seekhne ko mila.
Ye DevOps ke liye important skill hai.

