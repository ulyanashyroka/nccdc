# NECCDC Tryout — Study Module Index

The club's briefing describes a Linux website server (SSH) and Windows office server (RDP), and tells candidates to **diagnose and repair** broken systems. The exact tasks are deliberately undisclosed. All practice incidents in these files are invented exercises, not actual tryout questions.

Start with [README.md](README.md) for the full combined guide, or work through the modules in order:

- [01-introduction-and-foundations.md](01-introduction-and-foundations.md) — What the tryout actually asks you to do, Three levels of proficiency, Foundations: connect, establish a baseline, and troubleshoot.
- [02-linux-web-and-data.md](02-linux-web-and-data.md) — Linux web and data stack.
- [03-linux-systems-and-security.md](03-linux-systems-and-security.md) — Linux systems, security, and files.
- [04-windows-and-active-directory.md](04-windows-and-active-directory.md) — Windows and Active Directory.
- [05-advanced-windows-security.md](05-advanced-windows-security.md) — Windows: advanced security and identity.
- [06-network-discovery.md](06-network-discovery.md) — Networking and discovering hosts.
- [07-restaurant-incident-scenarios.md](07-restaurant-incident-scenarios.md) — End-to-end restaurant incidents.
- [08-practice-and-command-reference.md](08-practice-and-command-reference.md) — Timed practice and readiness rubric, Command index, References and tryout rules.

## Safety and rules

Run commands only in authorized environments. Most commands inspect state; changes require confirming the intended task and avoiding disruption. **At the actual tryout, AI tools are prohibited**, but the briefing says official documentation and your own notes are allowed. Consult the organizer for latest rules.

## What to practice first

1. Identify hosts/services/IPs and interpret web errors.
2. Linux web server and logs: Apache, Nginx, systemd, `ss`.
3. Windows AD users, groups, GPO, SMB/NTFS and RDP.
4. Connectivity: DNS, routing, firewall, listening addresses, TLS.
5. Dependency failures: PHP-FPM, databases, Docker, disk/logs.
6. Advanced security concepts: identify evidence and safeguards, not attack execution.
