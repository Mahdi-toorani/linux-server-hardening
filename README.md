# linux-server-hardening
hardening for linux ssh

# How to Secure a Linux Server from Scratch: SSH, Firewall, and Fail2Ban

I want to start this with something that actually happened to me.

I spun up a VPS, installed Ubuntu, and left it running while I worked on something else. Six hours later I opened the auth.log out of curiosity.

Over 400 failed SSH login attempts.

The server was less than a day old. Nobody knew its IP. Didn't matter — automated scanners don't need to know you exist, they sweep everything. That was the moment the theory from my Security+ prep stopped being abstract.

This tutorial walks through the steps I now apply to every new Linux server before it does anything else. Nothing fancy — just the fundamentals that actually matter.

---

## What You'll Need

- A Linux server (Ubuntu 20.04/22.04 or Debian) — local VM or VPS both work
- Root or sudo access
- A local machine to SSH from

---

## Step 1: Update Before Anything Else

I know this sounds obvious. Do it anyway.

```bash
sudo apt update && sudo apt upgrade -y
```

Running hardening steps on an unpatched system is like putting a deadbolt on a rotten door. Patches first, configuration second.

---

## Step 2: Create a Non-Root Admin User

Root has total control over your system. That's both its strength and its problem — one bad command, one compromised session, and there are no guardrails.

This comes directly from the principle of least privilege. If you've done Security+, you know this concept well. Accounts should only have access to what they actually need.

```bash
adduser secadmin
usermod -aG sudo secadmin
```

Switch to this user and don't go back to root unless you have a specific reason.

If you're coming from a Windows/MCSE background, think of this like separating your Domain Admin account from your daily-use account. Same concept, different OS.

---

## Step 3: Harden SSH — This Is the Important One

SSH defaults to port 22. It's in every scanner's default list. It was in my CCNA material, it's in every attacker's playbook, and it's the first thing automated bots check.

Changing the port isn't a real security measure by itself — security through obscurity isn't a defense. But combined with everything else here, it eliminates a huge amount of automated noise.

Open the SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

Find and update these lines (some may need to be uncommented):

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
LoginGraceTime 30
```

Why each one matters:

**Port 2222** — moves you off the default target. Won't stop a determined attacker, but eliminates most automated scanning traffic.

**PermitRootLogin no** — even if someone gets the root password, they can't log in directly.

**PasswordAuthentication no** — once you set up key-based auth, passwords become irrelevant. Remove the vector entirely.

**MaxAuthTries 3** — limits guesses per connection. Combined with Fail2Ban, this gets aggressive fast.

**LoginGraceTime 30** — default is 120 seconds. That's generous to bots. 30 is enough.

Restart SSH:

```bash
sudo systemctl restart sshd
```

> **Important:** Before closing your current session, open a second terminal and test the new port. This is the mistake everyone makes once.

---

## Step 4: Set Up SSH Key Authentication

From your local machine:

```bash
ssh-keygen -t ed25519 -C "server-label"
ssh-copy-id -p 2222 secadmin@your_server_ip
```

Why Ed25519 over RSA? It uses elliptic curve cryptography, produces smaller keys, verifies faster, and is harder to brute force. LPIC-1 covers SSH basics but this detail is worth knowing beyond the exam.

Once key login is confirmed working, the `PasswordAuthentication no` setting kicks in and passwords are permanently off.

---

## Step 5: Configure the Firewall

UFW is a readable front-end for iptables. If you've done CCNA or worked with Cisco ACLs, the logic is identical: default deny, explicit permit only for what's needed.

```bash
sudo apt install ufw

sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo ufw enable
```

Check the result:

```bash
sudo ufw status verbose
```

If a port isn't listed, it's blocked. That's the goal.

---

## Step 6: Install Fail2Ban

UFW controls what traffic reaches your server. Fail2Ban handles traffic that gets through and then misbehaves.

```bash
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit the local config:

```bash
sudo nano /etc/fail2ban/jail.local
```

Find the `[sshd]` section and configure it:

```ini
[sshd]
enabled  = true
port     = 2222
maxretry = 3
bantime  = 3600
findtime = 600
```

3 failed attempts within 10 minutes = banned for 1 hour. Set `bantime` to `86400` if you want to be less forgiving.

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Check active bans:

```bash
sudo fail2ban-client status sshd
```

First time I ran this on a live server, there were 9 IPs banned within two hours. Completely normal.

---

## Step 7: Enable Automatic Security Updates

Manual patching is easy to forget. Automate the critical stuff:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

This handles security patches only — it won't auto-update application packages and break things.

---

## Quick Verification Checklist

Before treating this server as ready for anything:

- [ ] System fully updated
- [ ] Non-root admin account created and tested
- [ ] SSH moved off port 22, root login disabled
- [ ] Key-based authentication confirmed working
- [ ] Password authentication disabled
- [ ] UFW active — verify: `sudo ufw status`
- [ ] Fail2Ban running — verify: `sudo systemctl status fail2ban`
- [ ] Automatic security updates enabled

---

## What This Doesn't Cover

This is a baseline, not a complete security posture. Depending on what your server runs, also look at:

- **AppArmor or SELinux** for mandatory access control
- **auditd** for detailed system call logging
- **Lynis** — run it and read what it finds
- Regular log reviews — the logs will tell you things if you read them

The certifications give you the framework. The logs give you the reality check.

If you run into issues applying any of this to your setup, drop it in the comments.
