# Leave Desk

A leave application app for office staff and site labour, hosted on your own server.

- **Staff** open the app on any phone, sign in with their **Employee ID and a PIN**, apply for leave, upload a medical certificate photo, and see HR's decision. They don't need an email address or an app store. The staff screens work in English, Arabic, Hindi, Urdu and Malayalam.
- **HR** sign in at `/hr` with email and password. New requests appear live with a pop-up, an optional sound and a count in the browser tab. HR can also get **push notifications** on phones and computers, and email alerts. HR approves or rejects with a note, manages employees, and exports to Excel.

Everything runs in Docker: a PostgreSQL database, the app, and Caddy, which provides free automatic HTTPS.

---

## What's in this folder

| Path | What it is |
|---|---|
| `setup.sh` | One-time setup script for the server |
| `docker-compose.yml` | Runs the database, the app and HTTPS together |
| `Dockerfile` | How the app is built |
| `Caddyfile` | HTTPS and web server settings |
| `.env.example` | Settings template (`setup.sh` creates your real `.env`) |
| `scripts/backup.sh`, `scripts/restore.sh` | Backup and restore |
| `src/` | The app's code (Next.js + React + TypeScript) |
| `src/db/schema.ts` | Database tables |
| `drizzle/` | Database migrations, applied automatically on start |
| `CLAUDE.md` | Notes for extending the app with Claude Code |

---

## What you need

- **A Linux server**, preferably Ubuntu 22.04 or 24.04, with at least 2 CPU cores, 4 GB RAM and 40 GB disk. A cloud server works as well as one in your office.
- **A web address**, such as `leave.yourcompany.ae`, that you can point at the server.
- **Ports 80 and 443 open to the internet.** Site staff use mobile data, so the app must be reachable from outside your office network.
- Someone who can type commands on the server over SSH. You'll copy and paste every command below.

---

## Step 1: Point your web address at the server

In your domain's DNS settings (wherever you manage `yourcompany.ae`), add an **A record**:

| Type | Name | Value |
|---|---|---|
| A | `leave` | your server's public IP address |

DNS changes can take anything from a few minutes to a few hours to work. You can check with `ping leave.yourcompany.ae`.

## Step 2: Install Docker on the server

Connect to the server with SSH, then run:

```bash
sudo apt update && sudo apt install -y curl openssl unzip
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

Log out and back in, then check that Docker works with `docker compose version`.

If the server uses the Ubuntu firewall, open the web ports:

```bash
sudo ufw allow OpenSSH && sudo ufw allow 80 && sudo ufw allow 443 && sudo ufw enable
```

## Step 3: Copy Leave Desk to the server

From your computer, copy the zip to the server. On Windows you can use **WinSCP**, or this command:

```bash
scp leave-desk.zip youruser@your-server-ip:~
```

Then on the server:

```bash
mkdir -p ~/apps && cd ~/apps
unzip ~/leave-desk.zip -d leave-desk
cd leave-desk
chmod +x setup.sh scripts/*.sh
```

## Step 4: Run setup

```bash
./setup.sh
```

The script asks five questions: web address, company name, your email, your name and time zone. It then:

1. creates `.env` with strong random passwords and secret keys,
2. builds the app, which takes 3–5 minutes the first time,
3. creates the push notification keys,
4. starts everything.

At the end it prints the **HR sign-in email and password**. Write them down.

Open `https://leave.yourcompany.ae/hr` and sign in. If the browser warns about the certificate, wait a minute and reload, because Caddy is still getting the HTTPS certificate.

> **Testing without a domain:** you can use the app inside your office network before DNS is ready. Answer `:80` when setup asks for the web address, then open `http://<server-ip>`. Phone push notifications need HTTPS, so they don't work in this mode. Later, change `DOMAIN` and `APP_URL` in `.env` and run `docker compose up -d`.

## Step 5: First sign-in checklist

1. **My account:** change the password that setup printed.
2. **Settings, Company and weekends:** set the weekly days off for office staff and for site staff.
3. **Settings, Leave types:** check each type's yearly allowance, whether it counts calendar or working days, and whether it needs a document. **The starting values are common defaults, not legal advice. Check them against your contracts and UAE labour law.**
4. **Settings, Public holidays:** add this year's holidays. Islamic holidays move each year, so add them when they're announced.
5. **Settings, HR team:** add your HR colleagues. Each one gets a temporary password to change under *My account*.

## Step 6: Add your employees

**Option A: one at a time.** Go to **Employees**, click **Add employee**, and the app shows the new PIN.

**Option B: import from Excel.** This is best for many people.

1. Make a sheet with these column headings in the first row:
   `Employee ID, Name, Mobile, Department, Site, Staff type, Joining date`
   Only *Employee ID* and *Name* are required. *Staff type* is `Office` or `Site`. Dates can be `18/10/2026` or `2026-10-18`.
2. In Excel, use **File → Save As → CSV UTF-8**. The UTF-8 version keeps Arabic and other scripts intact.
3. Go to **Employees**, click **Choose CSV file**, then **Download PIN list**.

Each new employee gets a random 4-digit PIN. **PINs are shown only once.** Print the PIN list and hand each person their PIN, through their supervisor for site staff. If someone forgets their PIN, open them under **Employees** and click **Reset PIN**.

Importing the same file again updates existing people. It doesn't change their PINs.

## Step 7: Give staff the link

- Staff address: `https://leave.yourcompany.ae`
- Make a **QR code** of that address with any QR generator and put it up at site offices, camps and time-attendance machines. You can also share it in WhatsApp groups.
- Staff can tap **Add to Home Screen** in Chrome or Safari, and the app then opens like a normal app.
- Staff stay signed in on their phone for 30 days. After 5 wrong PINs, their account locks for 15 minutes.

## Step 8: HR notifications

| How | Works when | Set up |
|---|---|---|
| Pop-up and tab count on the dashboard | the dashboard is open | Nothing to do. Click **Turn on sound alerts** for a chime. |
| **Push notifications** to phone or computer | even when the browser is closed | On the dashboard, click **Turn on notifications** and allow them. Do this on each device. |
| Email | always | Fill in `SMTP_…` in `.env` (see below). Each HR user can switch it off under *My account*. |

**Push on phones:** on Android, open `https://leave.yourcompany.ae/hr` in Chrome and turn on notifications. On iPhone (iOS 16.4 or later), open it in Safari, tap **Share → Add to Home Screen**, open the app from the home screen, and turn on notifications there.

**Email alerts:** edit `.env`, for example with `nano .env`. These are the values for Microsoft 365:

```
SMTP_HOST=smtp.office365.com
SMTP_PORT=587
SMTP_USER=leave-alerts@yourcompany.ae
SMTP_PASSWORD=the-mailbox-password
SMTP_FROM=Leave Desk <leave-alerts@yourcompany.ae>
```

Then run `docker compose up -d`. For Google Workspace use `smtp.gmail.com` with an app password.

## Step 9: Back up every night

Run a backup now to check it works:

```bash
./scripts/backup.sh
```

To schedule one every night at 2 am, run `crontab -e` and add this line, replacing the path with yours:

```
0 2 * * * cd /home/youruser/apps/leave-desk && ./scripts/backup.sh >> backups/backup.log 2>&1
```

Backups are kept for 30 days in `backups/`. **Also copy them off the server regularly**, for example to company OneDrive or another machine. A backup on the same disk doesn't protect you if that disk fails.

To restore:

```bash
./scripts/restore.sh backups/db-YYYYMMDD-HHMM.dump backups/uploads-YYYYMMDD-HHMM.tgz
```

---

## Everyday commands

Run these from the `leave-desk` folder.

| Task | Command |
|---|---|
| See if it's running | `docker compose ps` |
| See the app's log | `docker compose logs -f app` (Ctrl+C to stop) |
| Restart | `docker compose restart app` |
| Apply `.env` changes | `docker compose up -d` |
| Stop everything | `docker compose down` (your data is kept) |
| Install an updated version | copy the new files over this folder (keep `.env`), then `docker compose up -d --build` |

Database changes in a new version are applied automatically when the app starts. **Run a backup before every update.**

## Troubleshooting

| Problem | What to check |
|---|---|
| Browser can't reach the site | DNS (Step 1), ports 80/443 open in the firewall and the cloud provider's security group, `docker compose ps` |
| Certificate error | DNS must point at this server before Caddy can get a certificate. Check with `docker compose logs caddy`. |
| HR password lost | Another administrator can reset it under Settings. If no administrator can sign in, run `docker compose exec app node scripts/create-admin.mjs you@company.ae "Your Name"` on the server. It prints a temporary password. |
| Employee locked out | Wait 15 minutes, or reset their PIN under Employees. |
| "Turn on notifications" does nothing | The site must be opened over HTTPS. On iPhone, open the app from the home screen first. |
| No emails | Check the `SMTP_…` values, then look at `docker compose logs app` for "Email alert failed". |
| App keeps restarting | `docker compose logs app`. Usually `.env` is missing `SESSION_SECRET` or `DB_PASSWORD`. |

**Windows Server:** this setup is made for Linux. On Windows Server, the simplest route is a small Ubuntu virtual machine (Hyper-V), then follow this guide inside it.

---

## Security notes

- Leave and sick-leave records are personal data. Only give HR accounts to people who need them, and remove leavers promptly by deactivating them in Settings or Employees.
- Deactivating an employee or resetting a PIN signs them out of every phone.
- Every sign-in lockout, submission, decision, import and export is recorded in the `audit_log` table.
- Keep the server updated: run `sudo apt update && sudo apt upgrade` monthly, and rebuild the app with `docker compose up -d --build` to pick up base image updates.

---

## For developers: running and extending locally

You'll need Node.js 22 and Docker Desktop.

```bash
docker run -d --name ld-db -e POSTGRES_PASSWORD=dev -e POSTGRES_USER=leavedesk -e POSTGRES_DB=leavedesk -p 5432:5432 postgres:16-alpine
npm install
cp .env.example .env.local
#   set DATABASE_URL=postgresql://leavedesk:dev@localhost:5432/leavedesk
#   set SESSION_SECRET (any 32+ characters), ADMIN_EMAIL, ADMIN_PASSWORD
set -a; source .env.local; set +a
npm run db:migrate && npm run db:seed
npm run dev        # http://localhost:3000 for staff, http://localhost:3000/hr for HR
```

To change the database:
1. Edit `src/db/schema.ts`.
2. Run `npm run db:generate`. This writes a new file in `drizzle/`.
3. Commit both files. The server applies the change on its next start.

See `CLAUDE.md` for the project's conventions. It's written so Claude Code can extend the app safely.
