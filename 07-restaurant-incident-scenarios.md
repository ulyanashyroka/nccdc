# 07 Restaurant Incident Scenarios

**Source:** NECCDC Tryout Briefing, plus explicitly illustrative practice material. See the [complete guide](README.md).

## 9. End-to-end restaurant incidents

Each scenario mirrors the briefing's **plain-language owner emails**. The exact tasks are **invented practice exercises**, not leaked tryout questions. For each: state hypothesis, run read-only checks, identify the narrow authorized repair, and verify from a relevant client. The snippets emphasize diagnostics; repair steps should only be run after confirming the actual root cause.

### Scenario A — “The menu website is down” (Nginx vs Apache port conflict)

**Beginner** — `systemctl status nginx` reports `bind() ... 0.0.0.0:80 ... Address already in use`.

```bash
sudo ss -ltnp '( sport = :80 )'
sudo lsof -nP -iTCP:80 -sTCP:LISTEN
sudo apache2ctl -S                # if Apache is occupying port 80
sudo nginx -t
```

**Interpret:** another program has port 80. **Intermediate:** determine which website/scoring check Apache serves. **Advanced:** work with intended architecture (reverse proxy vs separate listeners); do not kill Apache blindly. **Verify:** correct Host-header requests and remote HTTP response.

### Scenario B — “Website only works when I sit at the server” (bind/firewall)

```bash
sudo ss -ltnp '( sport = :80 )'
curl -I http://127.0.0.1
curl -I http://192.168.56.10
sudo ufw status verbose
ip route
```

**Interpret:** if only `127.0.0.1:80` listens, investigate binding. If bound to accessible IP but remote TCP fails, inspect source-restricted firewall and route. **Verify from a client:** `curl -I http://192.168.56.10`.

### Scenario C — “Checkout fails, but other pages work” (PHP/database)

```bash
curl -i http://127.0.0.1/checkout.php
sudo tail -n 60 /var/log/nginx/error.log
systemctl list-units 'php*-fpm.service'
sudo ss -lxnp | grep php
sudo systemctl status mariadb
```

**Interpret:** differentiate 502 upstream/socket failures from DB login/permission errors. **Intermediate:** inspect the relevant app pool and app config without leaking secrets. **Advanced:** transaction errors vs database outage. **Verify:** intended checkout operation, not merely a static homepage.

### Scenario D — “All sessions vanish after reboot” (Redis durability)

```bash
sudo systemctl status redis-server
redis-cli INFO persistence
sudo journalctl -u redis-server -n 80 --no-pager
```

**Interpret:** determine whether persistent sessions were actually intended and how Redis stores data. **Verify:** harmless test session survives a controlled restart in an isolated lab.

### Scenario E — “Website certificate is invalid” (TLS/SAN)

```bash
openssl s_client -connect restaurant.test:443 -servername restaurant.test -verify_return_error </dev/null
openssl x509 -in /path/to/cert.pem -noout -dates -ext subjectAltName
```

**Interpret:** mismatch, expiration or chain. **Verify:** end-to-end hostname check from client, not just raw TCP 443.

### Scenario F — “Back-office worker cannot open the shared order folder” (SMB/NTFS)

```powershell
Get-SmbShareAccess -Name 'RestaurantDocs'
icacls 'C:\Shares\RestaurantDocs'
whoami /groups
```

**Interpret:** share vs NTFS restrictions, identity and inheritance. **Verify:** the specified user can perform the specified read/write operation, not just view the directory.

### Scenario G — “Former employee still has access” (AD identities)

```powershell
Get-ADUser jsmith -Properties Enabled,DistinguishedName
Get-ADPrincipalGroupMembership jsmith | Select-Object Name
Disable-ADAccount jsmith -WhatIf
```

**Interpret:** verify correct identity and scope, approvals and dependent services. **Verify:** disabled state and no unintended service-account disruption.

### Scenario H — “A staff member can't use Remote Desktop” (RDP)

```powershell
Get-Service TermService
Get-NetTCPConnection -LocalPort 3389 -State Listen
Get-LocalGroupMember 'Remote Desktop Users'
Get-NetFirewallRule -DisplayGroup 'Remote Desktop'
```

**Interpret:** service, listener, network, group and NLA/policy restrictions. **Verify:** actual RDP from permitted client with intended user.

### Scenario I — “The inventory report stopped running” (systemd timer / Windows scheduled task)

```bash
systemctl list-timers --all
systemctl status inventory-report.timer
journalctl -u inventory-report.service -n 50
```

```powershell
Get-ScheduledTask | Where-Object TaskName -Like '*inventory*'
Get-ScheduledTaskInfo -TaskName 'InventoryReport'
```

**Interpret:** missing schedule, failed job, wrong account, permissions or failed downstream database. **Verify:** report has correct new data and time stamp.

### Scenario J — “Server ran out of disk space” (disk/log rotation)

```bash
df -h; df -i
sudo du -xhd1 /var | sort -h
sudo lsof +L1
sudo logrotate -d /etc/logrotate.conf
```

**Interpret:** full filesystem, inode exhaustion, hidden space held by deleted/open file, or runaway logs. **Verify:** enough free space and intended app writes again. **Never delete unidentified database files.**

### Scenario K — “An unknown program returns after reboot” (Windows persistence)

```powershell
Get-ScheduledTask | Select-Object TaskName,State,TaskPath
Get-CimInstance Win32_Service | Select-Object Name,PathName,StartMode
Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Run'
Get-WinEvent -ListLog '*Sysmon*'
```

**Interpret:** correlate path, signer, task author and audit logs; normal management agents also persist. **Verify:** documented containment only after identifying an unauthorized item.

### Scenario L — “Nobody knows which machine hosts the website” (discovery)

```bash
ip -br addr
ip route
nmap -sn 192.168.56.0/24
nmap -sT -p 22,80,443 192.168.56.10
```

**Interpret:** map authorized subnet, discover hosts and HTTP endpoints, confirm intended target. **Verify:** hostname and browser content. Do not scan arbitrary campus/public networks.

---

