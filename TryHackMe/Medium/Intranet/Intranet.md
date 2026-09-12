# Intranet

**Difficulty:** 🟡 Medium · **Room:** [TryHackMe ↗](https://tryhackme.com/room/support)

The web application development company SecureSolaCoders has created their own intranet page. The developers are still very young and inexperienced, but they ensured their boss (Magnus) that the web application was secured appropriately. The developers said, "Don't worry, Magnus. We have learnt from our previous mistakes. It won't happen again". However, Magnus was not convinced, as they had introduced many strange vulnerabilities in their customers' applications earlier.

Magnus hired you as a third-party to conduct a penetration test of their web application. Can you successfully exploit the app and achieve root access?

---
## Reconnaissance

I started with simple `nmap` scan, to see the whole attack surface:
![](basic_nmap.png)
So there is `http` both on port `80` and `8080`. `FTP` runs on port `21`, `ssh` on `22`, `telnet` on `23` and `echo` on port `7`. The `echo` service stands out, because it is not used widely nowadays. I followed up with more detailed `nmap` scan, to gather more information:
![](detailed_nmap.png)
Results showed, that the `MACHINE_IP:8080/login` endpoint exist, so I visited it. And the login page appeared:
![](TryHackMe/Medium/Intranet/images/login_page.png)
The placeholder in `e-mail` field disclosed the it's format:
```text
firstname@securesolacoders.no
```
I thought about login brute-forcing to access an employ account. But first I needed to gather the employs names. The first name is `Magnus` who is the boss of the company. The second one, `anders` along with `devops@securesolacoders.no` e-mail were found by inspecting the source code of login page:
```text
<!--- Any bugs? Please report them to our developer team. We have an open bug bounty program! For any inquiries, contact devops@securesolacoders.no. Sincerely, anders (Senior Developer) -->
```
The directory enumeration using `feroxbuster` didn't find anything interesting. The login page had also another weakness: it revealed if particular e-mail existed or not, by replying to invalid login with either `Ivalid password` or `Ivanlid username`. Knowing that, I first generated a list of possible emails. To achieve the best result I had to use a large list of names, like `names.txt` located in `seclists`. This python script helps with generating a list of possible e-mails:
```python
emails = open('emails.txt', 'w')
with open('names.txt', 'r') as file:
        for name in file:
                name = name.strip()
                emails.write(f"{name}@securesolacoders.no\n")
emails.close()
```
So with `emails.txt` list I used `hydra` enumerate valid emails. `Burp suite` showed that the request used `username` and `password` parameters. I run `hydra` testing for valid emails:
```bash
hydra -L emails.txt -p 123 -u -e sr -s 8080 -m '/login:username=^USER^&password=^PASS^:Invalid username' MACIHNE_IP http-post-form
```
Hydra found only one additional valid email. After all, these were valid e-mails that I found:
```text
anders@securesolacoders.no
devops@securesolacoders.no
admin@securesolacoders.no
```
Next I run another `hydra` command, but this time with purpose to find users's passwords:
```bash
hydra -L users.txt -P /usr/share/wordlists/seclists/Passwords/Common-Credentials/10k-most-common.txt  -u -e sr -s 8080 -m '/login:username=^USER^&password=^PASS^:Invalid password' MACHINE_IP http-post-form
```
But this produced only few `false positives` (due to password policy, signs like `'` , `&` , `"`, `;`or `#` weren't allowed). Since the brute-forcing with common passwords lists doesn't work, maybe I have to generate password list myself? I used `cupp` to generate possible passwords, including company name for `admin`, `devops` and `anders`.  I generated password list using `john`:
```text
base_words.txt:
admin
anders
devops
SecureSolaCoders
```
```bash
john --wordlist=base_words.txt --rules=All --stdout > passwords.txt
```
And run hydra again. And it succeeded with `anders` password an first flag:
![](anders_pass.png)
![](TryHackMe/Medium/Intranet/images/first_flag.png)
For fully authenticate I had to pass the 4-digit code because 2FA was enabled on the portal. However the code appeared to be static, there was no information if the code expires. So like password and e-mails, I also tried to brute force it, but this time using `ffuf` as it allowed to transmit headers. First I generated list of all possible 4-digit codes with this python script:
```python
codes = open("codes.txt", "w")
code = ['0','0','0','0']
for first in range(10):
        for second in range(10):
                for third in range(10):
                        for fourth in range(10):
                                code[0]=str(first)
                                code[1]=str(second)
                                code[2]=str(third)
                                code[3]=str(fourth)
                                codes.write("".join(code)+'\n')
codes.close()
```
After that I run `ffuf` with `-H` parameter for `HTTP headers`, `-b` for `session` cookie and `-fs` to filter the wrong code response size, because every request returns `200` code. After few seconds it  successfully found valid code:
```bash
ffuf -w codes.txt \
  -u http://10.82.179.220:8080/sms \
  -X POST \
  -b 'session=eyJ1c2VybmFtZSI6ImFuZGVycyJ9.aqEUrA.8DD79-appHPIm8l9iOeBJStdoMQ' \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'Referer: http://10.82.179.220:8080/sms' \
  -d "sms=FUZZ" \
  -fs=1326
```
![](valid_code.png)
Now I could fully authenticate as `anders` and obtain second flag:
![](TryHackMe/Medium/Intranet/images/second_flag.png)
While browsing through the portal I found four other e-mail addresses:
```text
external@securesolacoders.no
internal@securesolacoders.no
hiring@securesolacoders.no
support@securesolacoders.no
```
I thought about trying to brute-force passwords for those accounts, but the login page produces `Invalid username` massage for those addresses. I decided to inspect a `session` cookie. And it was a good decision, because the `session` was in reality `Flask session cookie`:
![](cookie_flask.png)
`Hashcat` quickly found a proper signature value.
![](cookie_key.png)
