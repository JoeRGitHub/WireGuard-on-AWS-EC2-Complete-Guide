**backup/restore script** will make it easy to migrate your WireGuard server to another EC2 instance or recover after failure.

Here’s how we’ll do it:

---

# 🛡 WireGuard Backup & Restore

We want to back up:

* **Server keys** (`server_private.key`, `server_public.key`)
* **Main config** (`/etc/wireguard/wg0.conf`)
* **Customer configs** (`/etc/wireguard/customers/`)

---

## **1️⃣ Backup Script**

Save as `/etc/wireguard/backup_wg.sh`:

```bash
#!/bin/bash
# Backup WireGuard configs and customer files

BACKUP_DIR="/root/wg_backup"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
DEST="$BACKUP_DIR/wg_backup_$TIMESTAMP.tar.gz"

mkdir -p $BACKUP_DIR

# Create tarball
tar -czf $DEST \
  /etc/wireguard/wg0.conf \
  /etc/wireguard/server_private.key \
  /etc/wireguard/server_public.key \
  /etc/wireguard/customers

echo "✅ Backup created: $DEST"
```

Make it executable:

```bash
sudo chmod +x /etc/wireguard/backup_wg.sh
```

Run manually:

```bash
sudo /etc/wireguard/backup_wg.sh
```

Set up a **cron job** (daily backup at 2 AM):

```bash
sudo crontab -e
```

Add:

```
0 2 * * * /etc/wireguard/backup_wg.sh
```

---

## **2️⃣ Restore Script**

Save as `/etc/wireguard/restore_wg.sh`:

```bash
#!/bin/bash
# Restore WireGuard configs from backup

if [ $# -lt 1 ]; then
  echo "Usage: $0 <backup_file.tar.gz>"
  exit 1
fi

BACKUP_FILE=$1

if [ ! -f "$BACKUP_FILE" ]; then
  echo "❌ Backup file not found: $BACKUP_FILE"
  exit 1
fi

# Stop WireGuard
sudo systemctl stop wg-quick@wg0

# Extract backup
sudo tar -xzf $BACKUP_FILE -C /

# Restart WireGuard
sudo systemctl start wg-quick@wg0

echo "✅ Restore completed from $BACKUP_FILE"
```

Make it executable:

```bash
sudo chmod +x /etc/wireguard/restore_wg.sh
```

---

## **3️⃣ Migration to New EC2**

1. Launch new EC2 Ubuntu instance.
2. Install WireGuard + UFW (steps 2 from guide).
3. Copy backup file (`scp` or S3). Example:

   ```bash
   scp -i your-key.pem wg_backup_xxx.tar.gz ubuntu@<new-ec2-ip>:/tmp/
   ```
4. On new EC2, run restore:

   ```bash
   sudo /etc/wireguard/restore_wg.sh /tmp/wg_backup_xxx.tar.gz
   ```
5. Start service:

   ```bash
   sudo systemctl enable wg-quick@wg0
   sudo systemctl restart wg-quick@wg0
   ```

---

## ✅ Features

* Full backup of server + customer configs.
* Timestamped archives (`wg_backup_YYYYMMDD_HHMMSS.tar.gz`).
* Easy restore to same or new EC2.
* Can automate with cron for daily snapshots.

---

