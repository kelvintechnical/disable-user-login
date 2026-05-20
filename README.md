# Command-Line Email Testing (RHEL 9)
### RHCSA EX200 Lab | Part of [linux-ops-mastery](https://github.com/kelvintechnical/linux-ops-mastery)
> Use mutt and mail to test local SMTP configurations and verify messages are routed to the correct local /var/mail user spools.

![RHCSA](https://img.shields.io/badge/RHCSA-EX200-EE0000?style=flat&logo=redhat&logoColor=white)
![Topic](https://img.shields.io/badge/Topic-Remote_Administration-blue)

---

## 📋 Scenario

On **Node1**, use command-line email clients `mutt` and `mail` to test local SMTP configurations. Send messages between local users and verify they are delivered to the correct `/var/mail` spool files.

---

## 🎯 Requirements

1. Install `mutt` and `s-nail` (provides the `mail` command on RHEL 9).
2. Install and start the local MTA (`postfix`).
3. Send a test email to a local user using `mail`.
4. Verify the message landed in the correct `/var/mail` spool.
5. Read the message using `mutt`.

---

## ✅ Tasks

- Install `mutt` and `s-nail` via `dnf`
- Install, enable, and start `postfix`
- Send mail to a local user with `mail`
- Verify delivery in `/var/mail/`
- Read mail interactively with `mutt`

---

## 📚 Command Decision Map

| Lab Phrase | Question Being Asked | Tool |
|------------|---------------------|------|
| "Test SMTP" | What MTA handles local mail on RHEL? | `postfix` |
| "Send from command line" | How do I send email without a GUI? | `mail -s "subject" user` |
| "Verify mail spool" | Where does local mail land? | `/var/mail/username` |
| "Read mail interactively" | What CLI mail reader has a full UI? | `mutt` |
| "Is postfix running?" | How do I check a service? | `systemctl status postfix` |

---

## 🧠 Big Concept — Local Mail on Linux

Linux systems have a built-in mail system for **local delivery** — messages between users on the same machine. This is separate from internet email. It's used by:

- **Cron jobs** — failures and output get mailed to root
- **System services** — alerts and reports go to local admin accounts
- **SMTP testing** — verifying your MTA config before pointing it at the internet

**The flow:**
```
mail command → postfix (MTA) → /var/mail/username (spool file)
```

`/var/mail/username` is a plain text file. Every message is appended to it. `mutt` and `mail` read from this file.

---

## Step 1 — Install mutt and s-nail

```bash
sudo dnf install mutt s-nail -y
```

> **Important RHEL 9 note:** The package `mailx` no longer exists on RHEL 9 — it was renamed to `s-nail`. Installing `mailx` returns `No match for argument`. Always use `s-nail` on RHEL 9 — it provides the `mail` command.

**Expected output:**
```
Installed:
  mutt-5:2.2.6-2.el9.x86_64
  s-nail-14.9.22-9.el9_7.x86_64
  tokyocabinet-1.4.48-19.el9.x86_64
  urlview-0.9-31.20131022git08767a.el9.x86_64
Complete!
```

> `tokyocabinet` and `urlview` are dependencies pulled in automatically — expected.

---

## Step 2 — Install, enable, and start postfix

> **Important RHEL 9 AMI note:** `postfix` is NOT pre-installed on RHEL 9 AMIs — you must install it first. The `mail` command appears to send without it but nothing gets delivered.

```bash
sudo dnf install postfix -y
sudo systemctl enable --now postfix
```

**Verify:**
```bash
systemctl status postfix | grep Active
# Expected: Active: active (running)
```

**What `enable --now` does:**
- `enable` — creates a symlink so postfix starts automatically at every boot
- `--now` — also starts it immediately without needing a separate `systemctl start`

**Expected output from enable:**
```
Created symlink /etc/systemd/system/multi-user.target.wants/postfix.service → /usr/lib/systemd/system/postfix.service.
```

---

## Step 3 — Send a test email to a local user

```bash
echo "this is a test message body" | mail -s "Test Subject" tom
```

**Breaking it down:**

| Part | Meaning |
|------|---------|
| `echo "..."` | Produces the message body |
| `\|` | Pipes the body into the `mail` command |
| `mail -s "Test Subject"` | `-s` sets the subject line |
| `tom` | The local username to deliver to |

> No output after running this command is normal — it means the message was accepted by postfix for delivery.

**Send to root:**
```bash
echo "Root alert test" | mail -s "Root Test" root
```

---

## Step 4 — Verify delivery in the mail spool

```bash
sudo cat /var/mail/tom
```

> **Why `sudo`?** `/var/mail/` is restricted — use `sudo` to read another user's spool file.

**Expected output (mbox format):**
```
From ec2-user@ip-172-31-10-183.ec2.internal  Wed May 20 14:13:13 2026
Return-Path: <ec2-user@ip-172-31-10-183.ec2.internal>
X-Original-To: tom
Delivered-To: tom@ip-172-31-10-183.ec2.internal
Received: by ip-172-31-10-183.ec2.internal (Postfix, from userid 1000)
        id DD3CF18A8E9F; Wed, 20 May 2026 14:13:13 -0400 (EDT)
Subject: Test Subject
User-Agent: s-nail v14.9.22
From: Cloud User <ec2-user@ip-172-31-10-183.ec2.internal>

this is a test message body
```

**Reading the mbox output:**

| Field | Meaning |
|-------|---------|
| `From ec2-user@...` | Sent by your ec2-user account |
| `Delivered-To: tom@...` | Postfix delivered to tom locally |
| `Subject: Test Subject` | Your `-s` flag value |
| `User-Agent: s-nail` | Confirms s-nail sent it |
| `this is a test message body` | Your message body at the bottom |

**Check if the file exists and has content:**
```bash
sudo ls -lh /var/mail/tom
# Non-zero file size = mail was delivered
```

> `/var/mail/tom` is created on first delivery — it won't exist until the first message arrives.

---

## Step 5 — Read mail interactively with mutt

```bash
sudo mutt -f /var/mail/tom
```

> `-f` opens a specific mailbox file directly. `sudo` is required because `/var/mail/tom` is root-owned.

**mutt keyboard shortcuts:**

| Key | Action |
|-----|--------|
| `Enter` | Open and view the selected message |
| `q` | Go back to the message list |
| `d` | Mark message for deletion |
| `r` | Reply to message |
| `Q` | Quit mutt entirely |

> **Note:** If mutt opens a file in `vi` instead of displaying the message list, you accidentally triggered the editor. Press `Escape` then `:q!` to exit vi without saving, then navigate with the keys above to view messages — not edit them.

---

## Step 6 — Send multi-line mail from a file

```bash
cat > /tmp/mailbody.txt << EOF
Line 1: System check complete.
Line 2: All services nominal.
Line 3: No action required.
EOF

mail -s "System Report" root < /tmp/mailbody.txt
```

> Redirecting a file into `mail` is how cron jobs and scripts send formatted reports — this is a common real-world pattern.

---

## 🧠 Key Concepts

| Concept | What it means |
|---------|--------------|
| `postfix` | The default MTA (Mail Transfer Agent) on RHEL 9 — must be installed and running |
| MTA | Mail Transfer Agent — routes and delivers email messages |
| `s-nail` | RHEL 9 replacement for `mailx` — provides the `mail` command |
| `/var/mail/username` | Local mail spool — plain text file, one per user, created on first delivery |
| mbox format | Standard format for mail spools — messages appended sequentially |
| `mail -s` | Sets subject line; body comes from stdin or file redirect |
| `mutt -f /path` | Opens a specific mailbox file directly |
| Cron + mail | Failed cron jobs email output to root's local spool automatically |

---

## ⚠️ Pitfalls

- **`mailx` not found on RHEL 9** → package was renamed to `s-nail`; always use `s-nail` on RHEL 9
- **`postfix` not installed** → not pre-installed on RHEL 9 AMIs; run `dnf install postfix -y` first
- **Postfix not running** → `mail` appears to send but nothing is delivered; always check `systemctl status postfix`
- **`/var/mail/tom` doesn't exist** → file is created on first delivery; missing means mail wasn't delivered
- **Reading without sudo** → `/var/mail/` is restricted; use `sudo cat` or `sudo mutt -f`
- **mutt opens vi** → you triggered the editor accidentally; press `Escape` then `:q!` to exit, then use `Enter` to view messages
- **EC2 outbound SMTP blocked** → port 25 is blocked by AWS; this lab only works for local delivery between users on the same instance

---

## ✅ Lab Checklist

- `mutt` and `s-nail` installed via `dnf` ✓
- `postfix` installed, enabled, and running ✓
- `mail -s "Test Subject" tom` sends message ✓
- `sudo cat /var/mail/tom` confirms delivery in mbox format ✓
- `sudo mutt -f /var/mail/tom` opens the mailbox interactively ✓
- Multi-line mail sent from file with `<` redirect ✓

---

## 🔗 Related Labs

- [Command-Line Web and FTP Testing](https://github.com/kelvintechnical/elinks-iftp)
- [Network Troubleshooting with telnet and nmap](https://github.com/kelvintechnical/telnet-nmap)
- [Full RHCSA/RHCE Study Guide →](https://github.com/kelvintechnical/linux-ops-mastery)

---

## 👤 Author

**Kelvin R. Tobias** — [kelvinintech.com](https://kelvinintech.com) | [GitHub](https://github.com/kelvintechnical) | [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
