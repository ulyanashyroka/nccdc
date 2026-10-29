# NCCDC Windows Web Server Troubleshooting Guide

> **Core principle:** Troubleshooting means identifying **why** something isn't working before trying to fix it.

In NCCDC, this is especially important because you might accidentally bring down a critical service while trying to repair it.

## Example scenario

A company's website is down, but Windows says its web service is running.

## Step 1. Check whether the service is running

Open PowerShell on the Windows server:

```powershell
Get-Service -Name W3SVC
```

Suppose you receive:

```text
Status    Name      DisplayName
------    ----      -----------
Running   W3SVC     World Wide Web Publishing
```

**What do we know?**

- `W3SVC` is the Windows IIS web publishing service.
- Its status is `Running`.
- But **we do not know whether the website actually works**.

Think about your Operating Systems class: a process can be running without performing its intended task successfully. Similarly, a running Windows service doesn't guarantee a functioning application.

## Step 2. Check whether the server is listening on the correct port

Websites commonly use HTTP (**port 80**) and HTTPS (**port 443**).

```powershell
Get-NetTCPConnection -LocalPort 80 -State Listen
```

Possible output:

```text
LocalAddress  LocalPort  State
------------  ---------  ------
0.0.0.0       80         Listen
```

This means the system has a TCP listener on port 80.

**But what if the command returns nothing?**

That suggests no TCP listener is currently available on port 80. We should investigate the website bindings, server configuration, or whether the application is actually serving that port.

## Step 3. Test whether the website responds

```powershell
curl.exe -I http://localhost
```

`localhost` refers to the computer itself.

Now look at the response:

| Result | What it suggests |
| --- | --- |
| `HTTP/1.1 200 OK` | This HTTP request succeeded. |
| `HTTP/1.1 404 Not Found` | Server responds, but the requested resource isn't found. |
| `HTTP/1.1 500 Internal Server Error` | Application/server encountered an error. |
| Connection refused | Nothing accepted the connection at that address and port. |
| Timeout | Connection failed or did not complete in time. |

**Important:** A port can be listening even when the website returns a `500` error.

So checking the service status and checking the port are not sufficient. We also need to verify what the application returns.

## Step 4. Determine whether the problem is local or network-related

**Can the server access its own website?**

```powershell
curl.exe -I http://localhost
```

Then follow the appropriate path:

| If local access fails | If local access succeeds but remote access fails |
| --- | --- |
| Investigate application configuration, web server bindings, logs, or the listening port. | Investigate firewalls, routing, external access, and DNS. |

For instance, if the website works locally but not from another machine, you might inspect the Windows Firewall profiles:

```powershell
Get-NetFirewallProfile
```

And test connectivity **from the other Windows machine**:

```powershell
Test-NetConnection 192.168.1.10 -Port 80
```

Replace `192.168.1.10` with the actual server address.

If `TcpTestSucceeded` is `False`, you know a remote TCP connection could not be established. You still need to determine why.

## Step 5. Inspect error logs

This is another major NCCDC skill.

For example:

```powershell
Get-WinEvent -LogName Application -MaxEvents 20
```

This displays the 20 most recent Application log events. Look for errors related to the affected application, then inspect the relevant service-specific logs as needed.

You can discover clues about crashes, configuration issues, or application failures.

**Don't restart or reconfigure everything immediately.** First identify the failure, make a targeted change, and test again. Also confirm the website works from the location used by the competition's scoring system.

---

## Practice: NCCDC Troubleshooting Scenario #1

Your team receives a notification:

> "The company's website is unavailable. Please investigate."

You run:

```powershell
Get-Service -Name W3SVC
```

**Result:** `Running`

You then run:

```powershell
Get-NetTCPConnection -LocalPort 80 -State Listen
```

**Result:** A listener is present on port 80.

### Question: What should you do next?

- **A.** Restart the entire Windows server.
- **B.** Use `curl.exe -I http://localhost` to test the website's response.
- **C.** Disable Windows Firewall.
- **D.** Delete the existing web service.

**Your answer:** ______

We'll work through these scenarios one step at a time so you learn how to investigate a problem rather than memorize isolated commands.

---

**NCCDC troubleshooting mindset:** Investigate the symptoms, identify the cause, make a targeted change, and verify that the service works without disrupting anything else.
