# DevOps Intern Assignment

## 1. Project Overview

This project demonstrates the complete setup of a cloud-based Linux environment on AWS, automation of system monitoring, integration with AWS CloudWatch, and implementation of proactive alerting using systemd and shell scripting.

All tasks were performed on an Ubuntu 22.04 EC2 instance using AWS Free Tier resources. The goal was not only to configure the environment but also to ensure it is fully reproducible, well-documented, and production-ready.

---

## 2. Environment Setup (Part 1)

### 2.1 EC2 Instance Creation

An Ubuntu EC2 instance was launched with the following configuration:

* AMI: Ubuntu Server 22.04 LTS
* Instance Type: t2.micro
* Key Pair: dockerkey.pem
* Security Group: SSH (Port 22) allowed

Connection command used:

```
ssh -i dockerkey.pem ubuntu@<public-ip>
```

### 2.2 User Creation & Privilege Setup

Commands executed:

```
sudo adduser devops_intern
sudo visudo
```

Added line:

```
devops_intern ALL=(ALL) NOPASSWD:ALL
```

### 2.3 Hostname Configuration

```
sudo hostnamectl set-hostname shahroze-devops
```

Verification commands used:

```
hostname
cat /etc/passwd | grep devops_intern
sudo whoami
```

Sample Output:

```
shahroze-devops
devops_intern:x:1001:1001::/home/devops_intern:/bin/bash
root
```

---

## 3. Web Server Setup (Part 2)

### 3.1 Installation of Nginx

```
sudo apt update
sudo apt install nginx -y
```

### 3.2 HTML Page Creation

File edited:

```
sudo nano /var/www/html/index.html
```

Metadata commands used:

```
curl http://169.254.169.254/latest/meta-data/instance-id
uptime -p
```

HTML Content Example:

```
<html>
<body>
<h1>Shahroze Baig</h1>
<h2>Instance ID: i-0abcd12345</h2>
<h3>Server Uptime: up 1 hour, 20 minutes</h3>
</body>
</html>
```

Accessed via:

```
http://<EC2-Public-IP>
```

---

## 4. Monitoring Script (Part 3)

### 4.1 system_report.sh Script

Location:

```
/usr/local/bin/system_report.sh
```

Script content:

```bash
#!/bin/bash

echo "----------------------------------------"
echo "Date & Time: $(date)"
echo "Uptime: $(uptime -p)"

CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2 + $4}')
echo "CPU Usage: $CPU%"

MEMORY=$(free | grep Mem | awk '{printf "%.2f", $3/$2 * 100}')
echo "Memory Usage: $MEMORY%"

DISK=$(df -h / | awk 'NR==2 {print $5}')
echo "Disk Usage: $DISK"

echo "Top 3 Processes by CPU:"
ps -eo pid,comm,%cpu --sort=-%cpu | head -4

# Disk Alert
DISK_NUM=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')

if [ $DISK_NUM -gt 80 ]; then
    echo "WARNING: Disk usage is above 80% (Current: $DISK_NUM%)" \
    | mail -s "Disk Alert on $(hostname)" admin@example.com
fi

echo "----------------------------------------"
```

Permissions:

```
sudo chmod +x /usr/local/bin/system_report.sh
```

---

## 5. Logging & Execution

### Log File:

```
/var/log/system_report.log
```

Cron Configuration:

```
*/5 * * * * /usr/local/bin/system_report.sh >> /var/log/system_report.log
```

Sample Log Output:

```
----------------------------------------
Date & Time: Thu Nov 20 20:35:01 UTC 2025
Uptime: up 1 hour, 8 minutes
CPU Usage: 7.4%
Memory Usage: 49.87%
Disk Usage: 40%
Top 3 Processes by CPU:
3337 sshd 0.2
3196 aws 0.1
585 containerd 0.1
----------------------------------------
```

---

## 6. systemd Automation (Bonus A)

Service File:

```
/etc/systemd/system/system_report.service
```

```
[Unit]
Description=System Report Service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/system_report.sh
```

Timer File:

```
/etc/systemd/system/system_report.timer
```

```
[Unit]
Description=Run system_report.sh every 5 minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

Activation:

```
sudo systemctl daemon-reexec
sudo systemctl enable system_report.timer
sudo systemctl start system_report.timer
```

Verification:

```
systemctl list-timers --all
```

---

## 7. AWS CloudWatch Integration (Part 4)

### 7.1 AWS CLI Configuration

```
aws configure
```

### 7.2 Log Group & Stream

```
aws logs create-log-group --log-group-name /devops/intern-metrics
aws logs create-log-stream --log-group-name /devops/intern-metrics --log-stream-name system-report-stream
```

### 7.3 Log Upload Steps

```
jq -R -s 'split("\n") | map(select(. != "")) | map({timestamp:(now|floor*1000), message:.})' /var/log/system_report.log > events.json
```

Upload:

```
aws logs put-log-events --log-group-name /devops/intern-metrics --log-stream-name system-report-stream --sequence-token <TOKEN> --log-events file://events.json
```

---

## 8. Reproduction Steps

To recreate this environment:

1. Launch Ubuntu EC2 instance
2. SSH using key pair
3. Create devops user and configure sudo
4. Change hostname
5. Install Nginx and deploy page
6. Create monitoring script
7. Configure systemd timer
8. Install AWS CLI
9. Setup CloudWatch and upload logs

---

