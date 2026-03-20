# Akamai VM — Node.js App Deployment Guide

**Server:** `172.105.122.33`
**OS:** Ubuntu 24.04 LTS
**Stack:** Nginx + PM2 + Node.js

---

## Deploying a New Node.js App

### Step 1 — Add a DNS A Record

Go to **Spaceship.com** → DNS → Add A Record:

| Field | Value |
|---|---|
| Type | A |
| Name | `yourapp` (e.g. `myapi`) |
| Value | `172.105.122.33` |
| TTL | 300 |

This makes `yourapp.mahfuzul.online` point to your server. Wait 1–2 minutes for DNS to propagate.

---

### Step 2 — SSH into the Server

```bash
ssh mahfuzul@172.105.122.33
```

---

### Step 3 — Clone Your GitHub Repo

```bash
cd ~
git clone git@github.com:mhmitas-dev/YOUR_REPO_NAME.git
cd YOUR_REPO_NAME
```

> GitHub SSH is already configured. No token needed.

---

### Step 4 — Install Dependencies

```bash
npm install
```

---

### Step 5 — Add Environment Variables

```bash
nano .env
```

Paste your environment variables, then save with **Ctrl+X → Y → Enter**.

---

### Step 6 — Build (TypeScript projects only)

If your project uses TypeScript:

```bash
npm run build
```

---

### Step 7 — Start with PM2

```bash
pm2 start npm --name "your-app-name" -- start
```

Check it's running:

```bash
pm2 list
```

You should see `status: online`.

Save the process list so it survives reboots:

```bash
pm2 save
```

---

### Step 8 — Create Nginx Config

```bash
sudo nano /etc/nginx/sites-available/your-app-name
```

Paste this (replace `yourapp.mahfuzul.online` and `PORT`):

```nginx
server {
    listen 80;
    server_name yourapp.mahfuzul.online;

    location / {
        proxy_pass http://localhost:PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Save with **Ctrl+X → Y → Enter**.

---

### Step 9 — Enable the Site

```bash
sudo ln -s /etc/nginx/sites-available/your-app-name /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

### Step 10 — Get SSL Certificate

```bash
sudo certbot --nginx -d yourapp.mahfuzul.online
```

Follow the prompts. Certbot will automatically configure HTTPS.

---

### Step 11 — Test

```bash
curl https://yourapp.mahfuzul.online
```

You should get a response from your app. 🎉

---

## Updating an Existing App

When you push new code to GitHub:

```bash
cd ~/YOUR_REPO_NAME
git pull
npm install          # only if you added new packages
npm run build        # only for TypeScript projects
pm2 restart your-app-name
```

---

## One-Click Deploy Script (Optional)

Create a deploy script so you only run one command:

```bash
nano ~/YOUR_REPO_NAME/deploy.sh
```

Paste:

```bash
#!/bin/bash
cd ~/YOUR_REPO_NAME
git pull
npm install
npm run build        # remove this line if not TypeScript
pm2 restart your-app-name
echo "✅ Deployed successfully!"
```

Make it executable:

```bash
chmod +x ~/YOUR_REPO_NAME/deploy.sh
```

To deploy in the future, just run:

```bash
~/YOUR_REPO_NAME/deploy.sh
```

---

## Useful PM2 Commands

| What you need | Command |
|---|---|
| List all apps | `pm2 list` |
| View logs | `pm2 logs your-app-name` |
| Restart app | `pm2 restart your-app-name` |
| Stop app | `pm2 stop your-app-name` |
| Delete app | `pm2 delete your-app-name` |
| Monitor CPU/RAM | `pm2 monit` |

## Useful Nginx Commands

| What you need | Command |
|---|---|
| Test config | `sudo nginx -t` |
| Reload config | `sudo systemctl reload nginx` |
| Restart Nginx | `sudo systemctl restart nginx` |
| Check status | `sudo systemctl status nginx` |

---

## Currently Running Apps

| App | Domain | Port | PM2 Name |
|---|---|---|---|
| Teddybot API | `https://teddybot-api.mahfuzul.online` | 3100 | `free4talk-bot` |

---

*Last updated: March 2026*
