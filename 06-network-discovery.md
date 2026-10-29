# 06 Network Discovery

**Source:** NECCDC Tryout Briefing, plus explicitly illustrative practice material. See the [complete guide](README.md).

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

