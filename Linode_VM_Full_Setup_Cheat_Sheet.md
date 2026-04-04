# Linode/Akamai VM — Full Setup Cheat Sheet
**XFCE + TigerVNC + Chrome + Node.js + PM2 + Nginx + SSL on Ubuntu 24.04 LTS**

---

## Prerequisites

- A Linode VM with Ubuntu 24.04 LTS
- A domain (e.g. on Spaceship.com)
- RealVNC Viewer on Windows: https://www.realvnc.com/en/connect/download/viewer/windows/
- SSH client (Windows Terminal / PowerShell)

---

## Step 1 — Rebuild the VM (Fresh Start)

In **Akamai/Linode Cloud Manager**:
1. Go to **Linodes** → click your VM
2. Click the **3 dot menu** → **Rebuild**
3. Select **Image**: Ubuntu 24.04 LTS
4. Set a root password
5. Click **Rebuild Linode** (takes ~2 minutes)

---

## Step 2 — First SSH Connection (as root)

```bash
ssh root@YOUR_VM_IP
```

---

## Step 3 — Update the System

```bash
apt update && apt upgrade -y
```

---

## Step 4 — Install XFCE Desktop

```bash
apt install -y xfce4 xfce4-goodies
```

> If prompted to choose a display manager, select **lightdm**.

---

## Step 5 — Install TigerVNC

```bash
apt install -y tigervnc-standalone-server tigervnc-common dbus-x11
```

---

## Step 6 — Create a Non-Root User

> **Critical:** Chrome will not run as root. Always create a regular user for VNC and Chrome.

```bash
adduser mahfuzul
usermod -aG sudo mahfuzul
```

---

## Step 7 — Set Up VNC for the New User

Switch to the new user:

```bash
su - mahfuzul
```

Set VNC password:

```bash
vncpasswd
```

Create the xstartup file:

```bash
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/bash
export XDG_SESSION_TYPE=x11
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec /usr/bin/xfce4-session
EOF

chmod +x ~/.vnc/xstartup
```

Go back to root:

```bash
exit
```

---

## Step 8 — Create the VNC Systemd Service

> **Important:** `User=` must be your non-root user, NOT root.

```bash
sudo tee /etc/systemd/system/vncserver@.service << 'EOF'
[Unit]
Description=VNC Server for display :%i
After=network.target

[Service]
Type=simple
User=mahfuzul
ExecStart=/usr/bin/vncserver -fg -geometry 1920x1080 -depth 24 -localhost no :%i
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start VNC:

```bash
systemctl daemon-reload
systemctl enable vncserver@1.service
systemctl start vncserver@1.service
systemctl status vncserver@1.service
```

You should see **Active: active (running)**.

---

## Step 9 — Connect via VNC

Open RealVNC Viewer on Windows and connect to:

```
YOUR_VM_IP::5901
```

Enter your VNC password. You should see the XFCE desktop.

---

## Step 10 — Install Google Chrome

Run as root:

```bash
wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | gpg --dearmor -o /usr/share/keyrings/google-chrome.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google-chrome.gpg] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list
apt update && apt install -y google-chrome-stable
```

Chrome will now open normally from the XFCE application menu (since VNC runs as a non-root user).

---

## Step 11 — Install Emoji/Special Character Fonts

```bash
sudo apt install -y fonts-noto-color-emoji fonts-noto fonts-liberation
```

Log out and back into VNC to apply.

---

## Step 12 — Set Up the Linode Firewall

In **Akamai/Linode Cloud Manager** → your VM → **Firewall** tab, make sure these inbound rules exist:

| Label | Protocol | Port | Source |
|---|---|---|---|
| Allow-SSH | TCP | 22 | All IPv4 |
| port-80 | TCP | 80 | All IPv4 |
| port-443 | TCP | 443 | All IPv4 |
| vnc | TCP | 5901 | All IPv4 (or your IP only) |

> **Note:** Restrict port 5901 to your own IP for better security if possible.

---

## Step 13 — Install Node.js and PM2

Switch to your non-root user:

```bash
su - mahfuzul
```

Install Node.js 22 and PM2:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
```

---

## Step 14 — Set Up GitHub SSH Key

```bash
ssh-keygen -t ed25519 -C "your@email.com"
cat ~/.ssh/id_ed25519.pub
```

Copy the output and add it to **GitHub → Settings → SSH and GPG keys → New SSH key**.

Test the connection:

```bash
ssh -T git@github.com
```

You should see: `Hi username! You've successfully authenticated.`

---

## Step 15 — Clone and Run Your Node.js App

```bash
cd ~
git clone git@github.com:YOUR_USERNAME/YOUR_REPO.git
cd YOUR_REPO
npm install
nano .env                    # add your environment variables
npm run build                # TypeScript projects only
pm2 start npm --name "your-app-name" -- start
pm2 save
pm2 startup                  # copy and run the command it outputs
```

---

## Step 16 — Install and Configure Nginx

```bash
sudo apt install -y nginx
```

Add a DNS A record in Spaceship.com first:

| Field | Value |
|---|---|
| Type | A |
| Name | `yoursubdomain` |
| Value | `YOUR_VM_IP` |
| TTL | 300 |

Create Nginx config:

```bash
sudo nano /etc/nginx/sites-available/your-app-name
```

Paste:

```nginx
server {
    listen 80;
    server_name yoursubdomain.yourdomain.com;

    location / {
        proxy_pass http://localhost:YOUR_PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/your-app-name /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## Step 17 — Get SSL Certificate

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yoursubdomain.yourdomain.com
```

Test it:

```bash
curl https://yoursubdomain.yourdomain.com
```

---

## Deploying Updates

When you push new code to GitHub:

```bash
cd ~/YOUR_REPO
git pull
npm install          # only if new packages were added
npm run build        # TypeScript only
pm2 restart your-app-name
```

Or create a deploy script:

```bash
nano ~/YOUR_REPO/deploy.sh
```

```bash
#!/bin/bash
cd ~/YOUR_REPO
git pull
npm install
npm run build
pm2 restart your-app-name
echo "✅ Deployed!"
```

```bash
chmod +x ~/YOUR_REPO/deploy.sh
```

Run future deploys with:

```bash
~/YOUR_REPO/deploy.sh
```

---

## Quick Reference Commands

### VNC

| What you need | Command |
|---|---|
| Start VNC | `sudo systemctl start vncserver@1.service` |
| Stop VNC | `sudo systemctl stop vncserver@1.service` |
| Restart VNC | `sudo systemctl restart vncserver@1.service` |
| Check VNC status | `sudo systemctl status vncserver@1.service` |
| Change VNC password | `vncpasswd` then restart VNC |

### PM2

| What you need | Command |
|---|---|
| List all apps | `pm2 list` |
| View logs | `pm2 logs your-app-name` |
| Restart app | `pm2 restart your-app-name` |
| Stop app | `pm2 stop your-app-name` |
| Delete app | `pm2 delete your-app-name` |
| Monitor CPU/RAM | `pm2 monit` |

### Nginx

| What you need | Command |
|---|---|
| Test config | `sudo nginx -t` |
| Reload config | `sudo systemctl reload nginx` |
| Restart Nginx | `sudo systemctl restart nginx` |
| Check status | `sudo systemctl status nginx` |

### System

| What you need | Command |
|---|---|
| Check RAM | `free -h` |
| Check disk | `df -h` |
| Check running containers | `docker ps` |
| Check open ports | `ss -tlnp` |
| Public IP | `curl -s ifconfig.me` |

---

## Common Pitfalls

| Problem | Cause | Fix |
|---|---|---|
| Chrome won't open | Running VNC as root | Always run VNC as a non-root user (Step 6–8) |
| Emojis not showing | Missing fonts | `sudo apt install -y fonts-noto-color-emoji` |
| 502 Bad Gateway | Node.js app crashed | Check `pm2 list` and `pm2 logs your-app-name` |
| Can't reach domain | Firewall blocking port | Check Linode firewall rules (Step 12) |
| SSL fails | DNS not propagated yet | Wait 2–5 minutes after adding A record |
| PM2 app not starting after reboot | Startup not configured | Run `pm2 startup` and copy/paste the output command |
| Git requires token every time | Using HTTPS instead of SSH | Use SSH clone URL: `git clone git@github.com:...` |

---

*Tested on: Akamai/Linode, Ubuntu 24.04.4 LTS, March 2026*
