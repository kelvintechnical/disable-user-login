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

---

## 📚 Command Decision Map

| Lab Phrase | Question Being Asked | Tool |
|------------|---------------------|------|
| "Prevent login" | How do I block a user from logging in? | `usermod -s /sbin/nologin` |
| "Without deleting" | How do I keep account intact? | Don't use `userdel` — change the shell only |
| "Account must remain" | How do I verify account still exists? | `getent passwd tom` |
| "Files must remain" | How do I confirm files are untouched? | `ls -la /home/tom` |

---

## 🧠 Big Concept — Login Shell

Every user account has a **login shell** defined in `/etc/passwd`. This is the program that runs when a user logs in interactively:

```
tom:x:1001:1002::/home/tom:/bin/bash   ← normal login shell
tom:x:1001:1002::/home/tom:/sbin/nologin  ← login blocked
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

## Step 1 — Disable login shell

```bash
sudo usermod -s /sbin/nologin tom
```

> `usermod` modifies an existing user account. `-s` sets the login shell. `/sbin/nologin` blocks interactive login immediately.

---

## Step 2 — Verify the shell was changed

```bash
getent passwd tom
```

**Expected output:**
```
tom:x:1001:1002::/home/tom:/sbin/nologin
```

The last field changed from `/bin/bash` to `/sbin/nologin`.

---

## Step 3 — Confirm account and files are intact

```bash
# Account still exists
id tom

# Home directory and files untouched
ls -la /home/tom
```

---

## Step 4 — Verify login is actually blocked

```bash
sudo su -s /bin/bash tom -c "echo logged in"
# This works because su overrides the shell — not a true login test

# True test — attempt login as tom
su - tom
# Expected: This account is currently not available.
```

---

## 🧠 Key Concepts

| Concept | What it means |
|---------|--------------|
| Login shell | The program in `/etc/passwd` that runs at interactive login |
| `/sbin/nologin` | A binary that blocks login and prints a message — account stays intact |
| `/etc/passwd` format | `username:x:uid:gid:comment:home:shell` — shell is the last field |
| `usermod -s` | Changes the login shell for an existing user |
| `usermod -L` | Locks the password — does NOT change the shell |
| `getent passwd` | Reads `/etc/passwd` — confirms current shell setting |

---

## ⚠️ Pitfalls

- **Using `userdel`** → deletes the account entirely — never use this when the requirement says "keep intact"
- **Using `usermod -L`** → only locks the password; SSH key-based login still works — not a full block
- **Checking with `su -s /bin/bash tom`** → overrides the shell; not a valid login test — use `su - tom`
- **Confusing `/sbin/nologin` with `/bin/false`** → both block login but `/sbin/nologin` prints a message; either is acceptable on the exam

---  

## ✅ Lab Checklist

- `usermod -s /sbin/nologin tom` applied ✓
- `getent passwd tom` shows `/sbin/nologin` as shell ✓
- `id tom` confirms account still exists ✓
- `ls -la /home/tom` confirms files intact ✓
- `su - tom` returns `This account is currently not available.` ✓

---

## 🔗 Related Labs

- [User & Group Management / Permissions](https://github.com/kelvintechnical/linux-ops-mastery/blob/master/01-system-management/user-group-permissions.md)
- [Full RHCSA/RHCE Study Guide →](https://github.com/kelvintechnical/linux-ops-mastery)

---

## 👤 Author

**Kelvin R. Tobias** — [kelvinintech.com](https://kelvinintech.com) | [GitHub](https://github.com/kelvintechnical) | [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
