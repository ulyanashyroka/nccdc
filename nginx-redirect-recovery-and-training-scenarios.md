# NECCDC Nginx Redirect Recovery: Scenario Review and Advanced Challenge

## NECCDC Scenario #1: Incorrect — 0/1

You selected A. Restart the entire server because Nginx must be malfunctioning.

The correct answer is B. Preserve the configuration, compare it against a trusted baseline, and verify whether the redirect is unauthorized before correcting it.

Your instinct was to restore service availability, which is important. However, a restart would not fix the actual problem in this scenario.

Here's what we already know:

| Evidence                                  | What it proves                                                      |
| ----------------------------------------- | ------------------------------------------------------------------- |
| `systemctl status nginx` → Running        | Nginx is operational                                                |
| `nginx -t` → Successful                   | The configuration passes Nginx's validation checks                  |
| `return 302 https://notice.example.test;` | A configuration directive is instructing Nginx to redirect requests |
| Owner does not recognize the destination  | The redirect is suspicious and needs investigation                  |

The key lesson: A service can be running perfectly while following malicious or incorrect instructions.

Let's examine the configuration, investigate what changed, and restore the website using the same procedures you would use in your NECCDC tryout.

## 1. What exactly does `return 302` do?

Look at the suspicious configuration:

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

The most important line is:

```nginx
return 302 https://notice.example.test;
```

Let's break it down.

| Component                     | Meaning                                                        |
| ----------------------------- | -------------------------------------------------------------- |
| `return`                      | Stop processing the request and send a specified HTTP response |
| `302`                         | HTTP status code for a temporary redirect                      |
| `https://notice.example.test` | Destination provided to the client's browser                   |
| `;`                           | Ends the Nginx directive                                       |

The `return` directive tells Nginx to respond immediately rather than continue serving the normal website content.

### What happens to the customer?

Customer browser

Requests [http://orders.restaurant.test](http://orders.restaurant.test)

Nginx server

Receives the HTTP request and applies the redirect directive.

```http
HTTP/1.1 302 Moved Temporarily
Location: https://notice.example.test
```

Browser follows the redirect

Attempts to open notice.example.test instead

The website didn't necessarily crash. Nginx was instructed to send visitors somewhere else.

A `302` redirect can be legitimate, such as redirecting customers temporarily to a maintenance page. In our scenario, it is suspicious because the restaurant owner doesn't recognize the destination.

## 2. Why did `nginx -t` report success?

You executed:

```bash
sudo nginx -t
```

And received:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

This is entirely possible even with an unauthorized redirect.

`nginx -t` verifies the configuration's syntax and attempts to open referenced files. It does not judge whether the configuration meets the business's intended requirements.

Consider these two directives:

Intended behavior

```nginx
return 302 https://orders.restaurant.test;
```

A possible authorized redirect to the restaurant's HTTPS site.

Suspicious behavior

```nginx
return 302 https://notice.example.test;
```

Redirects customers to an unrecognized destination.

Both are syntactically valid.

This is a distinction I want you to remember for NCCDC:

Configuration validity ≠ configuration correctness ≠ service availability.

## 3. Exact investigation and recovery commands

Let's assume this is an authorized Ubuntu practice server, and the Nginx website configuration lives at:

`/etc/nginx/sites-available/restaurant.conf`

### Step 1 — Confirm the redirect exists

First, inspect what the website currently returns.

```bash
curl -I -H 'Host: orders.restaurant.test' \
  http://127.0.0.1/
```

Example:

```text
HTTP/1.1 302 Moved Temporarily
Server: nginx
Location: https://notice.example.test
```

The `-I` requests response headers. The `-H` supplies the expected website hostname.

This confirms that a local request using the intended hostname receives the suspicious redirect. We aren't following the redirect or visiting the suspicious destination.

### Step 2 — Discover the active Nginx configuration

```bash
sudo nginx -T
```

`-T` tests and dumps the configuration, including loaded include files.

For a long output, use:

```bash
sudo nginx -T 2>&1 | less
```

Search within `less` using `/notice.example.test`.

You can also search the configuration directory:

```bash
sudo grep -RIn "notice.example.test" /etc/nginx/
```

Example output:

```text
/etc/nginx/sites-available/restaurant.conf:7:
    return 302 https://notice.example.test;
```

What does `grep -RIn` mean?

| Option | Meaning                        |
| ------ | ------------------------------ |
| `-R`   | Search directories recursively |
| `-I`   | Skip binary files              |
| `-n`   | Show matching line numbers     |

This is useful because Nginx often uses multiple included configuration files rather than one large file.

Be aware that a full configuration dump may contain sensitive settings. Don't publish it in your GitHub notes without reviewing and removing secrets.

### Step 3 — Preserve the evidence

Before modifying a potentially unauthorized change, make a copy.

```bash
sudo install -d -m 700 /root/incident-evidence
```

Then:

```bash
sudo cp -a \
  /etc/nginx/sites-available/restaurant.conf \
  /root/incident-evidence/restaurant.conf.before-repair
```

Now you have a copy of the suspicious configuration.

A copy on the same server is useful operationally, but not necessarily trustworthy evidence if the server is compromised. Follow your team's rules for preserving evidence securely.

### Step 4 — Compare against a known-good configuration

Suppose the team has a trusted baseline:

`/root/known-good/restaurant.conf`

Run:

```bash
sudo diff -u \
  /root/known-good/restaurant.conf \
  /etc/nginx/sites-available/restaurant.conf
```

Example:

```diff
 location / {
-    try_files $uri $uri/ =404;
+    return 302 https://notice.example.test;
 }
```

Read this carefully.

- `-` means the line exists in the first file (the trusted baseline).
- `+` means the line exists in the second file (the current configuration).

The result suggests that the original static-file handling was replaced with a redirect.

Now we have evidence to support a targeted correction.

If you have no trustworthy baseline, first establish what the website is supposed to serve. Don't invent a replacement configuration simply because the current one looks suspicious.

### Step 5 — Make the authorized correction

Open the configuration:

```text
sudoedit /etc/nginx/sites-available/restaurant.conf
```

Assuming the trusted baseline confirms this is a static website, restore the appropriate location block:

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

`try_files` checks whether the requested file or directory exists and otherwise returns a 404 response. This example is not a universal fix; a PHP application or reverse-proxy website might require a different location block.

### Step 6 — Validate and apply

Before activating changes:

```bash
sudo nginx -t
```

If validation succeeds, apply the new configuration:

```bash
sudo systemctl reload nginx
```

Why reload instead of restart?

Nginx can start new workers with an updated configuration while gracefully retiring old workers. This generally avoids the interruption associated with a full restart.

### Step 7 — Verify that the actual problem is resolved

First, verify the local HTTP response:

```bash
curl -I -H 'Host: orders.restaurant.test' \
  http://127.0.0.1/
```

Then, from an authorized remote client, use the actual server IP while preserving the intended hostname:

```bash
curl -I --resolve orders.restaurant.test:80:192.168.20.10 \
  http://orders.restaurant.test/
```

Replace the IP with the server's real address.

A successful result might be:

```text
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
```

Also verify that the content and ordering workflow are correct. An HTTP 200 response alone doesn't establish that the whole application works.

If you suspect someone intentionally modified the configuration, investigate how they gained access, which accounts they used, and whether other settings were changed. Restoring the redirect doesn't necessarily resolve the security incident.

## 4. Memorize this configuration-recovery sequence

Read-only investigation → Targeted repair → Verification

| Step | Command | Purpose |
| --- | --- | --- |
| 1 | `systemctl status nginx` | Check service state |
| 2 | `curl -I ...`            | Observe HTTP behavior        |
| 3 | `nginx -T`               | Inspect loaded configuration |
| 4 | `grep`, `diff`, `stat`   | Find differences and clues   |
| 5 | `cp -a`                  | Preserve the original        |
| 6 | `sudoedit`               | Correct verified settings    |
| 7 | `nginx -t`               | Validate configuration       |
| 8 | `systemctl reload nginx` | Apply the change             |
| 9 | `curl` from client       | Verify the repaired service  |

## 5. NECCDC Scenario #2 — More challenging configuration incident

Now we're moving from a simple redirect problem to a multi-service failure.

**Incident:** Restaurant ordering system failing

**Status:** Active

**Environment:** WEB01 · Ubuntu Server · Nginx + PHP-FPM · Simulated tryout

The restaurant owner reports:

"Our homepage is working again, but customers cannot complete orders. The checkout page gives them an error. We haven't changed anything ourselves."

You investigate.

Check 1 — Nginx service

```bash
sudo systemctl status nginx
```

Result:

```text
Active: active (running)
```

Check 2 — Application response

```bash
curl -I -H 'Host: orders.restaurant.test' \
  http://127.0.0.1/checkout.php
```

Result:

```text
HTTP/1.1 502 Bad Gateway
Server: nginx
```

Check 3 — Nginx error log

```bash
sudo tail -n 30 /var/log/nginx/error.log
```

Relevant message:

```text
connect() to unix:/run/php/php8.3-fpm.sock
failed (2: No such file or directory)
while connecting to upstream
```

Check 4 — PHP-FPM services

```bash
systemctl list-units 'php*-fpm.service'
```

Result:

```text
UNIT                 ACTIVE  SUB
php8.2-fpm.service   active  running
```

You're told that the checkout application is business-critical and must remain available for scoring.

Question: What should you investigate next?

A. Restart the entire Linux server because PHP-FPM is malfunctioning.

B. Inspect Nginx's `fastcgi_pass` directive, PHP-FPM's actual listening socket, and the trusted configuration before modifying anything.

C. Disable the Linux firewall because the checkout page is unavailable.

D. Delete the PHP application and reinstall Nginx.

This next scenario tests whether you can distinguish a web server failure from a backend application failure. That's a useful step toward becoming the kind of all-rounder your NECCDC briefing describes.
