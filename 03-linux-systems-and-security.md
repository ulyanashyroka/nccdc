# 03 Linux Systems And Security

**Source:** NECCDC Tryout Briefing, plus explicitly illustrative practice material. See the [complete guide](README.md).

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

