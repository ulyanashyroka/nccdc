# NECCDC Tryout — Beginner to Advanced Defender Manual

> **Source:** `neccdc_tryout_briefing_v2(2).pptx`, especially slides 2, 5–10 and 12–14. The briefing outlines **possible study areas**, not confirmed tryout tasks. Command explanations, exercises, and lab examples in this manual are **additional study material**, not predictions of the tryout. Tested commands assume a Linux VM (often Ubuntu/Debian) and a Windows Server lab (with RSAT/AD tools where indicated).

## Table of contents

1. [What the tryout actually asks you to do](#1-what-the-tryout-actually-asks-you-to-do)
2. [Three levels of proficiency](#2-three-levels-of-proficiency)
3. [Foundations: connect, establish a baseline, and troubleshoot](#3-foundations-connect-establish-a-baseline-and-troubleshoot)
4. [Linux web and data stack](#4-linux-web-and-data-stack)
5. [Linux systems, security, and files](#5-linux-systems-security-and-files)
6. [Windows and Active Directory](#6-windows-and-active-directory)
7. [Windows: advanced security and identity](#7-windows-advanced-security-and-identity)
8. [Networking and discovering hosts](#8-networking-and-discovering-hosts)
9. [End-to-end restaurant incidents](#9-end-to-end-restaurant-incidents)
10. [Timed practice and readiness rubric](#10-timed-practice-and-readiness-rubric)
11. [Command index](#11-command-index)
12. [References and tryout rules](#12-references-and-tryout-rules)

## 1. What the tryout actually asks you to do

**From the briefing (slides 2, 5–8, 12–14):** You inherit a restaurant's IT environment. There are **two machines simultaneously**: a Linux server hosting the website (SSH), and a Windows back-office server (RDP). The owner sends plain-language requests describing symptoms, not solutions. Scoring is **automatic and objective**, with **partial credit**; some tasks contain a catch. The club explicitly says **no exploitation or attacking** during this tryout. The precise tasks are intentionally undisclosed.

The briefing states there are **3 main team places and 4 alternates**, and that selection also considers competence and learning speed. On the day, bring a laptop, charger, SSH and RDP clients. **AI assistants are not allowed**, and phones are collected; documentation and your own notes are encouraged. Check your organizer's latest instructions for any changes.

**What success looks like:** read every requirement, find evidence, change the smallest necessary thing, retest from the proper vantage point, and record what you did. Do not over-fix or spend the entire session on one task.

### A universal six-step workflow

1. **Restate the objective.** What exact behavior must the scorer see? Which server? Which users? Which port? Deadline?
2. **Baseline.** Record the current state before editing; note healthy services.
3. **Narrow down.** Check service → process/socket → local app response → remote network → DNS/permissions/dependencies.
4. **Back up relevant configuration.** Make a targeted change, preferably reversible.
5. **Validate and reload.** Run config tests where supported; avoid a blind reboot or broad firewall disable.
6. **Verify end to end.** Test the promised behavior as an actual user/client, then note evidence.

**Command safety labels in this manual:** `[READ]` inspect only; `[CHANGE]` alters configuration; `[DISRUPTIVE]` could interrupt service. Only modify your authorized lab/competition machines. Treat example hostnames, users and credentials as placeholders.

## 2. Three levels of proficiency

| Level | What you can demonstrate | Example |
| --- | --- | --- |
| **Beginner** | Connect safely, find settings, recognize service/process/network errors | Run `systemctl status nginx`, `Get-Service`, `ip route`, `Get-NetTCPConnection` |
| **Intermediate** | Form and test hypotheses, correct common breakages, verify users can reach a service | Find a bad web binding, DNS record, NTFS ACL, service dependency, or firewall rule |
| **Advanced / proficient** | Understand chains across systems, identity/authorization, logging, persistence, data durability; make minimal safe changes under time pressure | Distinguish broken PHP-FPM socket from Nginx failure; diagnose AD GPO vs NTFS share permissions; investigate suspicious Kerberos activity |

You do **not** need to be expert in every named technology to be a strong tryout candidate. The slides explicitly say the list is deliberately broad and some items may not appear.

## 3. Foundations: connect, establish a baseline, and troubleshoot

### 3.1 Connect to the two machines

From your laptop, SSH into Linux (`[READ]` connection):

```bash
ssh student@192.168.56.10
```

From Windows, RDP to the back-office server using `mstsc` (GUI), or the Remote Desktop client installed on your operating system. Never assume RDP is publicly exposed; it should be accessed only using the permitted lab network.

### 3.2 First 60 seconds — Linux baseline

```bash
whoami                           # current user
hostnamectl                      # host identity, OS details
pwd; ls -la                      # location, files and permissions
ip -br addr                      # interface/IP summary
ip route                         # default gateway and route prefixes
systemctl --failed               # failed systemd units
df -h                            # disk free space
sudo ss -ltnp                    # listening TCP sockets and processes
```

**When:** You get a mystery server, the restaurant site is unavailable, or you need to determine which dependencies are broken. `sudo` may be unavailable: use only authorized privileges.

### 3.3 First 60 seconds — Windows baseline

```powershell
hostname
whoami /all
Get-NetIPConfiguration
Get-Service | Where-Object Status -eq 'Stopped'
Get-NetTCPConnection -State Listen
Get-LocalUser
Get-NetConnectionProfile
Get-NetFirewallProfile -PolicyStore ActiveStore
```

**When:** You connect by RDP to an unfamiliar Windows Server. A stopped service is not automatically a problem; many are intentionally stopped. Don't start all of them.

### 3.4 A seven-layer hypothesis checklist

| Layer | Example failure | First checks |
| --- | --- | --- |
| Name | `restaurant.test` points at old IP | `dig`, `Resolve-DnsName` |
| Route | wrong gateway or missing return route | `ip route`, `Find-NetRoute`, `tracert` |
| Firewall | TCP allowed on localhost but denied remotely | `ufw status`, `nft list ruleset`, `Get-NetFirewallRule` |
| Listener | binds only to `127.0.0.1` | `ss -ltnp`, `Get-NetTCPConnection` |
| Service | process failed, wrong unit or dependency | `systemctl status`, `Get-Service`, `journalctl` |
| App | HTTP 500/502/503, incorrect vhost or backend | `curl -I`, web logs, config tests |
| Identity/data | permission denied, database login rejected, share access fails | `ls -l`, `namei -l`, database roles, ACLs |

**Rule:** A working `ping` is not proof TCP 80/443 works. A successful TCP connection is not proof that the website returns the right content. A local HTTP 200 is not proof that users on another subnet can access it.

---

## 4. Linux web and data stack

These eight entries correspond to the briefing's **Linux → Web and data** slide. Paths differ by distribution, packages and custom deployment; inspect before changing.

### 4.1 Apache httpd — virtual hosts, modules, configtest

**Beginner:** Apache serves HTTP requests, often multiple sites via *virtual hosts* (hostnames/ports). On Ubuntu the service is typically `apache2`; on RHEL-derived systems it is typically `httpd`.

**When the owner says:** “One restaurant website works but our reservations site shows the wrong page.”

```bash
sudo systemctl status apache2                # [READ] Ubuntu service
sudo apache2ctl -S                           # [READ] vhost mapping
sudo apache2ctl -M                           # [READ] loaded modules
sudo apache2ctl configtest                   # [READ] configuration test
sudo tail -n 60 /var/log/apache2/error.log  # [READ] errors, default Ubuntu path
curl -I -H 'Host: reservations.test' http://127.0.0.1/
```

**Interpretation:** `Syntax OK` checks syntax, not that the correct site is bound, paths exist, or the application works. A wrong default vhost can display the wrong website.

**Intermediate lab repair** (edit a dedicated lab vhost after saving a copy; sample only):

```apache
<VirtualHost *:80>
    ServerName reservations.test
    DocumentRoot /var/www/reservations
    ErrorLog ${APACHE_LOG_DIR}/reservations-error.log
</VirtualHost>
```

Run `sudo apache2ctl configtest`, then `sudo systemctl reload apache2` **[CHANGE]**, and verify using a matching Host header. **Advanced:** trace Apache → PHP-FPM or reverse proxy; distinguish wrong vhost selection from app failures, and avoid stopping another required listener.

### 4.2 Nginx — server blocks, redirects, `nginx -t`

**Beginner:** Nginx has `server {}` blocks, `location {}` matching, and upstream proxying. `301/302` are redirects; `502` often signals a bad upstream/backend connection.

**Scenario:** “The homepage works, but `/menu` returns 502.”

```bash
sudo systemctl status nginx
sudo nginx -t
sudo nginx -T | less                 # inspect effective loaded config (may expose secrets)
sudo journalctl -u nginx -n 40 --no-pager
sudo tail -n 60 /var/log/nginx/error.log
curl -i http://127.0.0.1/menu
sudo ss -ltnp '( sport = :80 )'
```

Sample proxy:

```nginx
server {
    listen 80;
    server_name restaurant.test;
    location /api/ {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

**Intermediate:** Check that the backend actually listens on 8080 before changing Nginx. **Advanced:** distinguish Nginx redirect loops, `proxy_pass` URI path rewriting, TLS termination, and backend health. Before applying edited config: `sudo nginx -t && sudo systemctl reload nginx` **[CHANGE]**. Recheck the website afterward.

### 4.3 HAProxy — load balancing and health checks

**Beginner:** HAProxy distributes client traffic among backend servers; a backend marked DOWN can yield a 503 or remove capacity.

**Scenario:** “Website is available sometimes, but requests fail every other time.”

```bash
sudo systemctl status haproxy
sudo haproxy -c -f /etc/haproxy/haproxy.cfg   # check config without reloading
sudo journalctl -u haproxy -n 60 --no-pager
sudo ss -ltnp | grep -E ':80|:443|:8080'
curl -I http://127.0.0.1:8080/health        # test backend locally
```

Example concept:

```haproxy
backend restaurant_web
    balance roundrobin
    option httpchk GET /health
    server app1 192.168.56.21:8080 check
    server app2 192.168.56.22:8080 check
```

**Intermediate:** Match health-check URL to actual backend behavior; check whether just one server fails. **Advanced:** distinguish L4 versus L7 health checks, TLS termination, sticky sessions, and shared database dependencies.

### 4.4 PHP-FPM — pools, sockets and logs

**Beginner:** Nginx/Apache can hand PHP requests to PHP-FPM, often via a Unix socket. A web server returning 502 can be healthy while PHP-FPM is down.

**Scenario:** “Static HTML loads, but checkout.php fails.”

```bash
systemctl list-units 'php*-fpm.service'
sudo systemctl status php8.3-fpm           # example version only
sudo journalctl -u php8.3-fpm -n 50        # version varies
sudo ss -lxnp | grep php                    # Unix domain sockets
ls -l /run/php/
grep -R 'fastcgi_pass\|listen =' /etc/nginx /etc/php 2>/dev/null
```

Illustrative mismatch: Nginx expects `/run/php/php8.3-fpm.sock`, while the PHP-FPM pool actually listens at `/run/php/php8.2-fpm.sock`. **Intermediate:** correct the *intended* socket reference after confirming versions/ownership and testing configuration. **Advanced:** pool user permissions, socket ACLs, max worker exhaustion, slow logs, and PHP application errors.

### 4.5 PostgreSQL — roles, `pg_hba.conf`, `pg_dump`

**Beginner:** PostgreSQL roles have privileges. `pg_hba.conf` determines which identities may authenticate from which clients using which method. `pg_dump` backs up database contents (not a complete cluster-level backup).

**Scenario:** “The website loads, but says it cannot log in to the reservations database.”

```bash
sudo systemctl status postgresql
sudo -u postgres psql -c '\du'            # inspect roles (admin access needed)
sudo -u postgres psql -c 'SHOW hba_file;'
sudo -u postgres psql -c 'SHOW listen_addresses;'
sudo ss -ltnp '( sport = :5432 )'
sudo journalctl -u postgresql -n 60 --no-pager
```

Example `pg_hba.conf` concept (not a default policy recommendation):

```text
# TYPE  DATABASE      USER         ADDRESS           METHOD
host    reservations  app_reader   192.168.56.0/24   scram-sha-256
```

**Intermediate:** Compare host/user/database to application connection string; beware `pg_hba.conf` first-match behavior. Test local and remote access separately. Backup example in an authorized lab:

```bash
pg_dump -h 127.0.0.1 -U backup_user -Fc reservations > reservations.dump
```

**Advanced:** roles vs object grants, transaction-level errors, TLS, replication, backups/restores, and avoiding overbroad `trust` authentication. Do not dump sensitive production data into a Git repository.

### 4.6 MariaDB — users, `GRANT`, `mysqldump`

**Beginner:** MySQL/MariaDB accounts can depend on BOTH username and allowed host (e.g., `'app'@'localhost'`). Permissions belong on specific schemas/tables.

**Scenario:** “The inventory app says Access denied for user.”

```bash
sudo systemctl status mariadb
sudo mariadb -e "SELECT User,Host FROM mysql.user;"       # requires admin privileges
sudo mariadb -e "SHOW GRANTS FOR 'app'@'localhost';"
sudo ss -ltnp '( sport = :3306 )'
sudo journalctl -u mariadb -n 50 --no-pager
```

**Intermediate:** Compare database username/host with exact error; grant *only necessary privileges*, not blanket `ALL` or remote `%` as a quick fix. Backup example:

```bash
mysqldump -u backup_user -p --single-transaction inventory > inventory.sql
```

**Advanced:** authentication plugins, storage engines, transactionally consistent backups, socket vs TCP connections, and data recovery verification.

### 4.7 Redis — persistence and protected mode

**Beginner:** Redis is commonly a cache or key-value store. A down cache may slow or break an app. Exposing Redis openly can be dangerous.

**Scenario:** “The ordering page randomly loses session state after a reboot.”

```bash
sudo systemctl status redis-server
redis-cli PING                                # local test; returns PONG if accessible
redis-cli INFO persistence                   # check RDB/AOF status
sudo ss -ltnp '( sport = :6379 )'
sudo journalctl -u redis-server -n 50 --no-pager
```

**Intermediate:** Diagnose whether the app expected persistence (`appendonly`, snapshots) or caching only; check `bind`, protected mode, and auth configuration. **Advanced:** understand snapshot vs append-only file, durability tradeoffs, maxmemory eviction, replication and safe recovery. Never disable protected mode merely to make Redis reachable.

### 4.8 OpenSSL — certificates, SANs, certificate authorities and `s_client`

**Beginner:** TLS certificates link a server's identity to a public key. The certificate's **Subject Alternative Name (SAN)** must match the hostname; a Certificate Authority (CA) helps establish trust.

**Scenario:** “Customers receive a certificate warning on HTTPS.”

```bash
openssl s_client -connect restaurant.test:443 -servername restaurant.test -showcerts </dev/null
openssl s_client -connect restaurant.test:443 -servername restaurant.test -verify_return_error </dev/null
openssl x509 -in /path/to/cert.pem -noout -dates -subject -issuer -ext subjectAltName
```

**Intermediate:** Distinguish expiration, hostname mismatch, missing intermediates, and untrusted CA; `-servername` supplies SNI. **Advanced:** investigate chain ordering, server/client TLS configuration, renewal processes, and test an endpoint externally after a certificate change.

---

## 5. Linux systems, security, and files

These eight entries cover the briefing's **Linux → System and security** list.

### 5.1 SSH and fail2ban — keys, brute force

**Scenario:** “Our administrator can't log in, but SSH works for another employee.”

```bash
sudo systemctl status ssh                   # Ubuntu; some distros use sshd
sudo ss -ltnp '( sport = :22 )'
sudo journalctl -u ssh -n 80 --no-pager
sudo sshd -t                               # syntax test; no restart
ls -ld ~/.ssh && ls -l ~/.ssh/authorized_keys
sudo fail2ban-client status
sudo fail2ban-client status sshd            # if jail exists
```

**Beginner:** public/private keys, `authorized_keys`, `sshd_config`, SSH port 22, password vs key auth. **Intermediate:** home-folder ownership/permissions, locked users, fail2ban accidental bans, disabled key authentication. **Advanced:** distinguish server authentication failure from account access policy, prioritize preserving your own administrative access, and avoid disabling security settings broadly.

### 5.2 systemd — units, timers, journalctl

**Scenario:** “A report that should regenerate every hour has stopped updating.”

```bash
systemctl --failed
systemctl status restaurant-report.service
systemctl list-timers --all
systemctl cat restaurant-report.service
systemctl cat restaurant-report.timer
journalctl -u restaurant-report.service --since '1 hour ago' --no-pager
```

**Beginner:** `.service` units manage processes, `.timer` units schedule work. **Intermediate:** `After=` is ordering, not necessarily a dependency; check `Requires=`, `Wants=`, environment, working directory and user. **Advanced:** analyze timer persistence, permissions, unit overrides, and failure restart policies.

**Safe change sequence after backup**: `systemctl daemon-reload` **[CHANGE]** after editing unit files, then restart only the affected unit **[DISRUPTIVE]**, and verify with `status` and `journalctl`.

### 5.3 Samba and NFS — Linux file shares

**Scenario:** “The Windows cashier can't open the shared menu folder.”

```bash
sudo systemctl status smbd               # Samba service on Ubuntu
sudo testparm -s                         # validate effective Samba config
sudo smbstatus                           # connections and locks
ls -ld /srv/restaurant-share
namei -l /srv/restaurant-share
showmount -e 127.0.0.1                   # NFS export list if RPC service exists
sudo exportfs -v                         # inspect exported NFS directories
```

**Beginner:** Samba provides SMB file sharing, often for Windows clients; NFS is commonly used between Unix/Linux hosts. **Intermediate:** distinguish share-level permissions from underlying Linux filesystem ACLs, NFS export restrictions, client identity mapping and path access. **Advanced:** diagnose SMB dialect compatibility, `root_squash`, stale mounts, and file locking without globally opening the share.

### 5.4 `ss` and `lsof` — ports and open files

**Scenario:** “Nginx can't bind to port 80.”

```bash
sudo ss -ltnp '( sport = :80 )'
sudo lsof -nP -iTCP:80 -sTCP:LISTEN
ps -fp 1826                               # example PID from above
```

`ss` shows sockets and owning processes. `lsof` can link open files/sockets to processes; `-nP` skips DNS/service-name resolution. **Intermediate:** decide whether the other service is required before stopping it. **Advanced:** inspect Unix sockets, inherited FDs, listening on `127.0.0.1` vs `0.0.0.0`, and multiple listeners.

### 5.5 AppArmor and SELinux — mandatory access control (MAC)

**Scenario:** “Unix permissions look correct but the web app still gets Permission denied.”

```bash
sudo aa-status                             # AppArmor-enabled machines
sudo journalctl -k -n 100 --no-pager | grep -i apparmor
getenforce                                 # SELinux machines, if installed
sudo ausearch -m AVC -ts recent            # SELinux denial events; audit tools needed
ls -Z /var/www/html                        # inspect SELinux labels if supported
```

**Beginner:** discretionary access control (DAC: owner/group/mode) differs from mandatory controls applied by security policy. **Intermediate:** identify the exact denied path and policy before changing anything. **Advanced:** create targeted policy/label corrections on a lab host; **do not** globally disable AppArmor or SELinux as a first response.

### 5.6 `ufw` and `nftables` — host firewall

**Scenario:** “The site returns 200 on the server but remote clients time out.”

```bash
sudo ufw status verbose
sudo ufw status numbered
sudo nft list ruleset
sudo ss -ltnp '( sport = :80 )'
curl -I http://127.0.0.1
```

**Beginner:** host firewall allows or blocks traffic; forwarding and return paths may matter too. **Intermediate:** evaluate interface, source subnet, protocol, destination port and rule order. A narrowly scoped Ubuntu lab allow rule might be:

```bash
sudo ufw allow from 192.168.56.0/24 to any port 80 proto tcp  # [CHANGE]
```

**Advanced:** trace packet flow and rule evaluation in nftables; avoid overwriting competition firewall policy or locking out SSH.

### 5.7 Docker — containers, logs and volumes

**Scenario:** “The menu page fails after a server restart.”

```bash
docker ps -a
sudo docker logs --tail 80 menu-app
sudo docker inspect menu-app --format '{{.State.Status}} {{.State.ExitCode}}'
sudo docker inspect menu-app --format '{{json .Mounts}}'
sudo docker exec menu-app sh -c 'id; pwd'  # only when container is running
```

**Beginner:** image vs container, ports, persistent volume, logs. **Intermediate:** distinguish a crashed application from missing mounted data or a bad port mapping. **Advanced:** container health checks, volume ownership, restart policies, networking and secret handling. Avoid `docker system prune`, deleting volumes, or rebuilding containers before establishing data safety.

### 5.8 `df`, `du`, `logrotate` — disk space and logs

**Scenario:** “The restaurant website stopped saving orders and PostgreSQL won't start.”

```bash
df -hT
sudo du -xhd1 /var | sort -h
sudo journalctl --disk-usage
sudo logrotate -d /etc/logrotate.conf    # dry-run, inspect; may warn on permissions
sudo find /var/log -type f -size +100M -print
```

**Beginner:** `df` shows space used/free for mounted filesystems; `du` shows estimated sizes under paths. **Intermediate:** investigate full partitions or inodes (`df -i`), identify a specific log or data growth source. **Advanced:** log retention and rotation rules; diagnose files that were deleted but remain open (`sudo lsof +L1`). Never truncate database files or delete unknown logs to make room.

---

## 6. Windows and Active Directory

The briefing asks you to **learn the GUI once, then repeat in PowerShell**. Most commands below require domain access, correct modules, and appropriate privileges. On a standalone Windows machine, AD-specific cmdlets will not work.

### 6.1 AD PowerShell — users, groups, disable, move

**Beginner:** Active Directory centrally manages domain identities. An organizational unit (OU) is an administrative container; groups grant access.

**Scenario:** “A departed employee can still log in. Check account status and group memberships.”

```powershell
Import-Module ActiveDirectory
Get-ADUser -Identity jsmith -Properties Enabled,LastLogonDate,DistinguishedName |
  Select-Object Name,Enabled,LastLogonDate,DistinguishedName
Get-ADPrincipalGroupMembership -Identity jsmith | Select-Object Name
Get-ADGroupMember -Identity 'Restaurant-Staff' | Select-Object Name,ObjectClass
```

**Intermediate:** find the correct account and confirm written authorization before disabling it:

```powershell
Disable-ADAccount -Identity jsmith -WhatIf          # simulate action
# Disable-ADAccount -Identity jsmith                 # [CHANGE] only after approval
```

To preview moving an account to another OU:

```powershell
Get-ADUser jsmith | Move-ADObject -TargetPath 'OU=FormerStaff,DC=restaurant,DC=lab' -WhatIf
```

**Advanced:** distinguish local users from domain users, group nesting, inherited access, and the impact of moving an object to a new OU with different GPOs.

### 6.2 Privileged groups — who should administer what?

**Scenario:** “A normal cashier appears to have administrative permissions.”

```powershell
Get-ADGroupMember -Identity 'Domain Admins' -Recursive |
  Select-Object Name,SamAccountName,ObjectClass
Get-LocalGroupMember -Group Administrators       # on member server
whoami /groups
```

**Beginner:** Domain Admins are broadly privileged in the domain; local Administrators control a particular server/workstation. **Intermediate:** verify whether access was intentional, identify membership path, and apply least privilege after authorization. **Advanced:** understand tiering, privileged service accounts, nested groups, and auditing membership changes.

### 6.3 Group Policy — `gpresult`, `gpupdate`, applied GPOs

**Scenario:** “A password policy or security setting is different from what IT expects.”

```powershell
gpresult /r
gpresult /scope computer /v
gpresult /h "$env:TEMP\gp-report.html"
Get-GPInheritance -Target 'OU=Cashiers,DC=restaurant,DC=lab'   # GroupPolicy module
Get-GPResultantSetOfPolicy -ReportType Html -Path 'C:\Temp\rsop.html'
```

**Beginner:** GPOs can apply at site/domain/OU levels, with scope/security filtering and inheritance. **Intermediate:** use results to identify which policy actually applied; understand that a configured GPO might be filtered out. **Advanced:** inspect enforcement, loopback processing, WMI filters, conflicts and refresh failures. `gpupdate /force` **[CHANGE]** reprocesses policy and can affect the system; do not use it to cover up a policy misunderstanding.

### 6.4 Shares and NTFS — both permission layers

**Scenario:** “A user can access the share but cannot modify a file.”

```powershell
Get-SmbShare
Get-SmbShareAccess -Name 'RestaurantDocs'
Get-Acl 'C:\Shares\RestaurantDocs' | Format-List
icacls 'C:\Shares\RestaurantDocs'
Get-LocalGroupMember -Group 'Users'
```

**Beginner:** SMB share permissions and NTFS permissions are separate restrictions; effective network access must satisfy both. **Intermediate:** inspect inherited versus explicit ACEs and group membership. **Advanced:** evaluate effective rights, explicit deny, ownership, inheritance, auditing and SMB session identity. Example preview of granting a lab group a file right:

```powershell
icacls 'C:\Shares\RestaurantDocs' /grant 'RESTAURANT\Cashiers:(OI)(CI)M' /T
```

**[CHANGE]** This recursively grants Modify; only perform with verified requirements, a tested path, and a backup. Prefer minimal scope over `/T` when possible.

### 6.5 Remote Desktop — allowed users, NLA, firewall scope

**Scenario:** “The office manager cannot connect to the back-office server via RDP.”

```powershell
Get-Service TermService
Get-NetTCPConnection -State Listen -LocalPort 3389
Get-LocalGroupMember -Group 'Remote Desktop Users'
Get-NetFirewallRule -DisplayGroup 'Remote Desktop' |
  Select-Object DisplayName,Enabled,Direction,Action,Profile
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' |
  Select-Object fDenyTSConnections
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' |
  Select-Object UserAuthentication
```

**Beginner:** `fDenyTSConnections=1` generally disables incoming RDP; `UserAuthentication=1` commonly indicates Network Level Authentication (NLA) is required. **Intermediate:** distinguish RDP service, listener, network, group membership and NLA/authentication failure. **Advanced:** examine user-right assignments, policy overrides, gateway infrastructure and narrow allowed source networks. Do not disable NLA or expose RDP broadly just to make a test connect.

### 6.6 Autoruns — scheduled tasks, services, Run keys

**Scenario:** “A mystery program starts again after every reboot.”

```powershell
Get-ScheduledTask | Select-Object TaskName,TaskPath,State
Get-CimInstance Win32_Service |
  Select-Object Name,State,StartMode,PathName
Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Run'
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run'
```

**Beginner:** persistence can come from legitimate scheduled tasks, services or registry Run keys. **Intermediate:** inspect execution paths, authors, timestamps, digital signatures and related logs; compare with approved baseline. **Advanced:** use Microsoft's Autoruns tool to review additional autostart locations, then contain and document verified malicious persistence without deleting unknown business tasks.

---

## 7. Windows: advanced security and identity

The briefing marks these **also worth knowing**, not promised hands-on topics. For tryout preparation, prioritize **recognition, evidence and defense**, not performing offensive attacks. All commands here are defensive inspection/auditing commands.

### 7.1 Domain passwords, fine-grained password policies, lockouts

**Scenario:** “The owner says the office manager keeps getting locked out.”

```powershell
Get-ADDefaultDomainPasswordPolicy
Get-ADFineGrainedPasswordPolicy -Filter * |
  Select-Object Name,Precedence,MinPasswordLength,LockoutThreshold
Get-ADUserResultantPasswordPolicy -Identity manager
Get-ADUser manager -Properties LockedOut,BadPwdCount,LastBadPasswordAttempt |
  Select-Object Name,LockedOut,BadPwdCount,LastBadPasswordAttempt
```

**Beginner:** policy can apply domain-wide or be fine-grained; lockouts may indicate mistyped passwords, stale saved credentials, or hostile attempts. **Intermediate:** inspect domain-controller security logs (e.g. lockout Event ID 4740), affected workstation, time, and recent changes. **Advanced:** distinguish replication and DC attribution nuances, authentication sources and delegated policy scope. Never reflexively disable lockout controls.

### 7.2 Kerberoasting, AS-REP roasting, Golden Ticket — defender overview

- **Kerberoasting:** abuse of service-ticket material associated with service accounts, particularly weak account secrets. **Defenses:** strong managed service account credentials, minimum privileges, review SPNs, monitor anomalous ticket requests (for example, relevant 4769 events).
- **AS-REP roasting:** risk when Kerberos preauthentication is disabled for an account. **Defenses:** inspect account flags, restore preauthentication where appropriate, strong secrets, and monitor relevant Kerberos authentication events.
- **Golden Ticket:** forging Kerberos ticket-granting tickets after compromise of critical domain secrets, especially the `krbtgt` secret. **Defenses:** protect DCs and privileged identities, monitor ticket anomalies, follow established incident-response procedure for `krbtgt` rotation.

Read-only triage:

```powershell
Get-ADUser -Filter * -Properties DoesNotRequirePreAuth |
  Where-Object DoesNotRequirePreAuth |
  Select-Object Name,SamAccountName
Get-ADUser -LDAPFilter '(servicePrincipalName=*)' -Properties ServicePrincipalName |
  Select-Object SamAccountName,ServicePrincipalName
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4768,4769; StartTime=(Get-Date).AddHours(-2)} |
  Select-Object -First 20 TimeCreated,Id,Message
```

Events must be enabled/available and interpreted in context; a 4769 event alone is not an attack indicator.

### 7.3 DCSync and Pass-the-Hash (PtH) — defender overview

- **DCSync:** misuse of directory-replication privileges to obtain sensitive AD credential data. **Defend:** restrict replication rights, monitor unexpected directory replication requests (e.g. event 4662 when appropriately audited), protect domain controller access.
- **Pass-the-Hash:** using stolen NTLM credential material for authentication without knowing the plaintext password. **Defend:** credential isolation, LAPS for local admin secrets, reduce NTLM exposure where feasible, logon restrictions, monitor suspicious authentication patterns (e.g. 4624/4776 in context).

Read-only examples:

```powershell
Get-ADGroupMember 'Domain Admins' -Recursive
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624,4776; StartTime=(Get-Date).AddHours(-1)} |
  Select-Object -First 20 TimeCreated,Id,Message
```

**Advanced proficiency:** know which permissions or secrets make these techniques possible and what normal versus unusual admin activity looks like; you do not need to run an attack to defend against one.

### 7.4 PingCastle — AD posture audit

**Scenario:** “The restaurant owner asks whether their AD domain has dangerous defaults.”

**Beginner:** PingCastle is a third-party AD security auditing tool; it reports risky configurations and privileges. **Intermediate:** review findings and corroborate high-risk results using native tools before changes. **Advanced:** prioritize risk reduction without disrupting login or critical service accounts. Use only an authorized, approved version and only in your isolated lab or permitted competition scope. Do not assume its findings are automatically exploitable or equally urgent.

### 7.5 LAPS and Sysmon — local admin secrets and richer logs

**LAPS:** Windows Local Administrator Password Solution helps manage unique local administrator passwords.

**Sysmon:** System Monitor provides detailed telemetry such as process creation, (where enabled) network connections, registry changes and DNS activity.

**Scenario:** “We suspect an unauthorized tool launched on the Windows server.”

```powershell
Get-WinEvent -ListLog '*Sysmon*'
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-Sysmon/Operational'; Id=1;
  StartTime=(Get-Date).AddHours(-2)
} -MaxEvents 30 | Select-Object TimeCreated,Id,Message
Get-WinEvent -ListLog '*LAPS*'
```

**Beginner:** these logs require the relevant feature to be present and configured. **Intermediate:** correlate executable path, parent process, command line, account and timestamp. **Advanced:** develop baselines and targeted detections; protect sensitive data in logs and avoid printing managed passwords into documentation.

### 7.6 ADFS and Keycloak — SSO, SAML, OIDC

**Beginner:** identity providers support single sign-on (SSO). SAML often uses XML assertions; OpenID Connect (OIDC) builds an identity layer on OAuth 2.0 and commonly uses ID tokens.

**Scenario:** “Staff can log in to Windows but cannot access the staff portal via SSO.”

```powershell
Get-Service *adfs*                              # if AD FS installed
Get-WinEvent -ListLog '*AD FS*'                 # discover available logs
Test-NetConnection sso.restaurant.test -Port 443
Resolve-DnsName sso.restaurant.test
```

For Keycloak in a containerized lab:

```bash
docker ps --format '{{.Names}} {{.Status}}'
sudo docker logs --tail 80 keycloak
curl -I https://sso.restaurant.test/
```

**Intermediate:** check DNS/TLS, identity provider health, client redirect URI, issuer, certificate and time synchronization. **Advanced:** differentiate token validation, trust metadata, SAML audience/claims and OIDC redirect errors. Do not disable TLS checks as a shortcut.

---

## 8. Networking and discovering hosts

The briefing explicitly notes you may **not be given IP addresses** and recommends learning Nmap. Only scan the **authorized tryout or practice subnet**.

### 8.1 Identify your own network first

Linux:

```bash
ip -br addr
ip route
ip neigh
resolvectl status                  # if systemd-resolved is used
```

Windows:

```powershell
Get-NetIPConfiguration
Get-NetRoute -AddressFamily IPv4
arp -a
Get-DnsClientServerAddress -AddressFamily IPv4
```

### 8.2 Discover hosts and ports in your authorized lab

```bash
nmap -sn 192.168.56.0/24          # host discovery; some hosts may block probes
nmap -sT -p 22,80,443,3389 192.168.56.10
nmap -sT -sV -p 22,80,443 192.168.56.10
```

**Interpretation:** `open` indicates a response consistent with an accessible service; `closed` indicates reachable but not listening; `filtered` suggests a probe did not yield enough evidence, often due to filtering. Scans do not automatically prove which host is the intended restaurant server. Combine with hostname, SSH banner and owner documentation.

### 8.3 Separate DNS, TCP and HTTP symptoms

```bash
dig restaurant.test A
curl -I --connect-timeout 5 http://192.168.56.10/
curl -I -H 'Host: restaurant.test' http://192.168.56.10/
```

```powershell
Resolve-DnsName restaurant.test -Type A
Test-NetConnection 192.168.56.10 -Port 80
Test-NetConnection restaurant.test -Port 443
```

**Advanced:** vhost and TLS SNI can make IP-only requests misleading. Use the correct hostname and authorized route; test from both the server and an actual client.

---

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

## 10. Timed practice and readiness rubric

### 4-hour emergency preparation order

| Time | Focus | Concrete outcome |
| --- | --- | --- |
| 0:00–0:30 | Both OSes, IPs, service/port basics | Know which machine and which listener you're touching |
| 0:30–1:30 | Linux: Nginx/Apache/systemd/logs | Solve web server outage and port conflict |
| 1:30–2:10 | Windows: users/groups/GPO/SMB/RDP | Diagnose permission or domain access issue |
| 2:10–2:50 | Network: DNS, routing, firewall, TLS | Diagnose local-works/remote-fails |
| 2:50–3:25 | Linux: PHP-FPM/database/Docker/disk | Recognize dependency failures |
| 3:25–4:00 | Timed simulated injects | Complete at least 2 tasks fully, document verification |

### 10-question self-quiz (answer without opening the next section)

1. `nginx -t` passes but the site returns 502. Which layer do you inspect next?
2. A listener appears at `127.0.0.1:80`, not `0.0.0.0:80`. Why might remote clients fail?
3. `df -h` shows free space, but writes fail. Which second capacity metric matters?
4. The SMB share allows Change, NTFS denies Write. Can the user write?
5. An AD account is disabled but a service stops working. What should you investigate?
6. An RDP connection reaches TCP 3389 but is refused at sign-in. What identity/policy checks matter?
7. A user's policy differs from an OU-linked GPO. What tool shows applied policy?
8. Redis responds to `PING` but session data disappears after restart. Which concept matters?
9. Nginx and PHP-FPM both show running, but PHP routes fail. What inter-service configuration could be wrong?
10. A port is filtered by Nmap. Does that prove the service is absent?

<details>
<summary>Suggested answers — open only after practicing</summary>

1. Upstream service, backend address/socket, PHP-FPM or app logs.
2. Loopback is local to the server; remote clients target its network IP.
3. Inodes (`df -i`); also consider quotas and filesystem states.
4. No. Both layers must allow the requested action.
5. Whether the account was used by a service, scheduled task or app; confirm desired disabling and service dependency.
6. Group rights, domain vs local credentials, NLA, logon rights and security logs.
7. `gpresult /r` or `gpresult /h`.
8. Persistence mechanisms, whether cache-only behavior was intended, and configuration/volume durability.
9. `fastcgi_pass` socket/path mismatch or Unix socket permissions.
10. No. Filtering/inconclusive probes don't prove absence.

</details>

### Progress tracker

- [ ] I can connect to both systems and map IPs/routes without guessing.
- [ ] I can diagnose a listener/service/port problem on Linux.
- [ ] I can validate Apache and Nginx configs before reloading.
- [ ] I can trace browser → web server → PHP/backend → database.
- [ ] I can inspect TLS hostname and certificate problems.
- [ ] I can distinguish Linux DAC from MAC denials.
- [ ] I can inspect Docker state and volumes without deleting data.
- [ ] I can inspect Windows accounts, domain groups and GPO application.
- [ ] I understand both SMB share and NTFS permission layers.
- [ ] I can investigate RDP, scheduled tasks and Windows logs.
- [ ] I can identify common AD security concepts without launching attacks.
- [ ] I can complete a practice inject, verify it from a client, and leave an audit trail.

### Inject note template

```text
Task / requirement:
System(s):
Expected result:
Initial symptom:
Evidence collected (commands + outputs):
Hypothesis:
Change performed (and configuration backup):
Verification from intended client/user:
Remaining risks / dependencies:
Completed at:
```

## 11. Command index

| Goal | Linux | Windows/PowerShell |
| --- | --- | --- |
| Identity | `whoami`, `id` | `whoami /all`, `Get-ADUser` |
| IP and routing | `ip addr`, `ip route` | `Get-NetIPConfiguration`, `Get-NetRoute` |
| Port listeners | `ss -ltnp`, `lsof` | `Get-NetTCPConnection` |
| Services | `systemctl status` | `Get-Service` |
| System logs | `journalctl` | `Get-WinEvent` |
| Host firewall | `ufw status`, `nft list ruleset` | `Get-NetFirewallProfile`, `Get-NetFirewallRule` |
| DNS | `dig`, `resolvectl` | `Resolve-DnsName` |
| Connectivity | `curl`, `ssh`, `nc` | `Test-NetConnection`, `curl.exe` |
| Web syntax | `nginx -t`, `apache2ctl configtest` | IIS `Get-Website`, `Get-WebBinding` (if WebAdministration installed) |
| User/group | `getent passwd`, `id`, `groups` | `Get-LocalUser`, `Get-ADGroupMember` |
| Storage | `df`, `du`, `lsof +L1` | `Get-Volume`, `Get-PSDrive` |

## 12. References and tryout rules

**Primary source:** Uploaded *NECCDC Tryout Briefing*, slides 2, 5–10 and 12–14. Its official scope is **deliberately broad**; it does not identify guaranteed tasks. The restaurant incidents and commands in this manual are **author-created expansions**.

Recommended resources identified in the briefing:

- [NECCDC 2027 history](https://neccdl.org/history/2027/)
- [NECCDC 2026 history](https://neccdl.org/history/2026/)
- [NECCDL public GitHub repositories](https://github.com/NECCDL)
- [OverTheWire Bandit](https://overthewire.org/wargames/bandit/)
- [Under the Wire](https://underthewire.tech/)
- [TryHackMe AD Basics](https://tryhackme.com/room/winadbasics)
- [Microsoft Learn](https://learn.microsoft.com/training/)
- [Nmap manual](https://nmap.org/book/man.html)
- [Arch Wiki nftables](https://wiki.archlinux.org/title/Nftables)
- [Nginx official Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)
- [Apache apachectl docs](https://httpd.apache.org/docs/current/programs/apachectl.html)
- [Microsoft Group Policy results docs](https://learn.microsoft.com/en-us/powershell/module/grouppolicy/get-gpresultantsetofpolicy)
- [Microsoft Sysmon docs](https://learn.microsoft.com/sysinternals/downloads/sysmon)

**Tryout restriction from the slide deck:** During the actual tryout, **AI assistants are not allowed**; phones are collected; your own notes and documentation are allowed. Prepare and practice beforehand. Respect the competition's networking and tool-usage rules; only work within explicitly authorized environments.
