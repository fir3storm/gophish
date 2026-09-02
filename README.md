# Gophish for the AwareCheck VPS

Production wrapper around official **[Gophish](https://github.com/gophish/gophish) v0.12.1**. It is meant to run on the same Ubuntu VPS as AwareCheck without taking ports **80/443** (nginx) or **8000 / 8001 / 9000**.

Use this only for **authorized security-awareness simulations**.

## What was changed vs stock Gophish

| Stock default | This package |
|---------------|--------------|
| Admin TLS on `127.0.0.1:3333` (self-signed) | Admin HTTP on `127.0.0.1:3333`, TLS at nginx |
| Phish server on public `:80` | Phish HTTP on `127.0.0.1:8082` |
| No systemd / nginx | `gophish` user, systemd unit, nginx site |
| Admin CSRF origin empty | `trusted_origins` = `gophish.awarechck.com` |

Runtime files (binary, SQLite DB) live in `/opt/gophish/runtime` so git pulls never overwrite campaigns.

## VPS layout

| App | Domain | Path | Local port |
|-----|--------|------|------------|
| AwareCheck | `awarechck.com` | `/opt/awarecheck` | `8000` |
| Data Prahari | `dpdpsec.com` | `/opt/dpdpsec` | `8001` |
| Insec-Hydra | `hydra.insec.in` | `/opt/insec-hydra` | `9000` |
| **Gophish admin** | `gophish.awarechck.com` | `/opt/gophish/runtime` | `3333` |
| **Gophish campaigns** | `phish.awarechck.com` | same | `8082` |

## 1. DNS (do this first)

In the `awarechck.com` DNS panel add:

| Type | Name | Value |
|------|------|--------|
| A | `gophish` | `77.107.95.65` |
| A | `phish` | `77.107.95.65` |

Wait until `dig +short gophish.awarechck.com` returns `77.107.95.65`.

## 2. Install on the VPS

SSH in as root, then:

```bash
apt-get update && apt-get install -y git
git clone https://github.com/fir3storm/gophish.git /opt/gophish
bash /opt/gophish/scripts/install.sh
```

That downloads the official Linux binary (SHA-256 checked), creates user `gophish`, starts `gophish.service`, and enables nginx site `gophish`.

Get the first-run password:

```bash
journalctl -u gophish -n 100 --no-pager | grep -i password
```

Username is always `admin`. Change it on first login.

## 3. HTTPS

```bash
certbot --nginx -d gophish.awarechck.com -d phish.awarechck.com
sudo nginx -t && sudo systemctl reload nginx
```

Open **https://gophish.awarechck.com**

When you create a campaign, set the **URL** to `https://phish.awarechck.com` (not the admin host).

## Useful commands

```bash
systemctl status gophish
journalctl -u gophish -f
systemctl restart gophish
```

Re-run `bash /opt/gophish/scripts/install.sh` to replace the binary. Existing `config.json` and `gophish.db` are kept.

## Campaign sending (SMTP)

Gophish needs an SMTP account (your mail server, SES, etc.). Configure it in the Gophish UI under **Sending Profiles**. This package does not send mail by itself.
