# 05 Advanced Windows Security

**Source:** NECCDC Tryout Briefing, plus explicitly illustrative practice material. See the [complete guide](README.md).

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

