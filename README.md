# CloudWatch-Agent-Setup-on-Ubuntu-EC2

## Overview

This SOP explains how to deploy and configure the **Amazon CloudWatch Agent on Ubuntu EC2 instances** to collect:

* System metrics (CPU, Memory, Disk)
* System logs (`syslog`)
* CloudWatch alarms with SNS notifications

After completing this guide, you will have:

* CloudWatch Agent running
* Metrics visible in the **CWAgent namespace**
* Logs flowing into **CloudWatch Logs**
* Alerts configured via **SNS**

---

# PART 1 — Launch Ubuntu EC2 Instance

## 1. Launch Instance

| Setting       | Value                    |
| ------------- | ------------------------ |
| AMI           | Ubuntu Server 22.04 LTS  |
| Instance Type | t2.micro / t3.micro      |
| Key Pair      | Create & download `.pem` |
| Network       | Public Subnet            |
| Public IP     | Enable                   |

---

## 2. Security Group

| Type         | Port | Source    |
| ------------ | ---- | --------- |
| SSH          | 22   | Your IP   |
| All Outbound | ALL  | 0.0.0.0/0 |

---

## 3. Attach IAM Role

Attach a role containing the following policies:

* `CloudWatchAgentServerPolicy`
* `AmazonSSMManagedInstanceCore`

### Why?

| Policy                       | Purpose                               |
| ---------------------------- | ------------------------------------- |
| CloudWatchAgentServerPolicy  | Send logs & metrics to CloudWatch     |
| AmazonSSMManagedInstanceCore | Future automation & remote management |

---

## 4. Connect to Instance

```bash
ssh -i key.pem ubuntu@EC2_PUBLIC_IP
```

---

# PART 2 — Clean Old Installation (Important)

This prevents **broken agent installs**.

## Remove previous agent

```bash
sudo dpkg --purge amazon-cloudwatch-agent
```

## Remove leftover directories

```bash
sudo rm -rf /opt/aws/amazon-cloudwatch-agent
```

## Clean unused packages

```bash
sudo apt-get autoremove -y
```

---

# PART 3 — Install CloudWatch Agent

## Update system

```bash
sudo apt update
```

## Download agent

```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb -O cwagent.deb
```

## Install agent

```bash
sudo dpkg -i cwagent.deb
```

## Verify installation

```bash
ls /opt/aws/amazon-cloudwatch-agent/etc/
```

Expected output:

```
amazon-cloudwatch-agent.d
common-config.toml
```

If missing → reinstall the agent.

---

# PART 4 — Install Dependency (Important)

CloudWatch metrics require **collectd**.

## Install collectd

```bash
sudo apt install collectd -y
```

## Verify installation

```bash
ls /usr/share/collectd/types.db
```

Expected output:

```
/usr/share/collectd/types.db
```

---

# PART 5 — Configure CloudWatch Agent

## Run configuration wizard

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

---

## Wizard Answers

| Question             | Answer            |
| -------------------- | ----------------- |
| OS                   | 1 (Linux)         |
| EC2 or On-Prem       | 1 (EC2)           |
| StatsD               | 2 (No)            |
| Existing Config      | 2 (No)            |
| Monitor host metrics | 1 (Yes)           |
| CPU metrics          | 1 (Yes)           |
| Memory metrics       | 1 (Yes)           |
| Disk metrics         | 1 (Yes)           |
| Disk resources       | `*`               |
| Collect logs         | 1 (Yes)           |
| Log file path        | `/var/log/syslog` |
| Storage class        | 1 (Standard)      |
| Log stream name      | `{instance_id}`   |
| Retention            | 7 days            |
| Additional logs      | 2 (No)            |
| X-Ray traces         | 2 (No)            |
| Store config in SSM  | 2 (No)            |

---

## Verify configuration file

```bash
ls /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

Expected:

```
/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

---

# PART 6 — Start CloudWatch Agent

## Start agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
-s
```

---

## Verify service

```bash
sudo systemctl status amazon-cloudwatch-agent
```

Expected:

```
active (running)
```

---

## Enable auto start

```bash
sudo systemctl enable amazon-cloudwatch-agent
```

---

# PART 7 — Verify Metrics in AWS

Navigate to:

```
CloudWatch → Metrics → CWAgent
```

Expected metrics:

* `cpu_usage_user`
* `mem_used_percent`
* `disk_used_percent`

---

# PART 8 — Verify Logs

Navigate to:

```
CloudWatch → Logs → Log Groups
```

Expected log group:

```
/ec2/syslog
```

Open the log stream → Instance ID → Logs should appear.

---

# PART 9 — Create CloudWatch Alarm

## 1. Create SNS Topic

Navigate:

```
SNS → Topics → Create Topic
```

Type:

```
Standard
```

---

## 2. Create Subscription

Protocol:

```
Email
```

Confirm the email subscription.

---

## 3. Create Alarm

Navigate:

```
CloudWatch → Alarms → Create Alarm
```

Metric:

```
CWAgent → cpu_usage_user
```

Condition:

```
> 80%
```

Period:

```
5 minutes
```

Action:

```
Send notification to SNS topic
```

---

# PART 10 — Test Alarm

Generate CPU load:

```bash
yes > /dev/null
```

Stop the test:

```bash
pkill yes
```

Alarm should trigger in **CloudWatch**.

---

# PART 11 — Troubleshooting

## Check service status

```bash
sudo systemctl status amazon-cloudwatch-agent
```

## Restart agent

```bash
sudo systemctl restart amazon-cloudwatch-agent
```

## Check agent logs

```bash
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

## Check system logs

```bash
sudo journalctl -u amazon-cloudwatch-agent -f
```

---

# Common Errors

## Error 1 — Missing `common-config.toml`

### Error

```
failed to open common-config.toml
```

### Cause

Broken installation.

### Fix

```bash
sudo dpkg --purge amazon-cloudwatch-agent
sudo rm -rf /opt/aws/amazon-cloudwatch-agent
sudo dpkg -i cwagent.deb
```

---

## Error 2 — Missing collectd dependency

### Error

```
open /usr/share/collectd/types.db: no such file
```

### Cause

collectd not installed.

### Fix

```bash
sudo apt install collectd -y
```

---

## Error 3 — Logs not appearing

Possible causes:

* IAM role missing
* Region mismatch
* Agent not running

Check:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

---

## Error 4 — Metrics not appearing

Wait **2–3 minutes**.

CloudWatch metrics are **not instant**.

---

# Final Result

After completing this SOP:

* CloudWatch Agent running
* System logs sent to CloudWatch
* Metrics visible in `CWAgent` namespace
* Alarm configured
* SNS notifications working

---


If you want, I can also show you **how DevOps engineers structure GitHub SOP repositories professionally** (with diagrams, architecture, and automation scripts). That will make this **look like a real production DevOps repo**, not just notes.
