# Web Application Firewall Home Lab — SafeLine WAF vs. SQL Injection

A hands-on security lab that stands up a deliberately vulnerable web app, attacks it from Kali Linux, and puts a Web Application Firewall (SafeLine) in front to detect and block the attack. Built end-to-end on VirtualBox.

> **TL;DR:** Kali launches a SQL injection at DVWA. Without protection it dumps the database. With SafeLine WAF reverse-proxying the app, the same request is inspected, blocked, and logged.

---

## Architecture

```
   ┌──────────────┐        SQLi / HTTP flood         ┌──────────────────┐        reverse proxy        ┌────────────────────┐
   │   Kali VM    │  ───────────────────────────▶    │  SafeLine WAF    │  ───────────────────────▶   │  Ubuntu: DVWA      │
   │  (attacker)  │        https://dvwa.local        │  HTTPS :443      │        http://:8080         │  Apache + MySQL    │
   └──────────────┘                                  │  Admin  :9443    │                             └────────────────────┘
                                                      └──────────────────┘
```

All three hosts run as VirtualBox VMs on a single machine, on a bridged network so they share the host LAN.

## Tech stack

| Component | Role |
|---|---|
| VirtualBox | Hypervisor for all VMs |
| Kali Linux | Attacker workstation |
| Ubuntu Server | Hosts the vulnerable app + the WAF |
| Apache + PHP + MySQL (LAMP) | Serves DVWA on port 8080 |
| DVWA | Deliberately vulnerable web application |
| SafeLine WAF | Reverse-proxy Web Application Firewall (Docker-based) |

---

## What this lab demonstrates

- Standing up a vulnerable web app (DVWA) on a LAMP stack
- Performing a SQL injection attack from Kali (`1' OR '1'='1`)
- Deploying SafeLine WAF as a reverse proxy in front of the app
- Terminating HTTPS at the WAF with a self-signed certificate
- The WAF **detecting, blocking, and logging** the attack in real time
- Advanced controls: HTTP flood defense, auth gateway, custom IP deny rules

---

## Build summary

### 1. Environment
- Kali and Ubuntu Server VMs in VirtualBox, both on a **Bridged Adapter** so they sit on the host LAN.
- Ubuntu Server IP: `192.168.1.23` (example — yours will differ).

### 2. Vulnerable app (Ubuntu)
```bash
sudo apt-get install -y apache2 php php-mysql mysql-server git
cd /var/www/html && sudo git clone https://github.com/digininja/DVWA.git
sudo chown -R www-data:www-data DVWA && sudo chmod -R 755 DVWA
```
- DVWA DB created in MySQL, config set in `config/config.inc.php`.
- Apache moved to **port 8080** (`/etc/apache2/ports.conf`), so the WAF can own 443.
- Initialized at `http://<ubuntu-ip>:8080/DVWA/setup.php` → Create/Reset Database.

### 3. TLS certificate (Ubuntu)
```bash
sudo mkdir -p /etc/ssl/dvwa
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/dvwa/dvwa.key -out /etc/ssl/dvwa/dvwa.crt \
  -subj "/C=US/ST=Lab/L=Lab/O=HomeLab/CN=dvwa.local"
```

### 4. SafeLine WAF (Ubuntu)
```bash
bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en
```
- Admin console at `https://<ubuntu-ip>:9443`.
- Imported the self-signed cert.
- Onboarded DVWA as a site: domain `dvwa.local`, listen **:443 (HTTPS)**, upstream `http://<ubuntu-ip>:8080`.
- Pointed `dvwa.local` at the WAF in Kali's `/etc/hosts`.

### 5. Attack & defense
- From Kali, browsed to `https://dvwa.local/DVWA/`, set DVWA security to **Low**.
- Submitted SQL injection `1' OR '1'='1` on the SQL Injection page.
- **Result:** SafeLine returned a block page and logged the attempt (source IP, payload, matched rule) in **Events / Security Logs**.

---

## Results

| Scenario | Outcome |
|---|---|
| SQL injection, WAF in **detect-only** | Attack succeeds — DVWA dumps the user table |
| SQL injection, WAF in **block** mode | Request blocked at the WAF, app never sees it, event logged |

*(See `/screenshots` for the before/after and the WAF event log.)*

---

## Screenshots

| File | Shows |
|---|---|
| `Safeline-WAF-Homelab/waf-block-page.png` | The WAF block response (money shot) |
| `Safeline-WAF-Homelab/safeline-logs.png` | Blocked SQLi events with Kali source IP |
| `Safeline-WAF-Homelab/safeline-dashboard.png` | Unprotected dump vs. protected block |

---

## Troubleshooting notes (things that actually broke)

Real gotchas hit during this build — documented because they're the useful part:

- **VM lost its LAN IP after a host network change.** Bridged adapter pointed at a stale interface; only Docker bridges (`docker0`, `safeline-ce`) had IPs. Fixed by re-selecting the active host interface and renewing DHCP.
- **Kali pointed at a stale IP.** `/etc/hosts` still mapped `dvwa.local` to an old address; connections timed out. Updated to the current Ubuntu IP.
- **Port 443 collision.** A **Wazuh dashboard** (node process) was squatting on 443, blocking SafeLine from serving HTTPS. Stopped/disabled the Wazuh services to free the port (and RAM).
- **Firewall dropping 443.** `ufw` was active and only allowed 8443; port 443 from Kali was silently dropped (timeout, not "refused"). Fixed with `sudo ufw allow 443/tcp`. Note: Docker-published ports (9443) bypass ufw via Docker's iptables rules, which is why the admin console worked while 443 didn't.
- **"130 SQL injections from one input."** DVWA's SQLi form uses **GET**, so the payload lives in the URL and gets re-sent on every reload/refresh/retry — each replay is logged as a new blocked event. It's one attack, many replays.

---

## Next steps

- HTTP flood defense: rate-limit + ban, tested with `ab -n 2000 -c 50`
- Auth gateway in front of DVWA
- Custom deny rule blocking the Kali IP
- Additional attacks (XSS, file inclusion, command injection) and how the WAF responds
- Ship WAF logs into a SIEM (e.g. Wazuh) for detection engineering

---

## Credits

Based on the *Web Application Firewall Home Lab using SafeLine WAF* guide by Royden Rebello (The Social Dork) · [walkthrough video](https://youtu.be/N0dEC1nuWCQ).

> ⚠️ **Lab use only.** DVWA is intentionally vulnerable — keep this environment isolated from any production or internet-facing network.
