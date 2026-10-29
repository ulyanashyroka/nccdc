# NECCDC Tryout Practice Workbook — No Solutions Shown

> Work only on machines and subnets you're explicitly authorized to administer. These are **invented practice incidents**, not disclosed tryout tasks. Do not use AI during the official tryout (briefing slide 12). Keep a record of your commands, evidence, smallest safe change, and client-side verification.

## Instructions

For **every scenario**, write: (1) likely failure area, (2) first 2–4 read-only commands with exact syntax, (3) what output would distinguish your hypotheses, (4) one minimal authorized change *if evidence confirms it*, (5) verification from the intended user/client. If a task takes more than 12 minutes in a timed practice, document your current hypothesis, bank other tasks, and return.

### Scenario 1 — Beginner: Website refuses connections

The restaurant owner says: "The menu page is gone." SSH still works. `systemctl status nginx` reports `Active: failed`, and the log says `bind() to 0.0.0.0:80 failed (98: Address already in use)`.

**Your turn:** Which command identifies the listener? What would you check *before* disabling that service? How would you verify the correct website is still served?

### Scenario 2 — Beginner: Remote connection fails

`curl -I http://127.0.0.1` returns 200 on Linux. A separate client cannot reach `192.168.56.10:80`. `ss -ltnp` shows `127.0.0.1:80` only.

**Your turn:** What does the listener tell you? Which configuration and firewall checks come next?

### Scenario 3 — Intermediate: Nginx returns 502 on PHP

Static `/menu.html` returns 200, `/checkout.php` returns 502. `systemctl status nginx` says running. Error log mentions `connect() to unix:/run/php/php8.3-fpm.sock failed`.

**Your turn:** What commands inspect PHP-FPM, sockets, and permissions? What is the difference between a missing socket and a permission-denied socket?

### Scenario 4 — Intermediate: Database rejects the application

PostgreSQL runs, but the site log says `no pg_hba.conf entry for host "192.168.56.24", user "app_reader", database "reservations"`.

**Your turn:** Identify the role of `pg_hba.conf`, how to locate it, what should be scoped tightly in a permitted repair, and how to test the application account safely.

### Scenario 5 — Intermediate: Sessions disappear

Redis replies `PONG`; the owner reports carts vanish after restart. Redis `INFO persistence` shows AOF disabled and RDB snapshots not being produced.

**Your turn:** What still needs to be determined about application design? How would you test persistence safely?

### Scenario 6 — Intermediate: Disk is "full"

An application can't write a report. `df -h` shows 25% free, but `df -i` shows 100% inodes on `/var`.

**Your turn:** What is an inode, and which directories/files would you inspect without deleting unrelated business data?

### Scenario 7 — Beginner: Windows employee permissions

A cashier can open `\OFFICE01\RestaurantDocs` but can't save changes. Share permissions show Change for `RestaurantStaff`; NTFS ACL shows Read for the same group.

**Your turn:** Which layer controls the effective write access? What exact PowerShell commands inspect the two layers?

### Scenario 8 — Intermediate: RDP fails for one employee

`TermService` runs, TCP 3389 is listening, and other employees connect successfully. This employee reaches the sign-in screen but receives "not authorized to sign in through Remote Desktop Services."

**Your turn:** Which group membership, user-right assignment, and GPO result checks would you make?

### Scenario 9 — Intermediate: GPO changed unexpectedly

One office Windows machine has a different lockout policy than the others. The help desk says the OU's GPO looks correct.

**Your turn:** Which command proves the **applied** computer and user policy rather than just the configured GPO? What could prevent a GPO from applying?

### Scenario 10 — Intermediate: Unfamiliar scheduled task

A program runs after every reboot. It is not visible in the Startup folder. The task name is `UpdaterDaily` and launches a binary from a user's temporary directory.

**Your turn:** How do you inspect tasks, service paths and Run keys and preserve evidence before removing anything?

### Scenario 11 — Advanced: Kerberos account risk

An AD audit flags a service account with SPNs, a weak rotation history, and membership in a privileged group.

**Your turn:** Explain the Kerberoasting risk conceptually, which group and SPN checks you can run without extracting ticket material, and what least-privilege and credential protections matter.

### Scenario 12 — Advanced: Sensitive replication permissions

A non-DC service principal appears to have unexpected directory replication rights.

**Your turn:** Why could this matter for DCSync? What should you document and escalate? Do **not** attempt replication or credential extraction.

### Scenario 13 — Advanced: TLS warning

`openssl s_client` shows the certificate is within its validity dates, but the browser rejects `portal.restaurant.test`. The SAN only lists `www.restaurant.test`.

**Your turn:** Why won't merely renewing an identical certificate solve the problem? What hostname and chain details must match?

### Scenario 14 — Intermediate: Docker app is healthy but data missing

The container runs, yet order history disappeared after a recreation. `docker inspect` shows the app path wasn't mounted to a persistent volume.

**Your turn:** How do you inspect mounts and backup location? Why is `docker system prune --volumes` unsafe?

### Scenario 15 — Advanced: SSO login loop

Windows authentication works. Staff website redirects to SSO and then back to login. DNS and port 443 are healthy, with no certificate warning.

**Your turn:** What OIDC/SAML fields and time synchronization issues might cause a loop? Which logs/paths would you inspect first?

### Scenario 16 — Timed multi-system inject

**Owner request:** "The menu website must work from the cashier's computer, the new night manager needs *read-only* access to the order share, and the inventory report must run by the top of the hour. Please don't interrupt existing sales."

**Your turn:** In 15 minutes, prioritize tasks, split Linux and Windows investigation, avoid over-permissioning, explain each verification and write the report you would hand in. No blanket firewall resets or server reboots.

## After-action worksheet

```text
Scenario:
Intended user/client:
What was already working:
Top two hypotheses:
Commands run and evidence:
Smallest reversible repair (if verified):
Exact proof of success:
What I would not change:
Elapsed minutes:
```

**See [`README.md`](README.md) for the underlying lessons and example command syntax.**
