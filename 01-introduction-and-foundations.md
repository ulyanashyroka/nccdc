# 01 Introduction And Foundations

**Source:** NECCDC Tryout Briefing, plus explicitly illustrative practice material. See the [complete guide](README.md).

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

