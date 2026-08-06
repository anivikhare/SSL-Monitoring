# SSL Monitor

A self-hosted web application for tracking SSL/TLS certificate expiry across your domains. It checks certificates on a schedule, sends email alerts before they expire, and gives you a dashboard to see the health of every domain at a glance.

## Features

- **Dashboard** with per-project tabs, live search, status filters, and sortable domain lists
- **Certificate details** per domain: issuer (CA), subject CN, SAN list, chain validation, TLS protocol/cipher, and handshake response time
- **Certificate change detection** — flags when a cert is renewed or replaced
- **Email alerts** at configurable thresholds (e.g. 30/14/7 days before expiry) with daily repeats when close
- **Reminders** — standalone or domain-linked, with email + dashboard banner, using the same threshold logic as cert alerts
- **Projects & Groups** to organize domains, with per-project/group alert recipients
- **Bulk import** domains from CSV/text, with duplicate detection
- **Bulk actions** — re-check or delete multiple domains at once
- **Proxy support** for domains reachable only through a corporate HTTP proxy
- **User management** with roles (admin / user / read-only), local or LDAP/AD authentication, and last-login tracking
- **Activity log** of logins and changes
- **Trash** with soft-delete and 30-day retention

## Architecture

- **Backend**: Node.js + Express, SQLite database, runs on port `8000`
- **Frontend**: React (built to static files), served on port `3000`
- **Process manager**: PM2
- **Node version**: 20.x

---

## Prerequisites

- A Linux server (tested on RHEL/CentOS and Ubuntu)
- Root or sudo access
- Outbound network access to the domains you want to monitor
- An SMTP server if you want email alerts

---

## Installation

### 1. Install Node.js 20

```bash
cd /opt
wget https://nodejs.org/dist/v20.14.0/node-v20.14.0-linux-x64.tar.gz
tar -xzf node-v20.14.0-linux-x64.tar.gz
rm node-v20.14.0-linux-x64.tar.gz

# Add Node to PATH
echo 'export PATH=/opt/node-v20.14.0-linux-x64/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Verify
node -v      # should print v20.14.0
npm -v
```

### 2. Install PM2 and serve globally

```bash
npm install -g pm2 serve
```

### 3. Extract the application

```bash
cd /opt
tar -xzf ssl-monitor-v12.tar.gz
# This creates /opt/ssl-monitor
```

### 4. Install and start the backend

```bash
cd /opt/ssl-monitor/backend
npm install
pm2 start server.js --name ssl-backend
```

The backend creates the SQLite database (`ssl_monitor.db`) and a default admin user on first run.

### 5. Build and start the frontend

Replace `YOUR_SERVER_IP` with the server's IP or hostname:

```bash
cd /opt/ssl-monitor/frontend
npm install
REACT_APP_API_URL=http://YOUR_SERVER_IP:8000 npm run build
pm2 start "serve -s /opt/ssl-monitor/frontend/build -l 3000" --name ssl-frontend
```

> The `REACT_APP_API_URL` must point to the backend so the browser can reach the API. If you change the server IP later, rebuild the frontend with the new value.

### 6. Save the PM2 process list and enable startup on boot

```bash
pm2 save
pm2 startup
# Run the command that pm2 startup prints
```

### 7. Open the firewall ports

```bash
firewall-cmd --permanent --add-port=3000/tcp
firewall-cmd --permanent --add-port=8000/tcp
firewall-cmd --reload
```

(On Ubuntu with ufw: `ufw allow 3000/tcp && ufw allow 8000/tcp`)

---

## First Login

Open `http://YOUR_SERVER_IP:3000` in a browser.

| | |
|---|---|
| **Username** | `admin` |
| **Password** | `admin123` |

**Change the admin password immediately** via the "Change Password" link in the top bar.

---

## Configuration

All configuration is done in the app under **Settings** (admin only). Nothing needs to be edited in code.

### SMTP (for email alerts)
Settings → SMTP tab. Enter your mail server host, port, from-address, and (if required) username/password. Use "Send Test Email" to verify.

### Alert thresholds
Settings → Alerts tab. Configure the days-before-expiry thresholds, the daily-repeat window, and a global CC address. These thresholds apply to both certificate alerts and reminders.

### Schedule
Settings → Schedule tab. Set how often certificates are checked and what time daily alerts are sent (in IST).

### LDAP / Active Directory (optional)
Settings → LDAP tab. Enable and enter your directory URL and domain if you want users to authenticate against AD instead of local passwords.

### Proxy (optional)
Settings → Proxy tab. If some domains are only reachable through a corporate HTTP proxy, enter the proxy host and port here. Then enable "Via Proxy" per-domain when adding it.

---

## Adding Domains

1. Go to **Projects** → create a Project (e.g. a team or environment).
2. Add a Group inside the project.
3. Add domains (FQDNs) individually, or use **Bulk Import** to paste/upload a CSV.

CSV format for bulk import:

```
fqdn,port,type,proxy
example.com,443,external,false
internal.example.com,443,intranet,false
service.example.com,443,external,true
```

Certificates are checked automatically after they're added, on the configured schedule, and whenever you click the check button.

---

## Common Operations

### View logs
```bash
pm2 logs ssl-backend
pm2 logs ssl-frontend
```

### Restart services
```bash
pm2 restart ssl-backend
pm2 restart ssl-frontend
```

### Check status
```bash
pm2 status
```

### Back up the database
The entire application state lives in one SQLite file:
```bash
cp /opt/ssl-monitor/backend/ssl_monitor.db /path/to/backup/ssl_monitor_$(date +%F).db
```

### Migrate data to another server
Copy `ssl_monitor.db` into the new server's `/opt/ssl-monitor/backend/` directory before starting the backend, and it will pick up all projects, domains, users, and settings.

---

## Upgrading

To deploy a new version while keeping your data:

```bash
pm2 stop ssl-backend

# Copy over the changed backend files (or the whole backend folder,
# but do NOT overwrite ssl_monitor.db)
# Then run any migration commands included with the release.

pm2 restart ssl-backend

# Rebuild the frontend
cd /opt/ssl-monitor/frontend
REACT_APP_API_URL=http://YOUR_SERVER_IP:8000 npm run build
pm2 restart ssl-frontend
```

The database schema auto-migrates on backend startup (new columns/tables are added automatically). Your existing data is preserved.

---

## Troubleshooting

**Frontend loads but can't log in / API errors**
The frontend was built with the wrong API URL. Rebuild with the correct `REACT_APP_API_URL` pointing to `http://YOUR_SERVER_IP:8000`.

**No emails arriving**
Check Settings → SMTP is filled in and use "Send Test Email". Confirm the server can reach your SMTP host on the configured port. Check `pm2 logs ssl-backend` for mail errors.

**A domain shows "403 Proxy Blocked"**
The proxy is refusing the CONNECT tunnel for that domain. It needs to be whitelisted on the proxy for the server's IP — this is a network-side change, not an app issue.

**Backend won't start**
Run `pm2 logs ssl-backend --err` to see the error. Most startup failures are a missing `npm install` or a port already in use.

**Changes not showing in the browser**
Hard-refresh with `Ctrl+Shift+R` to clear cached JavaScript after a frontend rebuild.

---

## Ports Summary

| Service | Port | Purpose |
|---|---|---|
| Frontend | 3000 | Web UI |
| Backend | 8000 | REST API |

---

## Default Credentials

| Username | Password |
|---|---|
| admin | admin123 |

**Change this immediately after first login.**
