# Automated Backup and Rotation Script with Google Drive Integration

## Project Overview

This project implements a fully automated backup management solution using Bash scripting on AWS EC2. The solution creates timestamped ZIP backups of a project directory, stores them in a structured backup hierarchy, uploads them to Google Drive using Rclone, sends webhook notifications on successful execution, maintains backup logs, and automatically removes old backups based on a retention policy.

The project demonstrates practical DevOps automation concepts including:

* Backup Automation
* Shell Scripting
* Google Drive Integration
* Configuration Management
* Log Management
* Retention Policies
* Cron Job Scheduling
* Cloud Infrastructure Setup
* Webhook Notifications

---

# Architecture

```text
EC2 Instance
│
├── Source Project
│
├── backup.sh
│
├── config.env
│
├── Local ZIP Backup
│
├── Retention Cleanup
│
├── Google Drive Upload (rclone)
│
├── Logging
│
├── Webhook Notification
│
└── Cron Scheduler
        │
        ▼
Google Drive
```

![](./img/Architecture%20Dig.png)

---

# Features

* Automated ZIP backup creation
* Timestamp-based backup naming
* Structured backup storage
* Google Drive integration using Rclone
* Webhook notification support
* Daily backup retention
* Weekly retention configuration
* Monthly retention configuration
* Detailed logging
* Cron-based scheduling
* Configurable settings using environment variables

---

# Phase 1: Infrastructure Setup

## Create EC2 Instance

| Configuration  | Value             |
| -------------- | ----------------- |
| Name           | Backup-Automation |
| OS             | Ubuntu 24.04 LTS  |
| Instance Type  | t2.micro          |
| Storage        | 20 GB gp3         |
| Security Group | SSH Only          |
| Public IP      | Enabled           |

### Security Group Rules

| Type | Port | Source |
| ---- | ---- | ------ |
| SSH  | 22   | My IP  |

![](./img/Screenshot%202026-05-31%20090339.png)

---

# Phase 2: Server Preparation

## Connect to EC2

```bash
ssh -i key.pem ubuntu@PUBLIC_IP
```
![](./img/Screenshot%202026-05-31%20090711.png)

## Update Server

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

Reconnect after reboot.

## Install Required Packages

```bash
sudo apt install \
zip \
unzip \
curl \
git \
jq \
tree \
cron \
logrotate \
-y
```
![](./img/Screenshot%202026-05-31%20090853.png)

## Verify Installation

```bash
zip -v
curl --version
git --version
```
![](./img/Screenshot%202026-05-31%20091107.png)
---

# Phase 3: Project Structure

## Create Workspace

```bash
mkdir -p ~/backup-project
cd ~/backup-project
```

## Create Project Directories

```bash
mkdir backups
mkdir logs
mkdir config
mkdir scripts
mkdir test-project
```

## Verify Structure

```bash
tree
```

Expected:

```text
backup-project
├── backups
├── config
├── logs
├── scripts
└── test-project
```

![](./img/Screenshot%202026-05-31%20091335.png)

---

# Phase 4: Create Sample Project

Move into sample project directory:

```bash
cd ~/backup-project/test-project
```

Create sample files:

```bash
echo "index" > index.html

echo "config" > config.txt

mkdir src

echo "console.log('app')" > src/app.js
```

Verify:

```bash
tree
```

![](./img/Screenshot%202026-05-31%20091659.png)

---

# Phase 5: Google Drive Integration

## Install Rclone

```bash
curl https://rclone.org/install.sh | sudo bash
```
![](./img/Screenshot%202026-05-31%20091859.png)
Verify installation:

```bash
rclone version
```
![](./img/Screenshot%202026-05-31%20091921.png)
---

## Configure Google Drive

Start configuration:

```bash
rclone config
```

![](./img/Screenshot%202026-05-31%20092530.png)

### Create Remote

Choose:

```text
n
```

### Remote Name

```text
gdrive
```

### Storage Type

Choose:

```text
drive
```

### Client ID

Press Enter.

### Client Secret

Press Enter.

### Scope

Choose:

```text
1
```

### Root Folder ID

Press Enter.

### Service Account File

Press Enter.

### Advanced Config

Choose:

```text
n
```

### Auto Config

If using local system:

```text
y
```

If using AWS EC2:

```text
n
```

Authorize Google Account through browser.

### Shared Drive

Choose:

```text
n
```

### Save Configuration

Choose:

```text
y
```

### Exit

Choose:

```text
q
```

---

## Verify Remote

```bash
rclone listremotes
```

Expected:

```text
gdrive:
```
![](./img/Screenshot%202026-05-31%20101502.png)
![](./img/Screenshot%202026-05-31%20101645.png)
## Verify Google Drive Access

```bash
rclone lsd gdrive:
```
![](./img/Screenshot%202026-05-31%20102007.png)

---

## Create Backup Folder

Create folder in Google Drive:

```text
ProjectBackups
```

Verify:

```bash
rclone lsd gdrive:
```

---

## Upload Test File

```bash
echo "hello" > test.txt

rclone copy test.txt gdrive:ProjectBackups
```

Verify:

```bash
rclone ls gdrive:ProjectBackups
```
![](./img/Screenshot%202026-05-31%20102251.png)
---

## Final Validation

```bash
rclone about gdrive:
```

This confirms:

* Authentication works
* Google Drive is reachable
* Upload permissions are available

---

# Phase 6: Configuration Management

Create configuration file:

```bash
nano ~/backup-project/config/config.env
```

Add:

```bash
# Project Information
PROJECT_NAME=BackupAutomation

# Source Project Directory
SOURCE_DIR=/home/ubuntu/backup-project/test-project

# Backup Storage Location
BACKUP_ROOT=/home/ubuntu/backup-project/backups

# Log File
LOG_FILE=/home/ubuntu/backup-project/logs/backup.log

# Google Drive
GDRIVE_REMOTE=gdrive
GDRIVE_FOLDER=ProjectBackups

# Retention Policy
DAILY_KEEP=7
WEEKLY_KEEP=4
MONTHLY_KEEP=3

# Notifications
ENABLE_NOTIFY=true

WEBHOOK_URL=https://webhook.site/YOUR_UNIQUE_ID
```
![](./img/Screenshot%202026-05-31%20113923.png)

## Verify Variables

```bash
source ~/backup-project/config/config.env

echo $PROJECT_NAME
echo $SOURCE_DIR
echo $BACKUP_ROOT
echo $LOG_FILE
echo $GDRIVE_REMOTE
echo $GDRIVE_FOLDER
```
![](./img/Screenshot%202026-05-31%20115000.png)

---

## Create Log File

```bash
touch ~/backup-project/logs/backup.log

chmod 644 ~/backup-project/logs/backup.log
```

---

# Phase 7: Backup Script Creation

Create script:

```bash
nano ~/backup-project/scripts/backup.sh
```

Make executable:

```bash
chmod +x ~/backup-project/scripts/backup.sh
```

Verify:

```bash
ls -l ~/backup-project/scripts/backup.sh
```
![](./img/Screenshot%202026-05-31%20115234.png)
![](./img/Screenshot%202026-05-31%20115255.png)
---

# Phase 8: Timestamp Generation

Generate timestamps using:

```bash
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
```

Format:

```text
YYYYMMDD_HHMMSS
```

Example:

```text
20260531_121530
```
![](./img/Screenshot%202026-05-31%20115336.png)
![](./img/Screenshot%202026-05-31%20115357.png)

---

# Phase 9: Backup Folder Structure

Required structure:

```text
~/backups/ProjectName/YYYY/MM/DD/
```

Generated path example:

```text
/home/ubuntu/backup-project/backups/BackupAutomation/2026/05/31
```
![](./img/Screenshot%202026-05-31%20115438.png)
![](./img/Screenshot%202026-05-31%20115509.png)

---

# Phase 10: ZIP Backup Creation

Backup naming format:

```text
ProjectName_YYYYMMDD_HHMMSS.zip
```

Example:

```text
BackupAutomation_20260531_121530.zip
```

Verify:

```bash
find ~/backup-project/backups -name "*.zip"
```
![](./img/Screenshot%202026-05-31%20115709.png)
![](./img/Screenshot%202026-05-31%20115723.png)
![](./img/Screenshot%202026-05-31%20120301.png)

---

# Phase 11: Google Drive Upload Automation

Upload backup using:

```bash
rclone copy \
"$BACKUP_FILE" \
"$GDRIVE_REMOTE:$GDRIVE_FOLDER"
```
![](./img/Screenshot%202026-05-31%20120447.png)

Validate upload:

```bash
rclone ls gdrive:ProjectBackups
```

Expected:

```text
BackupAutomation_20260531_121530.zip
```
![](./img/Screenshot%202026-05-31%20120500.png)
![](/img/Screenshot%202026-05-31%20171238.png)

---

# Phase 12: Logging System

Log file:

```text
logs/backup.log
```

Sample output:

```text
2026-05-31 13:15:22 : Backup Started
2026-05-31 13:15:24 : Backup Created : BackupAutomation_20260531_131522.zip
2026-05-31 13:15:30 : Upload Successful
```

View logs:

```bash
cat ~/backup-project/logs/backup.log
```
![](./img/Screenshot%202026-05-31%20120748.png)

---

# Phase 13: Webhook Notifications

Create webhook using:

```text
https://webhook.site
```

Update:

```bash
WEBHOOK_URL=https://webhook.site/YOUR_UNIQUE_ID
```
![](./img/Screenshot%202026-05-31%20121016.png)

After successful backup, the script sends:

```json
{
  "project":"BackupAutomation",
  "date":"BackupDate",
  "status":"BackupSuccessful"
}
```

Verify request on Webhook.site dashboard.

![](/img/Screenshot%202026-05-31%20121522.png)

---

# Phase 15: Rotational Backup Strategy

Retention Configuration:

```bash
DAILY_KEEP=7
WEEKLY_KEEP=4
MONTHLY_KEEP=3
```

Current implementation actively enforces:

```text
Keep Last 7 Daily Backups
```

Older backups are automatically removed.

### Verify Backup Count

```bash
find ~/backup-project/backups -name "*.zip" | wc -l
```

Expected:

```text
7
```

### Verify Deletion Logs

```bash
grep "Deleted Old Backup" ~/backup-project/logs/backup.log
```

Example:

```text
Deleted Old Backup : BackupAutomation_20260531_062646.zip
Deleted Old Backup : BackupAutomation_20260531_063408.zip
```

### Weekly and Monthly Retention

The project includes configurable retention values:

```bash
WEEKLY_KEEP=4
MONTHLY_KEEP=3
```

These settings allow future extension for weekly and monthly backup classification based on backup creation date.

---

# Phase 16: Cron Scheduling

Open cron:

```bash
crontab -e
```
![](/img/Screenshot%202026-05-31%20123440.png)

Add:

```bash
0 2 * * * /home/ubuntu/backup-project/scripts/backup.sh >> /home/ubuntu/backup-project/logs/cron.log 2>&1
```

Meaning:

* Runs daily
* Executes at 2:00 AM
* Stores execution output in cron.log

Verify:

```bash
crontab -l
```

---

# Phase 17: Cron Testing

Temporary schedule:

```bash
*/2 * * * *
```
![](/img/Screenshot%202026-05-31%20123540.png)

Runs every 2 minutes.

Check output:

```bash
cat ~/backup-project/logs/cron.log
```
![](/img/Screenshot%202026-05-31%20123932.png)
![](/img/Screenshot%202026-05-31%20124006.png)

After verification, revert to:

```bash
0 2 * * *
```

---

# Phase 18: Weekly and Monthly Retention Configuration

Verify configuration:

```bash
WEEKLY_KEEP=4
MONTHLY_KEEP=3
```

Current project keeps these values configurable for future retention expansion.

---

# Phase 19: Log Rotation

When log size exceeds 1 MB:

```text
backup.log
```

becomes:

```text
backup.log.20260531_120000
```

and a new:

```text
backup.log
```

is automatically created.

This prevents unlimited log growth.

---

# Phase 20: Final Validation

## Run Backup

```bash
~/backup-project/scripts/backup.sh
```
![](/img/Screenshot%202026-05-31%20124549.png)
![](/img/Screenshot%202026-05-31%20124843.png)

## Verify Local Backups

```bash
find ~/backup-project/backups -name "*.zip"
```

## Verify Backup Count

```bash
find ~/backup-project/backups -name "*.zip" | wc -l
```

Expected:

```text
7
```

## Verify Google Drive Uploads

```bash
rclone ls gdrive:ProjectBackups
```

## Verify Logs

```bash
tail -20 ~/backup-project/logs/backup.log
```

Expected:

```text
Backup Started
Backup Created
Upload Successful
Webhook Sent
Starting Retention Cleanup
Backup Completed
```
![](/img/Screenshot%202026-05-31%20124918.png)
![](/img/Screenshot%202026-05-31%20130803.png)
## Verify Cron

```bash
crontab -l
```

Expected:

```bash
0 2 * * * /home/ubuntu/backup-project/scripts/backup.sh >> /home/ubuntu/backup-project/logs/cron.log 2>&1
```

---

# Security Considerations

* Google Drive access is authenticated using OAuth.
* Backup files remain archived in Google Drive even after local cleanup.
* Configuration values are centralized in config.env.
* Logs are rotated to prevent disk space exhaustion.
* Backup retention protects against uncontrolled storage growth.
* Cron automation ensures backups occur without manual intervention.

---

# Sample Usage

Run with notifications:

```bash
~/backup-project/scripts/backup.sh
```

Run without webhook notification:

```bash
~/backup-project/scripts/backup.sh --no-notify
```

---

# Project Outcome

The project successfully automates backup creation, cloud upload, retention cleanup, logging, webhook notifications, and scheduled execution using AWS EC2, Bash scripting, Rclone, and Google Drive integration.
