# 02 Linux Web And Data

**Source:** NECCDC Tryout Briefing, plus explicitly illustrative practice material. See the [complete guide](README.md).

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

