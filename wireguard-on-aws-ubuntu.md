**full guide from scratch** for deploying a **WireGuard VPN server on AWS EC2 Ubuntu**, including scripts to add customers and cameras, with isolation and automation.

---

# 📘 WireGuard on AWS EC2 – Complete Guide

---

## **1️⃣ Launch AWS EC2 Instance**

* AMI: **Ubuntu 22.04 LTS**
* Type: `t3.micro` (or larger if many clients)
* Storage: 10–20 GB
* Assign **Elastic IP** (static address for VPN)
* **Security Group** rules:

  * `22/tcp` → allow from **your IP only** (SSH)
  * `51820/udp` → allow from all (`0.0.0.0/0`) or client IP ranges

---

## **2️⃣ Initial Server Setup**

SSH into your EC2:

```bash
ssh -i your-key.pem ubuntu@<elastic-ip>
```

Update + install packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wireguard ufw -y
```

Enable firewall:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 51820/udp
sudo ufw enable
```

---

## **3️⃣ Configure WireGuard Server**

Generate server keys:

```bash
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

Create a directory for scripts + configs:

```bash
mkdir -p wireguard
```

Create server config `vi /wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = <SERVER_PRIVATE_KEY>
Address = 10.0.0.1/24
ListenPort = 51820
SaveConfig = true
```

Enable WireGuard:

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

---

## **4️⃣ Automation Directory**

Create a directory for scripts + configs:

```bash
mkdir -p wireguard/customers
cd wireguard
```

---

## **5️⃣ Script: Add New Customer**

Save as `/etc/wireguard/add_customer.sh`:

```bash
#!/bin/bash
# Add new customer with NVR + X cameras

if [ $# -lt 2 ]; then
  echo "Usage: $0 <CustomerName> <CameraCount>"
  exit 1
fi

CUSTOMER=$1
CAM_COUNT=$2
BASE_DIR="/etc/wireguard/customers/$CUSTOMER"
mkdir -p $BASE_DIR

# Find next available subnet
LAST_SUBNET=$(ls /etc/wireguard/customers | wc -l)
SUBNET=$((LAST_SUBNET + 2))   # Start at 10.0.2.0/24
NET="10.0.$SUBNET"

echo "Creating customer $CUSTOMER on $NET.0/24"

# Create NVR
wg genkey | tee $BASE_DIR/nvr.key | wg pubkey > $BASE_DIR/nvr.pub
NVR_PRIV=$(cat $BASE_DIR/nvr.key)
NVR_PUB=$(cat $BASE_DIR/nvr.pub)

NVR_IP="$NET.1"
cat > $BASE_DIR/${CUSTOMER}_NVR.conf <<EOF
[Interface]
PrivateKey = $NVR_PRIV
Address = $NVR_IP/32
DNS = 1.1.1.1

[Peer]
PublicKey = $(cat /etc/wireguard/server_public.key)
Endpoint = <SERVER_PUBLIC_IP>:51820
AllowedIPs = $NET.0/24
PersistentKeepalive = 25
EOF

# Add NVR peer to server
sudo tee -a /etc/wireguard/wg0.conf > /dev/null <<EOF

[Peer]
PublicKey = $NVR_PUB
AllowedIPs = $NVR_IP/32
EOF

# Create cameras
for i in $(seq 1 $CAM_COUNT); do
  wg genkey | tee $BASE_DIR/cam${i}.key | wg pubkey > $BASE_DIR/cam${i}.pub
  CAM_PRIV=$(cat $BASE_DIR/cam${i}.key)
  CAM_PUB=$(cat $BASE_DIR/cam${i}.pub)
  CAM_IP="$NET.$((i+1))"

  cat > $BASE_DIR/${CUSTOMER}_CAM${i}.conf <<EOF
[Interface]
PrivateKey = $CAM_PRIV
Address = $CAM_IP/32
DNS = 1.1.1.1

[Peer]
PublicKey = $(cat /etc/wireguard/server_public.key)
Endpoint = <SERVER_PUBLIC_IP>:51820
AllowedIPs = $NET.0/24
PersistentKeepalive = 25
EOF

  # Add camera peer to server
  sudo tee -a /etc/wireguard/wg0.conf > /dev/null <<EOF

[Peer]
PublicKey = $CAM_PUB
AllowedIPs = $CAM_IP/32
EOF

done

# Restart WireGuard
sudo systemctl restart wg-quick@wg0

# Summary
echo "Customer $CUSTOMER created with subnet $NET.0/24"
echo "| Device | IP        | Config File |"
echo "|--------|-----------|-------------|"
echo "| NVR    | $NVR_IP   | $BASE_DIR/${CUSTOMER}_NVR.conf |"
for i in $(seq 1 $CAM_COUNT); do
  echo "| CAM$i  | $NET.$((i+1)) | $BASE_DIR/${CUSTOMER}_CAM${i}.conf |"
done
```

Make executable:

```bash
sudo chmod +x /etc/wireguard/add_customer.sh
```

---

## **6️⃣ Script: Add Cameras to Existing Customer**

Save as `/etc/wireguard/add_cameras.sh`:

```bash
#!/bin/bash
# Add cameras to an existing customer

if [ $# -lt 2 ]; then
  echo "Usage: $0 <CustomerName> <AdditionalCameraCount>"
  exit 1
fi

CUSTOMER=$1
ADD_COUNT=$2
BASE_DIR="/etc/wireguard/customers/$CUSTOMER"

if [ ! -d "$BASE_DIR" ]; then
  echo "Customer not found!"
  exit 1
fi

NET=$(grep "Address" $BASE_DIR/${CUSTOMER}_NVR.conf | cut -d' ' -f3 | cut -d. -f1-3)
LAST_CAM=$(ls $BASE_DIR | grep CAM | wc -l)

for i in $(seq 1 $ADD_COUNT); do
  NEW_INDEX=$((LAST_CAM + i))
  wg genkey | tee $BASE_DIR/cam${NEW_INDEX}.key | wg pubkey > $BASE_DIR/cam${NEW_INDEX}.pub
  CAM_PRIV=$(cat $BASE_DIR/cam${NEW_INDEX}.key)
  CAM_PUB=$(cat $BASE_DIR/cam${NEW_INDEX}.pub)
  CAM_IP="$NET.$((NEW_INDEX+1))"

  cat > $BASE_DIR/${CUSTOMER}_CAM${NEW_INDEX}.conf <<EOF
[Interface]
PrivateKey = $CAM_PRIV
Address = $CAM_IP/32
DNS = 1.1.1.1

[Peer]
PublicKey = $(cat /etc/wireguard/server_public.key)
Endpoint = <SERVER_PUBLIC_IP>:51820
AllowedIPs = $NET.0/24
PersistentKeepalive = 25
EOF

  # Add to server
  sudo tee -a /etc/wireguard/wg0.conf > /dev/null <<EOF

[Peer]
PublicKey = $CAM_PUB
AllowedIPs = $CAM_IP/32
EOF

  echo "Added Camera $NEW_INDEX -> $CAM_IP"
done

# Restart WG
sudo systemctl restart wg-quick@wg0
```

Make executable:

```bash
sudo chmod +x /etc/wireguard/add_cameras.sh
```

---

## **7️⃣ Usage Examples**

* Add customer with NVR + 4 cameras:

```bash
sudo /etc/wireguard/add_customer.sh "CustomerA" 4
```

* Add 10 more cameras later:

```bash
sudo /etc/wireguard/add_cameras.sh "CustomerA" 10
```

---

## **8️⃣ Verification**

Check peers:

```bash
sudo wg show
```

Test ping from client:

```bash
ping 10.0.2.1   # NVR
ping 10.0.2.2   # Camera
```

---

# ✅ Features of This Setup

* Each customer has its **own subnet** (isolated).
* Automatically generates **NVR + camera configs**.
* Supports **expansion** (add cameras anytime).
* Outputs **Markdown table** for documentation.
* Uses **Cloudflare DNS (1.1.1.1)** for clients.
* Secure firewall setup with **UFW + AWS SG**.

---
