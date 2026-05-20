# Disable User Login Without Removing the Account (RHEL 9)
### RHCSA EX200 Lab | Part of [linux-ops-mastery](https://github.com/kelvintechnical/linux-ops-mastery)
> Prevent tom from logging in interactively while keeping the account and all associated files intact.

![RHCSA](https://img.shields.io/badge/RHCSA-EX200-EE0000?style=flat&logo=redhat&logoColor=white)
![Topic](https://img.shields.io/badge/Topic-User_Management-blue)

---

## 📋 Scenario

On **Node1**, prevent the user `tom` from logging in to the system without deleting the account. The user account and all associated files must remain intact — only interactive login access should be disabled.

---

## 🎯 Requirements

1. Disable interactive login for `tom`.
2. Keep the user account intact — no deletion.
3. Keep all files owned by `tom` intact.
4. Verify login is blocked.

---

## ✅ Tasks

- Use `usermod -s /sbin/nologin` to disable the login shell
- Verify with `getent passwd tom`
- Confirm account and files still exist
- Test login is blocked with `su - tom`

---

## 📚 Command Decision Map

| Lab Phrase | Question Being Asked | Tool |
|------------|---------------------|------|
| "Prevent login" | How do I block a user from logging in? | `usermod -s /sbin/nologin` |
| "Without deleting" | How do I keep account intact? | Don't use `userdel` — change the shell only |
| "Account must remain" | How do I verify account still exists? | `getent passwd tom` |
| "Files must remain" | How do I confirm files are untouched? | `sudo ls -la /home/tom` |

---

## 🧠 Big Concept — Login Shell

Every user account has a **login shell** defined in `/etc/passwd`. This is the program that runs when a user logs in interactively:

```
tom:x:1001:1002::/home/tom:/bin/bash     ← normal login shell
tom:x:1001:1002::/home/tom:/sbin/nologin ← login blocked
```

`/sbin/nologin` is a special binary that immediately prints a message and exits when called — it never gives the user a shell. The account stays in `/etc/passwd` with all its files, groups, and permissions intact.

**Three ways to disable login — know all three for the exam:**

| Method | Command | Effect |
|--------|---------|--------|
| Change shell | `usermod -s /sbin/nologin tom` | Shell replaced — cleanest method |
| Lock password | `usermod -L tom` | Locks password only — SSH keys still work |
| Set expiry | `usermod -e 1 tom` | Sets account expiry date to the past |

> For RHCSA, `usermod -s /sbin/nologin` is the standard expected answer.

---

## Step 1 — Check tom's current shell

```bash
getent passwd tom
```

This reads `/etc/passwd` for tom's entry. You're looking at 7 colon-separated fields:

```
tom : x : 1001 : 1002 : : /home/tom : /bin/bash
 ↑     ↑    ↑     ↑    ↑      ↑           ↑
name  pw   uid   gid  comment home       shell
```

**Reading the output field by field:**

| Field | Value | Meaning |
|-------|-------|---------|
| `tom` | tom | Username |
| `x` | x | Password stored in `/etc/shadow` |
| `1001` | 1001 | User ID |
| `1002` | 1002 | Primary group ID |
| *(empty)* | | Comment/GECOS field — blank |
| `/home/tom` | /home/tom | Home directory |
| `/bin/bash` | /bin/bash | Login shell — this is what we're changing |

**Expected output:**
```
tom:x:1001:1002::/home/tom:/bin/bash
```

---

## Step 2 — Disable the login shell

```bash
sudo usermod -s /sbin/nologin tom
```

> `usermod` modifies an existing user account. `-s` sets the login shell. `/sbin/nologin` blocks interactive login immediately.

**Verify the change:**
```bash
getent passwd tom
```

**Expected output:**
```
tom:x:1001:1002::/home/tom:/sbin/nologin
```

Notice exactly what changed — only the last field:

```
Before: tom:x:1001:1002::/home/tom:/bin/bash
After:  tom:x:1001:1002::/home/tom:/sbin/nologin
```

Everything else — UID, GID, home directory — is completely untouched. The account is intact, only the door is locked.

---

## Step 3 — Confirm account and files are intact

```bash
# Confirm account still exists
id tom

# Confirm home directory and files are untouched
sudo ls -la /home/tom
```

> **Why `sudo`?** Tom's home directory has `drwx------` permissions — only tom and root can read it. Running `ls` without sudo returns `Permission denied` even though the directory exists. That's correct Linux behavior, not an error.

**Expected output:**
```
total 12
drwx------. 2 tom  tom   62 May 20 11:16 .
drwxr-xr-x. 4 root root  33 May 20 11:16 ..
-rw-r--r--. 1 tom  tom   18 Feb 15  2024 .bash_logout
-rw-r--r--. 1 tom  tom  141 Feb 15  2024 .bash_profile
-rw-r--r--. 1 tom  tom  492 Feb 15  2024 .bashrc
```

**Reading the home directory files:**

| File | Meaning |
|------|---------|
| `drwx------` | Directory owned by tom — no access for other users |
| `.bash_logout` | Runs when tom logs out |
| `.bash_profile` | Runs at login — sets environment variables |
| `.bashrc` | Runs for interactive shells — aliases and functions |

> These files came from `/etc/skel` when tom was created and are completely untouched.

---

## Step 4 — Set a password and verify login is blocked

> Tom needs a password set before `su -` can test the nologin block — otherwise it fails at authentication before reaching the shell check.

```bash
sudo passwd tom
```

**Then test the login block:**
```bash
su - tom
```

**Expected output:**
```
This account is currently not available.
```

Notice what happened in sequence:
- Password was accepted — authentication succeeded ✓
- Login was blocked — `/sbin/nologin` fired immediately after auth ✓
- Account and files intact — nothing was deleted ✓

> **Why not `su -s /bin/bash tom`?** That command overrides the shell — it bypasses `/sbin/nologin` entirely and is not a valid login test. Always use `su - tom` to test the actual login behavior.

---

## 🧠 Key Concepts

| Concept | What it means |
|---------|--------------|
| Login shell | The program in `/etc/passwd` that runs at interactive login |
| `/sbin/nologin` | A binary that blocks login and prints a message — account stays intact |
| `/etc/passwd` format | `username:x:uid:gid:comment:home:shell` — shell is the last field |
| `usermod -s` | Changes the login shell for an existing user |
| `usermod -L` | Locks the password only — SSH key-based login still works |
| `getent passwd` | Reads `/etc/passwd` — confirms current shell setting |
| `drwx------` | Home directory permission — only owner and root can access |
| `/etc/skel` | Template directory — files copied to every new user's home on creation |

---

## ⚠️ Pitfalls

- **Using `userdel`** → deletes the account entirely — never use this when the requirement says "keep intact"
- **Using `usermod -L`** → only locks the password; SSH key-based login still works — not a full block
- **Testing with `su -s /bin/bash tom`** → overrides the shell; not a valid login test — always use `su - tom`
- **Confusing `/sbin/nologin` with `/bin/false`** → both block login but `/sbin/nologin` prints a message; either is acceptable on the exam
- **No password set** → `su - tom` fails at authentication before reaching the nologin check — set a password first with `sudo passwd tom`
- **`ls /home/tom` without sudo** → returns `Permission denied` — use `sudo ls -la /home/tom` to verify files as root

---

## ✅ Lab Checklist

- `getent passwd tom` shows `/bin/bash` as starting shell ✓
- `usermod -s /sbin/nologin tom` applied ✓
- `getent passwd tom` shows `/sbin/nologin` as updated shell ✓
- `id tom` confirms account still exists ✓
- `sudo ls -la /home/tom` confirms files intact ✓
- `sudo passwd tom` sets a password for testing ✓
- `su - tom` returns `This account is currently not available.` ✓

---

## 🔗 Related Labs

- [User & Group Management / Permissions](https://github.com/kelvintechnical/User-Group-Management-Permissions)
- [Full RHCSA/RHCE Study Guide →](https://github.com/kelvintechnical/linux-ops-mastery)

---

## 👤 Author

**Kelvin R. Tobias** — [kelvinintech.com](https://kelvinintech.com) | [GitHub](https://github.com/kelvintechnical) | [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
