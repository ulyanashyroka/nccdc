# 08 Practice And Command Reference

**Source:** NECCDC Tryout Briefing, plus explicitly illustrative practice material. See the [complete guide](README.md).

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
