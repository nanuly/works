# How to Set Up Incremental Backups with rsync and cron on Linux

**Posted on October 1st, 2025**

Incremental backups only copy files that changed since the last backup. This saves time and storage space. You do not need to copy everything again and again.

## Why use incremental backups:

- Save storage space by copying only changed files
- Backup faster than full backups
- Use less internet bandwidth
- Run automatically without your help
- Keep your files safe

## What You Need Before Starting

Make sure you have these things ready:

- Linux computer with admin access
- rsync program installed
- Basic knowledge of command line
- Place to store backups

Check if rsync is installed. Type this command:

```bash
rsync --version
```

If not installed, install it:

```bash
# For Ubuntu
sudo apt install rsync
# For CentOS
sudo yum install rsync
```

## Basic rsync Commands

rsync is a program that copies files between folders. Here are important options:

- `-a` means archive mode, keeps file details
- `-v` means verbose, shows what is happening
- `-z` means compress files while copying
- `-h` means show file sizes in easy format
- `--delete` removes old files from backup
- `--exclude` skips certain files

The basic way to use rsync:

```bash
rsync [options] source destination
```

## Step 1: Create Your Backup Script

First make folders for your backup:

```bash
sudo mkdir -p /backup/scripts
sudo mkdir -p /backup/data
```

Create a new backup script:

```bash
sudo nano /backup/scripts/backup.sh
```

Put this code in the file:

```bash
#!/bin/bash
SOURCE_DIR="/home"
BACKUP_DIR="/backup/data"
LOG_FILE="/var/log/backup.log"
DATE=$(date '+%Y-%m-%d_%H-%M-%S')
echo "Starting backup at $DATE" >> $LOG_FILE
rsync -avzh \
    --delete \
    --exclude='*.tmp' \
    --exclude='*.cache' \
    --log-file=$LOG_FILE \
    $SOURCE_DIR/ $BACKUP_DIR/
echo "Backup finished at $(date '+%Y-%m-%d_%H-%M-%S')" >> $LOG_FILE
```

Make the script runnable:

```bash
sudo chmod +x /backup/scripts/backup.sh
```

## Step 2: Test Your Backup

Run the backup script to test it:

```bash
sudo /backup/scripts/backup.sh
```

Check if it worked by looking at the log:

```bash
sudo tail -10 /var/log/backup.log
```

See if files were copied:

```bash
ls -la /backup/data/
```

## Step 3: Make Backups Automatic with cron

cron runs programs automatically at set times. Edit cron settings:

```bash
sudo crontab -e
```

Add one of these lines for different backup times:

```bash
# For daily backup at 2 AM
0 2 * * * /backup/scripts/backup.sh
# For weekly backup on Sunday at 3 AM
0 3 * * 0 /backup/scripts/backup.sh
# For backup every hour from 9 AM to 5 PM
0 9-17 * * * /backup/scripts/backup.sh
```

## Understanding cron Time Format

cron uses this format: minute hour day month weekday command

- `*` means any time
- Numbers mean exact times
- `-` means range like 9-17
- `,` means list like 1,3,5

Examples:

- `0 2 * * *` means every day at 2 AM
- `30 14 * * 1-5` means weekdays at 2:30 PM
- `0 */4 * * *` means every 4 hours

## Step 4: Advanced Backup Options

### Backup to Remote Server

To backup files to another computer over the network:

```bash
#!/bin/bash
SOURCE_DIR="/home"
REMOTE_USER="backup"
REMOTE_HOST="backup-server.com"
REMOTE_DIR="/backup/data"
SSH_KEY="/root/.ssh/backup_key"
rsync -avzh \
    --delete \
    -e "ssh -i $SSH_KEY" \
    $SOURCE_DIR/ $REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/
```

### Keep Multiple Backup Copies

This keeps several backup versions:

```bash
#!/bin/bash
SOURCE_DIR="/home"
BACKUP_BASE="/backup"
DATE=$(date '+%Y-%m-%d')
CURRENT_BACKUP="$BACKUP_BASE/current"
DAILY_BACKUP="$BACKUP_BASE/daily-$DATE"
cp -al $CURRENT_BACKUP $DAILY_BACKUP 2>/dev/null || mkdir -p $DAILY_BACKUP
rsync -avzh --delete $SOURCE_DIR/ $CURRENT_BACKUP/
# Remove old backups (keep 7 days)
find $BACKUP_BASE -name "daily-*" -type d -mtime +7 -exec rm -rf {} \;
```

### Skip Certain Files

Create a file to list what to skip:

```bash
sudo nano /backup/scripts/skip_files.txt
```

Add these patterns:

```
*.tmp
*.cache
*.log
.Trash*
Downloads/
```

Use it in your backup:

```bash
rsync -avzh \
    --delete \
    --exclude-from='/backup/scripts/skip_files.txt' \
    $SOURCE_DIR/ $BACKUP_DIR/
```

## Step 5: Monitor Your Backups

### Check Backup Status

Make a script to check backup status:

```bash
#!/bin/bash
LOG_FILE="/var/log/backup.log"
BACKUP_DIR="/backup/data"
echo "=== Backup Status Report ==="
echo "Recent backup activity:"
tail -10 $LOG_FILE
echo "Backup folder size:"
du -sh $BACKUP_DIR
echo "Last backup time:"
stat -c %y $BACKUP_DIR | head -1
```

### Get Email Alerts

Install email program:

```bash
sudo apt install mailutils
```

Add email to backup script:

```bash
if [ $? -eq 0 ]; then
    echo "Backup worked fine" | mail -s "Backup Success" admin@example.com
else
    echo "Backup failed" | mail -s "Backup Failed" admin@example.com
fi
```

### Check Disk Space

Add disk space check:

```bash
AVAILABLE_SPACE=$(df -BG $BACKUP_DIR | awk 'NR==2 {print $4}' | sed 's/G//')
if [ $AVAILABLE_SPACE -lt 10 ]; then
    echo "Warning: Low disk space" | mail -s "Low Space" admin@example.com
    exit 1
fi
```

## Fixing Common Problems

### Permission Problems

Fix file ownership:

```bash
sudo chown -R root:root /backup/scripts/
sudo chmod +x /backup/scripts/*.sh
```

### cron Not Working

Check if cron is running:

```bash
sudo systemctl status cron
sudo systemctl start cron
```

See cron activity:

```bash
sudo tail -f /var/log/cron
```

### Big File Problems

For large files add these options:

```bash
rsync -avzh \
    --partial \
    --progress \
    --timeout=300 \
    $SOURCE_DIR/ $BACKUP_DIR/
```

### Network Problems

Add retry for network backups:

```bash
#!/bin/bash
MAX_RETRIES=3
RETRY_COUNT=0
while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
    if rsync -avzh --delete $SOURCE_DIR/ $REMOTE_DESTINATION/; then
        echo "Backup worked"
        break
    else
        RETRY_COUNT=$((RETRY_COUNT + 1))
        echo "Backup failed, trying again"
        sleep 60
    fi
done
```

## Best Practices

- Test restore regularly to make sure backups work
- Keep backups in different places
- Check backup logs for errors
- Update skip file patterns
- Write down important details
- Set up failure alerts
- Plan for more storage space

## Security Tips

### Use SSH Keys for Remote Backups

Make special key for backups:

```bash
ssh-keygen -t rsa -b 4096 -f /root/.ssh/backup_key
```

Copy key to remote server:

```bash
ssh-copy-id -i /root/.ssh/backup_key.pub backup@backup-server.com
```

### Encrypt Important Backups

For sensitive files, encrypt backups:

```bash
tar -czf - $SOURCE_DIR | gpg --symmetric --output backup-$(date +%Y%m%d).tar.gz.gpg
```

## Conclusion

Setting up automatic backups with rsync and cron gives you a reliable way to protect your files. This method saves space, works faster, and runs by itself.

Remember these key points:

- Start simple with local backups first
- Test your backups regularly
- Watch backup logs and disk space
- Skip files you do not need
- Use proper security for network backups

Regular backups keep your data safe. With this guide, you can make a professional backup system that protects your files automatically.

Always test that you can restore files from your backups. This makes sure everything works when you really need it.
