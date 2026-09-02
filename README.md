# Gophish on `itsupport.insec.in`

Production wrapper around official **[Gophish](https://github.com/gophish/gophish) v0.12.1**. It runs on the same Ubuntu VPS as AwareCheck without taking ports **80/443** (nginx) or **8000 / 8001 / 9000**.

Public names are on **insec.in**, not AwareCheck:

| Role | Hostname |
|------|----------|
| Staff click this (campaign URL) | `https://itsupport.insec.in` |
| You log in here (Gophish admin) | `https://admin.itsupport.insec.in` |

Admin and landing pages cannot share the same host path: both apps serve `/`. Employees must hit the landing server; you must hit the admin server.

Use this only for **authorized security-awareness simulations**.

## What was changed vs stock Gophish

| Stock default | This package |
|---------------|--------------|
| Admin TLS on `127.0.0.1:3333` (self-signed) | Admin HTTP on `127.0.0.1:3333`, TLS at nginx |
| Phish server on public `:80` | Phish HTTP on `127.0.0.1:8082` |
| No systemd / nginx | `gophish` user, systemd unit, nginx site |
| Admin CSRF origin empty | `trusted_origins` = `admin.itsupport.insec.in` |

Runtime files (binary, SQLite DB) live in `/opt/gophish/runtime` so git pulls never overwrite campaigns.

## VPS layout

| App | Domain | Path | Local port |
|-----|--------|------|------------|
| AwareCheck | `awarechck.com` | `/opt/awarecheck` | `8000` |
| Data Prahari | `dpdpsec.com` | `/opt/dpdpsec` | `8001` |
| Insec-Hydra | `hydra.insec.in` | `/opt/insec-hydra` | `9000` |
| **Gophish admin** | `admin.itsupport.insec.in` | `/opt/gophish/runtime` | `3333` |
| **Gophish campaigns** | `itsupport.insec.in` | same | `8082` |

## 1. DNS (do this first)

In the **insec.in** DNS panel add:

| Type | Name | Value |
|------|------|--------|
| A | `itsupport` | `77.107.95.65` |
| A | `admin.itsupport` | `77.107.95.65` |

Wait until `dig +short itsupport.insec.in` returns `77.107.95.65`.

## 2. Install on the VPS

SSH in as root, then:

```bash
apt-get update && apt-get install -y git
git clone https://github.com/fir3storm/gophish.git /opt/gophish
bash /opt/gophish/scripts/install.sh
```

If `/opt/gophish` already exists:

```bash
cd /opt/gophish && git pull origin main && bash scripts/install.sh
systemctl restart gophish
```

Get the first-run password:

```bash
journalctl -u gophish -n 100 --no-pager | grep -i password
```

Username is always `admin`. Change it on first login.

## 3. HTTPS

```bash
certbot --nginx -d itsupport.insec.in -d admin.itsupport.insec.in
nginx -t && systemctl reload nginx
```

Open **https://admin.itsupport.insec.in**

When you create a campaign, set the **URL** to `https://itsupport.insec.in` (not the admin host). Email templates should use `{{.URL}}`.

## Useful commands

```bash
systemctl status gophish
journalctl -u gophish -f
systemctl restart gophish
```

## 4. Import email + landing templates

The repo includes original INSEC templates plus HailBytes, LinkSec, and piyush27pawar packs under `templates/`.

```bash
cd /opt/gophish && git pull origin main
apt-get install -y sqlite3 python3
export GOPHISH_API_KEY=$(sqlite3 /opt/gophish/runtime/gophish.db "SELECT api_key FROM users WHERE username='admin';")
python3 /opt/gophish/scripts/import-templates.py
```

Then in Gophish: pick an **Email Template** and a **Landing Page**, campaign URL `https://itsupport.insec.in`. Pair INSEC emails with **INSEC - IT Support login** or **INSEC - Education / you were phished**.

Import is English-only. If Spanish/Portuguese HailBytes items are already in Gophish:

```bash
cd /opt/gophish && git pull origin main
export GOPHISH_API_KEY=$(sqlite3 /opt/gophish/runtime/gophish.db "SELECT api_key FROM users WHERE username='admin';")
python3 /opt/gophish/scripts/keep-english-templates.py --gophish
```

See `templates/README.md`.

After a click or form submit, send people to **https://itsupport.insec.in/got-phished**.

Do **not** re-run `install.sh` for this — it would overwrite the live nginx file and drop certbot TLS. Patch the existing HTTP and HTTPS `itsupport.insec.in` servers:

```bash
cd /opt/gophish && git pull origin main
bash /opt/gophish/scripts/enable-got-phished.sh
```

Hard-refresh the URL after that. Set each landing page **Redirect to** `https://itsupport.insec.in/got-phished`.

## Campaign sending (SMTP)

Gophish needs an SMTP account (your mail server, SES, etc.). Configure it in the Gophish UI under **Sending Profiles**. This package does not send mail by itself.
