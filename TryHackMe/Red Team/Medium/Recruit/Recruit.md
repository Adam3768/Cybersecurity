# Recruit

**Difficulty:** 🟡 Medium · **Room:** [TryHackMe ↗](https://tryhackme.com/room/recruitwebchallenge)

**Recruit** has just launched its new recruitment portal, allowing HR staff to manage candidate applications and administrators to oversee hiring decisions. While the platform appears functional, management suspects that security may have been overlooked during development. Your task is to assess the application like a real attacker, mapping its structure, abusing exposed functionality, and exploiting vulnerabilities.

Can you gain an initial foothold, escalate your access, and ultimately log in as the **administrator**?

---

## Reconnaissance

I started with a TCP connect scan and default script enumeration to see the attack surface:

```bash
nmap -sT -sC TARGET-IP
```

**Results:**

| Port | Service |
|:--:|---|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |

![Nmap Scan](screenshots/nmap_scan.png)

With HTTP as the obvious target, I focused the rest of my enumeration on the web application.

A directory brute-force against the web root turned up content that wasn't linked anywhere in the app itself:

![Directory Enumeration](screenshots/directory_bruteforce.png)

That's how I found an exposed log file:

```
http://TARGET-IP/mail/mail.log
```

The log gave me two useful things: a valid application username, **hr**, and a reference to credentials being stored in `config.php` — a strong hint that a file disclosure bug was the next step.

![Mail Log](screenshots/mail_info.png)

---

## Initial Access

Poking around further, I found an API endpoint:

```
http://TARGET-IP/api.php
```

This showed that a candidate's CV was fetched through a separate endpoint:

```
/file.php?cv=<URL>
```

![API Endpoint](screenshots/fetch_info.png)

Passing local paths and stream wrappers into the `cv` parameter confirmed a **Local File Inclusion (LFI)** vulnerability — the endpoint wasn't restricting retrieval to remote or expected file locations at all.

Using the `file://` wrapper, I requested the config file the log had pointed me toward:

```
http://TARGET-IP/file.php?cv=file:////var/www/html/config.php
```

The response gave up valid application credentials in plaintext.

![Retrieved Credentials](screenshots/hr_pass.png)

I used those to log in as **hr**, which got me the **User flag**.

![User Flag](screenshots/user_flag.png)

---

## SQL Injection

Once logged in as `hr`, I turned my attention to the app's internal search feature, which looked like it was querying the database directly.

Testing the search parameter with standard SQL injection payloads confirmed it was vulnerable to **UNION-based SQL Injection**.

![SQL Injection Confirmed](screenshots/sql_injection_confirmed.png)

Before I could pull any data, I needed to figure out how many columns the underlying query returned, so I could build a matching `UNION SELECT`:

![UNION Enumeration](screenshots/sql_injection_union.png)

With that sorted, I used the injection point to enumerate the database schema, starting with the available tables:

![Database Tables](screenshots/tables_names.png)

The `users` table looked like the obvious place to find login data, so I enumerated its columns next:

![Users Table Columns](screenshots/columns_names.png)

With the table and column names in hand, I extracted the administrator credentials directly from `users`:

![Administrator Credentials](screenshots/admin_pass.png)

Logging in with those credentials gave me administrator access and the **Admin flag**, completing the room.

![Admin Flag](screenshots/admin_flag.png)

---

## Kill Chain

This box came down to two separate bugs that each did half the work. Directory brute-forcing turned up a log file that was never meant to be public, and that log pointed straight at a config file holding real credentials — I just needed a way to actually read it. The application's own CV-fetching feature provided that, since it accepted a `file://` path with no restriction at all.

Getting from a regular HR account to full admin was a completely separate issue: an internal search feature built its SQL queries by concatenating user input directly, so a standard UNION-based injection was enough to read straight out of the `users` table.

`Anonymous visitor → hr (via LFI) → administrator (via SQL injection)`

- **Initial Access:** Exposed log file → disclosed config path → LFI via the CV-fetch endpoint → plaintext credentials → login as `hr`
- **Privilege Escalation:** UNION-based SQL injection in the search feature → database and table enumeration → extracted administrator credentials → full admin access

Neither vulnerability here was particularly advanced — an unrestricted file-fetch parameter and an unsanitized search query — but between them they took an anonymous visitor all the way to administrator.

> Thanks for reading!