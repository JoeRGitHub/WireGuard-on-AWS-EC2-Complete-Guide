 **backup-to-S3 script** so it automatically **keeps only the last X backups** (for example: the last **7 daily backups**) and deletes older ones.

---

# ☁️ WireGuard Backup to S3 with Retention

Save as `/etc/wireguard/backup_to_s3.sh`:

```bash
#!/bin/bash
# Backup WireGuard configs and upload to S3 with retention policy

BACKUP_DIR="/root/wg_backup"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="wg_backup_$TIMESTAMP.tar.gz"
DEST="$BACKUP_DIR/$BACKUP_FILE"

S3_BUCKET="s3://wg-backups-mycompany"
RETENTION=7   # keep last 7 backups

mkdir -p $BACKUP_DIR

echo "📦 Creating backup..."
tar -czf $DEST \
  /etc/wireguard/wg0.conf \
  /etc/wireguard/server_private.key \
  /etc/wireguard/server_public.key \
  /etc/wireguard/customers

echo "☁️ Uploading $BACKUP_FILE to $S3_BUCKET..."
aws s3 cp $DEST $S3_BUCKET/

if [ $? -eq 0 ]; then
  echo "✅ Backup uploaded: $S3_BUCKET/$BACKUP_FILE"
else
  echo "❌ Failed to upload backup to S3"
  exit 1
fi

echo "🧹 Applying retention policy (keep last $RETENTION backups)..."
# List backups sorted by date, skip the last $RETENTION, and delete older ones
OLD_BACKUPS=$(aws s3 ls $S3_BUCKET/ | awk '{print $4}' | sort | head -n -$RETENTION)

for FILE in $OLD_BACKUPS; do
  echo "🗑 Deleting old backup: $FILE"
  aws s3 rm $S3_BUCKET/$FILE
done

echo "✅ Backup + retention completed."
```

---

## **Usage**

* Run manually:

  ```bash
  sudo /etc/wireguard/backup_to_s3.sh
  ```
* Automate with cron (daily at 3 AM):

  ```bash
  sudo crontab -e
  ```

  Add:

  ```
  0 3 * * * /etc/wireguard/backup_to_s3.sh
  ```

---

## **How Retention Works**

1. Script uploads a new backup to S3.
2. Lists all backups in the bucket.
3. Keeps the **last N (7 by default)**.
4. Deletes all older ones.

---

## ✅ Features

* Keeps **backups safe in S3**.
* **Retention policy**: auto-cleanup of old backups.
* Keeps storage costs low.
* Restores work the same way (download + restore script).

---
