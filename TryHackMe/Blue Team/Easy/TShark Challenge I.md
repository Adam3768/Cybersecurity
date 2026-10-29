# TShark Challenge I : Teamwork

**Difficulty:** 🟢 Easy · **Room:** [TryHackMe ↗](https://tryhackme.com/room/tsharkchallengesone)

An alert has been triggered: "The threat research team discovered a suspicious domain that could be a potential threat to the organisation."

The case was assigned to me. I inspected the provided `teamwork.pcap`, located in `~/Desktop/exercise-files`, and created artefacts for detection tooling.

---

## Q1: What is the full URL of the malicious/suspicious domain address?

Command:

```bash
tshark -r teamwork.pcap -T fields -e dns.qry.name | sort | uniq
```

Breakdown:

- `-r teamwork.pcap` — loads the capture file
- `-T fields` — tells `tshark` to print only specified fields instead of full packets
- `-e dns.qry.name` — prints the `dns.qry.name` field, which holds the domain name requested in each DNS query
- `| sort | uniq` — sorts the results alphabetically and removes duplicates

Result:

![](question_one_result.png)

The last result looked very suspicious. It tries to impersonate `paypal.com`, but the domain actually has two `.com`-style endings stitched together, with the real registered domain being something else entirely:

```
www.paypal.com4uswebappsresetaccountrecovery.timeseaways.com
```

I defanged the URL using CyberChef before handling it any further, to avoid accidentally visiting it.

---

## Q2: When was the URL of the malicious/suspicious domain address first submitted to VirusTotal?

I visited virustotal.com and pasted the malicious URL. Then, under **Details → History**, the first submission date was visible:

![](question_two.png)

---

## Q3: Which known service was the domain trying to impersonate?

The answer sits right in the URL itself — the domain is built around the word `paypal`, impersonating **PayPal**.

---

## Q4: What is the IP address of the malicious domain?

Command:

```bash
tshark -r teamwork.pcap -Y 'dns.qry.name == "www.paypal.com4uswebappsresetaccountrecovery.timeseaways.com" && dns.flags.response == 1' -T fields -e dns.a | sort | uniq
```

Breakdown:

- `-Y '...'` — applies a display filter instead of a capture filter, letting me query already-captured packets
- `dns.qry.name == "..."` — matches only DNS packets concerning the malicious domain
- `dns.flags.response == 1` — keeps only DNS **responses**, since that's where the resolved IP address actually appears
- `-e dns.a` — prints the resolved IPv4 address from those responses

Result:

![](question_trhee.png)

---

## Q5: What is the email address that was used?

Command:

```bash
tshark -r teamwork.pcap -Y 'http.request.method == "POST"' -T fields -e http.file_data
```

Breakdown:

- `-Y 'http.request.method == "POST"'` — filters for HTTP POST requests, since that's typically how submitted form data (like a login form) gets sent
- `-e http.file_data` — prints the actual body/payload of those POST requests, which is where submitted form fields show up

Result:

![](answ.png)

So, sadly, the user behind this traffic had become a victim of phishing — their email address and password were submitted directly to the fake PayPal login page.

---

## Summary

This case started from a single alert about a suspicious domain, and `tshark` filtering made it straightforward to build out the full picture from the capture alone. DNS query enumeration surfaced a domain impersonating PayPal through a deceptive, doubled `.com` structure, and VirusTotal confirmed it had already been flagged by the community. From there, resolving the malicious domain's IP and inspecting POST request bodies in the capture confirmed that a user had submitted their email address to the phishing page — a successful credential-phishing incident, caught after the fact through traffic analysis.

> Thanks for reading!