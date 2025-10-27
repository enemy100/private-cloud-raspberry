# 🌐 Private Cloud Solution - Samba + FileBrowser

## 📋 Summary

**RAID 1 Disk** (`/dev/sdb`) → Important files (redundancy)  
**Normal Disk** (`/dev/sdd`) → Less important files

## 🎯 How It Works

### **Local Access (Windows)**
```
Windows Explorer
  → \\raspberry-ip\arquivos
  → \\raspberry-ip\temp
  → Drag and drop directly!
```

### **Remote Access (Outside)**
```
Web Browser
  → https://cloud.yourdomain.com
  → Upload/Download files
```

## 🔧 Implementation

### **Step 1: Prepare the Disks**

```bash
# 1. Check disks
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,UUID

# 2. Create partitions (IF NEEDED)
# RAID 1 disk (/dev/sdb)
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 0% 100%

# Normal disk (/dev/sdd)
sudo parted /dev/sdd mklabel gpt
sudo parted /dev/sdd mkpart primary ext4 0% 100%

# 3. Format
sudo mkfs.ext4 /dev/sdb1 -L cloud_main
sudo mkfs.ext4 /dev/sdd1 -L cloud_temp

# 4. Create mount directories
sudo mkdir -p /mnt/cloud_main    # RAID 1 (important)
sudo mkdir -p /mnt/cloud_temp     # Normal disk

# 5. Mount temporarily
sudo mount /dev/sdb1 /mnt/cloud_main
sudo mount /dev/sdd1 /mnt/cloud_temp

# 6. Add to fstab for automatic mounting
sudo bash -c 'cat >> /etc/fstab << EOF
/dev/sdb1  /mnt/cloud_main  ext4  defaults,noatime  0  2
/dev/sdd1  /mnt/cloud_temp  ext4  defaults,noatime  0  2
EOF'

# 7. Create shared folders
sudo mkdir -p /mnt/cloud_main/{arquivos,compartilhado}
sudo mkdir -p /mnt/cloud_temp/{temporarios,downloads}

# 8. Configure permissions IMPORTANT!
# FileBrowser runs as UID 1000, so disks need permissions for UID 1000
sudo chown -R robson:robson /mnt/cloud_main
sudo chown -R robson:robson /mnt/cloud_temp

# 9. Give write permissions
sudo chmod -R 775 /mnt/cloud_main
sudo chmod -R 775 /mnt/cloud_temp
```

### **Step 2: Add to .env**

Edit `~/Downloads/.env` and add:

```env
# Cloud domains
DOMINIO_CLOUD=cloud.yourdomain.com

# Samba credentials
SAMBA_USER=robson
SAMBA_PASSWORD=YourStrongPassword123
SAMBA_GUEST=no
```

### **Step 3: Add to docker-compose.yml**

Add these two new services:

```yaml
  # Samba - Local Network Sharing
  samba:
    image: dperson/samba:latest
    container_name: samba
    restart: unless-stopped
    networks:
      - network_public
    volumes:
      - /mnt/cloud_main:/mnt/cloud_main:rw
      - /mnt/cloud_temp:/mnt/cloud_temp:rw
    environment:
      - TZ=America/Sao_Paulo
      - USER=${SAMBA_USER};${SAMBA_PASSWORD}
      - SHARE=arquivos;/mnt/cloud_main/arquivos;yes;no;no;all;${SAMBA_USER};${SAMBA_USER}
      - SHARE=temp;/mnt/cloud_temp/temporarios;yes;no;no;all;${SAMBA_USER};${SAMBA_USER}
    labels:
      - "traefik.enable=true"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    ports:
      - "445:445"
      - "139:139"

  # FileBrowser - Web Remote Access
  filebrowser:
    image: filebrowser/filebrowser:latest
    container_name: filebrowser
    restart: unless-stopped
    networks:
      - network_public
    volumes:
      - /mnt/cloud_main:/srv/cloud_main:rw
      - /mnt/cloud_temp:/srv/cloud_temp:rw
      - filebrowser_data:/config
    environment:
      - FB_DATABASE=/config/database.db
      - FB_SCHEME=https
      - FB_HOST=${DOMINIO_CLOUD}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.filebrowser.rule=Host(`${DOMINIO_CLOUD}`)"
      - "traefik.http.routers.filebrowser.entrypoints=web"
      - "traefik.http.routers.filebrowser.priority=1"
      - "traefik.http.services.filebrowser.loadbalancer.server.port=80"
      - "traefik.http.services.filebrowser.loadbalancer.passHostHeader=1"
      - "traefik.docker.network=network_public"
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          memory: 128M
```

**IMPORTANT:** Also add to volumes:
```yaml
volumes:
  filebrowser_data:
```

### **Step 4: Add DNS Route in Cloudflare**

```bash
cloudflared tunnel route dns n8n-raspberry cloud.yourdomain.com
```

### **Step 5: Update Cloudflare Tunnel Config**

Edit `~/.cloudflared/config.yml` and add:

```yaml
ingress:
  - hostname: central2.yourdomain.com
    service: http://localhost:80
  - hostname: hook2.yourdomain.com
    service: http://localhost:80
  - hostname: app2.yourdomain.com
    service: http://localhost:80
  - hostname: cloud.yourdomain.com    # NEW!
    service: http://localhost:80
  - service: http_status:404
```

### **Step 6: Copy to /etc/cloudflared**

```bash
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml
sudo systemctl restart cloudflared
```

### **Step 7: Start Services**

```bash
cd ~/Downloads
docker compose up -d
```

### **Step 8: Configure FileBrowser User**

Access https://cloud.yourdomain.com

**Login:**
- Username: `admin`
- Password: Check logs or auto-generated

**IMPORTANT - Change password IMMEDIATELY:**
1. After first login
2. Go to Settings → Change Password
3. Change to a strong, memorable password!
4. Optionally, add more users if needed

**Get current password (if needed):**
```bash
docker compose logs filebrowser | grep "password"
```
Look for: "User 'admin' initialized with randomly generated password: XXXX"

### **Step 9: Configure Windows**

**Windows Explorer:**
```
1. Press Win + R
2. Type: \\[RASPBERRY-IP]
   Example: \\192.168.1.100
3. Enter username: robson
4. Enter password: [configured password]
5. Done! You'll see:
   - arquivos (RAID 1)
   - temp (normal disk)
```

## 📊 Final Structure

```
/mnt/cloud_main (RAID 1)      → Important files
  └── arquivos/                → Samba + FileBrowser
      └── My Documents
          └── Work
          └── Personal

/mnt/cloud_temp (Normal)       → Temporary files
  └── temporarios/             → Samba + FileBrowser
      └── Downloads
      └── Temp
```

## 🔒 Security

### **Firewall (Ubuntu)**

```bash
# Allow Samba only on local network
sudo ufw allow from 192.168.0.0/16 to any port 445
sudo ufw allow from 192.168.0.0/16 to any port 139
sudo ufw reload
```

### **Automatic Backup (Optional)**

```bash
# Create backup script
cat > ~/backup_nuvem.sh << 'EOF'
#!/bin/bash
# Backup RAID 1 → external
DATE=$(date +%Y%m%d)
rsync -av /mnt/cloud_main/ /media/backup/nuvem_main_$DATE
EOF

chmod +x ~/backup_nuvem.sh

# Schedule daily backup at 3am
crontab -e
# Add: 0 3 * * * /home/robson/backup_nuvem.sh
```

## ✅ How to Use

### **Local (Windows):**
1. Open Windows Explorer
2. Type `\\192.168.x.x`
3. Enter credentials
4. Drag files to `arquivos` or `temp`

### **Remote (Mobile/PC):**
1. Open browser
2. Access https://cloud.yourdomain.com
3. Login
4. Upload/Download files

## 🚀 Advantages

✅ **Fast local access** (Samba)  
✅ **Secure remote access** (Cloudflare Tunnel)  
✅ **RAID 1 protects important data**  
✅ **Simple drag and drop**  
✅ **Automatic SSL** (via Traefik)  
✅ **No open ports** (Cloudflare)  
✅ **Multiplatform** (Windows, Android, etc)

## 📝 Useful Commands

```bash
# Check disk usage
df -h /mnt/cloud_main /mnt/cloud_temp

# Check space used per folder
du -sh /mnt/cloud_main/*

# Restart Samba
docker compose restart samba

# View FileBrowser logs
docker compose logs -f filebrowser

# View Samba logs
docker compose logs -f samba
```

---

## ⚠️ Troubleshooting

### **404 Error in FileBrowser**

If Traefik returns 404, check if the `docker-compose.yml` Traefik configuration is correct:

**❌ WRONG (causes 404):**
```yaml
command:
  - "--log.level=INFO"
  - "--entrypoints.web.http.middlewares=headers@docker"  # ← CAUSES 404!
```

**✅ CORRECT:**
```yaml
command:
  - "--log.level=INFO"
  # Remove the line above if present
```

**Remove the line** `"--entrypoints.web.http.middlewares=headers@docker"` if present, as it references a non-existent middleware.

**Then execute:**
```bash
docker compose up -d traefik
```

### **DNS_PROBE_FINISHED_NXDOMAIN on Windows**

**Problem:** Static Windows DNS cache.

**Solution:**
```cmd
# Open CMD as Administrator
ipconfig /flushdns
netsh winsock reset
```

Or restart Windows.

### **Samba Not Showing on Windows**

Check if ports are open on firewall:
```bash
sudo ufw allow from 192.168.0.0/16 to any port 445
sudo ufw allow from 192.168.0.0/16 to any port 139
```

### **Cannot Create Files/Folders in FileBrowser**

**Problem:** Incorrect permissions. Disks mounted as `root:root`.

**Solution:**
```bash
# Check current owner
ls -ld /mnt/cloud_main /mnt/cloud_temp

# Fix to UID 1000 (robson or user)
sudo chown -R robson:robson /mnt/cloud_main
sudo chown -R robson:robson /mnt/cloud_temp

# Give write permissions
sudo chmod -R 775 /mnt/cloud_main
sudo chmod -R 775 /mnt/cloud_temp

# Restart FileBrowser
docker compose restart filebrowser
```

### **FileBrowser Shows Wrong Space (117 GB instead of 5TB/7TB)**

This is normal! FileBrowser shows:
- At root: System disk space (~117 GB)
- Inside `cloud_main`: Shows ~5.4 TB
- Inside `cloud_temp`: Shows ~7.2 TB

Expected FileBrowser behavior - it calculates space of current disk where it's operating.

### **Files Created with Restricted Permission (640)**

**If files created by FileBrowser have 640 permission:**
```bash
# Fix permissions of existing files
chmod -R 664 /mnt/cloud_main/arquivos/*
chmod -R 664 /mnt/cloud_temp/temporarios/*

# After that, new files will be created with 664 (configured with umask 002)
```

**Samba access:** Will work normally since you use user `robson` which has permission.

---

## 📝 Important Notes

### **FileBrowser Permissions**

- FileBrowser container runs as user `user` (UID/GID 1000)
- User `robson` on host also has UID 1000
- When configuring permissions, use `robson:robson` or `1000:1000`
- The FileBrowser "admin" is just web login, doesn't affect system permissions

### **Permission Structure**

```
robson (UID 1000) on host
    ↕
user (UID 1000) in FileBrowser container
    ↕
Access to files in /mnt/cloud_main and /mnt/cloud_temp
```

---

**Next step:** Prepare disks and add to docker-compose.yml

