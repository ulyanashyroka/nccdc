# NECCDC Training: Configuration Files, Unauthorized Changes & Recovery

This is one of the most important skills for your tryout. Imagine someone accessed the restaurant's Linux server and changed its configuration. The website stops working, employees lose access, or a previously secure service becomes exposed.

Your job is to figure out what was changed, where the change was made, what the correct configuration should be, and how to restore it safely.

Your NECCDC briefing specifically mentions Nginx, Apache, SSH, systemd, firewalls, Active Directory, Group Policy, and Windows permissions. We'll work with those systems.

*Source context: the NECCDC tryout briefing referenced in the original discussion.*

## Segment 1 — What exactly is a configuration?

A configuration tells an application or operating system how it should behave.

For example, a web server's configuration can determine which port to listen on, which website directory to serve, which hostname to accept, and which backend application to contact.

1. **Configuration file:** defines what the service should do; loaded when it starts or reloads.
2. **Running service:** applies the configuration to actual requests.
3. **Website, database, SSH, or file sharing:** works correctly—or fails because of the settings.

A critical distinction: changing a configuration file does not always change the running service immediately. Some applications require a reload, others a restart, and certain changes are managed dynamically.

### Common configurations you might encounter

| Technology       | Common location or interface                           | What someone could change                        |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------ |
| Nginx            | `/etc/nginx/nginx.conf`, `sites-available/`, `conf.d/` | Ports, web roots, redirects, proxy targets       |
| Apache           | `/etc/apache2/` on Ubuntu                              | Virtual hosts, modules, directory permissions    |
| SSH              | `/etc/ssh/sshd_config`, `sshd_config.d/`               | Authentication, permitted users, listening ports |
| systemd          | `/etc/systemd/system/` and unit drop-ins               | Startup commands, dependencies, environment      |
| PostgreSQL       | `postgresql.conf`, `pg_hba.conf`                       | Database listening addresses, authentication     |
| Redis            | `redis.conf` and runtime settings                      | Bind addresses, security, persistence            |
| Linux firewall   | nftables/ufw configurations                            | Which connections are allowed                    |
| Windows          | Registry, local policies, service settings             | Startup behavior, RDP, network security          |
| Active Directory | Users, groups, organizational units, GPOs              | Permissions, policies, authentication            |
| SMB shares       | Share settings and NTFS ACLs                           | Who can read or modify files                     |

These locations are representative. Actual paths vary by distribution and installation, and the effective configuration may combine several files or policy sources.

## Segment 2 — Linux: An attacker changes an Nginx configuration

Let's simulate the restaurant's web server.

The owner sends this message:

> "Our ordering website worked yesterday. Now customers can't access it. The previous administrator may have changed something."

We know the intended website is:

`orders.restaurant.test`

And it should use TCP port `80`.

### Step 1 — Check whether the service is running

```bash
sudo systemctl status nginx
```

Illustrative output:

```text
● nginx.service - A high performance web server
     Loaded: loaded
     Active: active (running)
```

Interesting: Nginx is running.

But a running service doesn't guarantee that its configuration is correct.

Now check its listening ports:

```bash
sudo ss -ltnp
```

Suppose the relevant output is:

```text
Local Address:Port   Process
0.0.0.0:8080         nginx
```

The service is listening on port `8080`, not the required port `80`.

We now have a specific hypothesis: perhaps the configuration was changed to use the wrong port.

### Step 2 — Find the active configuration

Start with:

```bash
sudo nginx -T
```

This is an extremely useful command.

`nginx -T` tests the configuration and prints the loaded configuration files, including included configuration content. It helps you discover which settings Nginx is actually reading.

`nginx -t` only validates the configuration and referenced files; it doesn't print the whole configuration.

Because Nginx can load multiple files, inspect the enabled sites:

```bash
ls -l /etc/nginx/sites-enabled/
```

Example:

```text
restaurant.conf -> /etc/nginx/sites-available/restaurant.conf
```

The arrow means `restaurant.conf` is a symbolic link pointing to the configuration file in `sites-available`.

On an Ubuntu installation using this layout, we would inspect the target file:

```bash
sudo less /etc/nginx/sites-available/restaurant.conf
```

Use `q` to exit `less`.

### Step 3 — Read the suspicious configuration

Suppose we find:

```nginx
server {
    listen 8080;
    server_name orders.restaurant.test;

    root /srv/restaurant/old-site;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Let's break down each setting.

| Directive     | Value                      | What it controls                             |
| ------------- | -------------------------- | -------------------------------------------- |
| `server`      | Block                      | Defines a virtual web server                 |
| `listen`      | `8080`                     | TCP port used by this server block           |
| `server_name` | `orders.restaurant.test`   | Hostname associated with this website        |
| `root`        | `/srv/restaurant/old-site` | Directory from which static files are served |
| `location /`  | `/`                        | Rules for requests under the root path       |
| `try_files`   | `$uri $uri/ =404`          | Tests candidate paths, otherwise returns 404 |

Two settings deserve investigation:

CURRENT

```nginx
listen 8080;
```

Website listens on the wrong port for our stated requirement.

EXPECTED

```nginx
listen 80;
```

Website should listen on HTTP port 80.

CURRENT

```nginx
root /srv/restaurant/old-site;
```

EXPECTED

```nginx
root /srv/restaurant/site;
```

Both changes could prevent users from reaching the intended website, even though Nginx starts successfully.

But how do we know these values are wrong?

Because we must compare them with a trusted expected configuration, not guess based on what looks unusual.

## Segment 3 — Detecting what changed

Now we move from basic troubleshooting to incident investigation.

### Step 1 — Check the file's metadata

```bash
sudo stat /etc/nginx/sites-available/restaurant.conf
```

Example:

```text
File: restaurant.conf
Size: 249
Modify: 2026-10-29 14:12:33
Change: 2026-10-29 14:12:33
```

`Modify` generally indicates when file content was last changed. `Change` is the inode metadata-change time, which can also change when content is modified. Neither is a reliable record of who edited the file.

The times give you a starting point for investigation, but timestamps can be altered.

### Step 2 — Compare against a trusted configuration

Suppose your team has a verified, known-good file at:

`/root/known-good/restaurant.conf`

Use:

```bash
sudo diff -u \
  /root/known-good/restaurant.conf \
  /etc/nginx/sites-available/restaurant.conf
```

Illustrative result:

```diff
- listen 80;
+ listen 8080;

- root /srv/restaurant/site;
+ root /srv/restaurant/old-site;
```

This is extremely useful in a competition.

The `-` lines show the expected values that disappeared, while `+` lines show the values currently present.

Git can serve a similar purpose if the configuration was version-controlled. But a local repository on a compromised server is not necessarily trustworthy.

### Step 3 — Look for related activity

Check service logs:

```bash
sudo journalctl -u nginx -n 50 --no-pager
```

Investigate recent SSH sessions:

```bash
last -a | head
```

For relevant authentication logs on a systemd-based server:

```bash
sudo journalctl -u ssh --since "2 hours ago"
```

On some systems the SSH service is named `sshd`, rather than `ssh`.

These logs can help establish a timeline. They do not automatically prove which user changed the file. File-level auditing would need to have been configured beforehand to provide stronger attribution evidence.

If you genuinely suspect unauthorized access, notify the team lead, preserve relevant evidence, and investigate whether credentials or other configurations were affected. Merely changing two settings back may not remove the underlying security problem.

## Segment 4 — Safely restore the Linux configuration

We now have evidence that the port and document root differ from the authorized configuration.

Here is the workflow to follow.

1. **Preserve evidence:** record current settings and save a copy.
2. **Confirm the intended configuration:** compare against a trusted source.
3. **Make a targeted correction:** change only the verified incorrect settings.
4. **Validate the configuration:** check syntax and dependencies.
5. **Reload and verify:** test the website locally and remotely.

### Step 1 — Preserve the original file

In a permitted practice environment:

```bash
sudo cp -a \
  /etc/nginx/sites-available/restaurant.conf \
  /root/restaurant.conf.before-repair
```

`cp -a` copies the file while preserving relevant file attributes.

Store incident evidence outside the web root, restrict access to it, and follow the competition's incident-handling rules. Don't overwrite the only copy of the suspicious file.

### Step 2 — Edit the configuration

Use:

```text
sudoedit /etc/nginx/sites-available/restaurant.conf
```

The command opens the file using your configured editor with the privileges needed to save changes.

Make the two authorized corrections:

```nginx
server {
    listen 80;
    server_name orders.restaurant.test;

    root /srv/restaurant/site;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

This assumes the intended site is static, the directory exists, and no other configured service conflicts with port 80.

### Step 3 — Validate before applying

```bash
sudo nginx -t
```

Successful output might include:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

This tells us that the configuration is syntactically valid and Nginx could access the files it checked.

It does not guarantee the website serves the correct content.

### Step 4 — Reload instead of unnecessarily restarting

```bash
sudo systemctl reload nginx
```

A reload asks Nginx to apply its new configuration without the same interruption normally involved in a full restart.

Then verify:

```bash
sudo ss -ltnp '( sport = :80 )'
```

And test the expected site:

```bash
curl -I -H 'Host: orders.restaurant.test' \
  http://127.0.0.1/
```

Then use an authorized client to verify remote access, correct page content, and any checkout or ordering functionality the restaurant actually needs.

If the repair fails, examine the new logs and use the preserved, verified previous configuration to plan a rollback rather than making unrelated changes.

## Segment 5 — What if someone changed systemd instead?

This is a different configuration layer.

Nginx's own configuration controls website behavior. systemd configuration controls how the Nginx service is managed and started.

Someone could create a systemd override that changes its startup behavior, even though the Nginx configuration itself is fine.

Inspect it using:

```bash
systemctl cat nginx
```

This displays the main service unit and any applicable drop-in configuration.

To find where the service definition and overrides are stored:

```bash
systemctl show nginx \
  -p FragmentPath \
  -p DropInPaths
```

For example, a drop-in might be found at:

```text
/etc/systemd/system/nginx.service.d/override.conf
```

The important lesson is that checking only `/etc/nginx/nginx.conf` would not identify a bad systemd override.

You can inspect or edit service overrides through:

```bash
sudo systemctl edit nginx
```

If a unit file is edited directly, its new definition generally needs to be loaded by systemd:

```bash
sudo systemctl daemon-reload
```

Then you may need to restart the affected service for appropriate settings to take effect.

`daemon-reload` and `systemctl reload nginx` are different. The former reloads systemd unit definitions; the latter asks the Nginx service to reload its application configuration.

Do not indiscriminately use `systemctl revert nginx`; it can remove legitimate overrides along with unwanted ones.

## Segment 6 — Windows: Someone changes file-sharing permissions

Now let's switch to your second tryout machine.

The restaurant owner reports:

> "The managers can access the shared schedule folder, but the employees suddenly cannot edit it."

Windows does not keep every setting in one readable configuration file. Some settings live in the registry, policy stores, Active Directory, or access-control lists.

For this task, investigate both SMB share permissions and NTFS file-system permissions.

### Step 1 — Find the share

```powershell
Get-SmbShare
```

Suppose you find:

```text
Name       Path
----       ----
StaffDocs  C:\Shares\StaffDocs
```

### Step 2 — Check the share permission

```powershell
Get-SmbShareAccess -Name "StaffDocs"
```

Illustrative output:

```text
AccountName       AccessRight
-----------       -----------
RESTAURANT\Staff   Read
```

This tells you that the Staff group has a Read allow entry on the share.

### Step 3 — Check NTFS permissions

```powershell
icacls "C:\Shares\StaffDocs"
```

Illustrative output:

```text
RESTAURANT\Staff:(OI)(CI)(M)
```

Here, `M` represents Modify permission. `(OI)` and `(CI)` are inheritance flags for files and subdirectories.

We now know:

| Permission layer | Staff access |
| ---------------- | ------------ |
| SMB share        | Read         |
| NTFS             | Modify       |

When accessing a folder over SMB, the effective permission is constrained by both layers.

Even though NTFS allows modification, the share's Read permission may prevent the required operation.

### Step 4 — Preview a targeted correction

If the documented requirement confirms Staff should have Change permission on the share:

```powershell
Grant-SmbShareAccess `
  -Name "StaffDocs" `
  -AccountName "RESTAURANT\Staff" `
  -AccessRight Change `
  -WhatIf
```

`-WhatIf` previews the change without applying it.

Once authorized, remove `-WhatIf` to perform the operation. Then recheck the share ACL and test access using an appropriate staff account.

This is much safer than granting Full Control to Everyone.

### What if Group Policy changed the behavior?

A Windows policy might override what you configured locally.

Investigate applied policy using:

```powershell
gpresult /r
```

Or produce an HTML report:

```powershell
gpresult /h C:\Temp\policy-report.html
```

The destination folder must already exist and be writable.

Group Policy Results can identify which GPO supplied applicable policy settings. Changes should be made at the actual policy source, not repeatedly overridden locally if a domain GPO will reapply them.

## Segment 7 — How do we find out who changed a configuration?

This is where troubleshooting transitions into incident response.

On Linux, file metadata and login logs may provide clues. For definitive file-change attribution, appropriate auditing must generally have been enabled beforehand.

On Windows, inspect relevant security events:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    Id        = 4663,4670
    StartTime = (Get-Date).AddHours(-2)
} -MaxEvents 30
```

Event 4663 can reflect audited file access, while 4670 can record permission changes. These events are not guaranteed to exist: the relevant audit policy and object-level auditing must have been configured.

When someone may have entered and modified a system, ask:

| Question                                     | Why it matters                      |
| -------------------------------------------- | ----------------------------------- |
| What stopped working?                        | Establishes the observable incident |
| Which configuration controls it?             | Narrows the investigation           |
| What differs from a trusted baseline?        | Identifies suspected changes        |
| When did the change occur?                   | Helps build the timeline            |
| Which account had access?                    | Helps investigate authorization     |
| Was the change accidental or malicious?      | Determines the response             |
| What else may have changed?                  | Prevents incomplete recovery        |
| Is functionality restored and access secure? | Confirms the actual objective       |

If malicious access is suspected, don't assume the configuration repair concludes the incident. Other accounts, credentials, scheduled tasks, startup settings, and persistence mechanisms may need investigation.

## Segment 8 — Your first configuration recovery challenge

Let's apply everything we've learned. I won't reveal the correct answer until you submit.

### NECCDC Scenario #1 — Unauthorized web configuration change

**Status:** Website down

**Environment:** WEB01 · Ubuntu Server · Restaurant website

The restaurant owner reports:

"Our website worked this morning, but customers are now being redirected to a different page. I think someone was changing server settings."

You investigate:

```bash
sudo systemctl status nginx
```

Result:

```text
Active: active (running)
```

Next:

```bash
sudo nginx -t
```

Result:

```text
syntax is ok
test is successful
```

You inspect the active website configuration:

```nginx
server {
    listen 80;
    server_name orders.restaurant.test;

    root /srv/restaurant/site;

    location / {
        return 302 https://notice.example.test;
    }
}
```

The owner confirms the restaurant has never used `notice.example.test`.

Question 1: What is the best next action?

A. Restart the entire server because Nginx must be malfunctioning.

B. Preserve the current configuration, compare it with a trusted baseline, and verify whether the redirect is unauthorized before correcting it.

C. Disable the Linux firewall because customers cannot reach the intended website.

D. Delete `/etc/nginx/` and reinstall the service immediately.

The most important principle to remember is that a configuration can be syntactically valid, the service can be running, and the system can still be completely misconfigured or compromised.

That's why the real objective is not simply making a service start. It's restoring the intended behavior, preserving security, and verifying that the business can function again.
