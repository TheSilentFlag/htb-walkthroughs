# Information Gathering (Web Recon)

<br>

A practical write-up documenting the reconnaissance and enumeration workflow for information gathering. This lab highlights methodology, troubleshooting wordlist discrepancies, and leveraging automation tools for content discovery.

<br>

---
<br>

## Objective & Scope

<br>

The goal of this assessment is to perform thorough web reconnaissance on the target domain running on a dynamic high-level port. Key objectives include:

* Identifying basic domain registration details
* Fingerprinting the web server technology
* Discovering hidden virtual hosts and subdomains
* Uncovering sensitive files, hidden directories, emails, and API keys

<br>

## Initial Reconnaissance & Footprinting

<br>

First, we need to add our target's IP address to `/etc/hosts` file, so that we can resolve the domain name to the given IP:

<br>

```bash
sudo vim /etc/hosts
```

<img width="634" height="282" alt="image" src="https://github.com/user-attachments/assets/3db211cb-d63f-49c6-b4be-49275aa940a9" />

<br>

<br>

We start by gathering basic information about the target domain and checking the HTTP headers to identify the underlying server software and port configuration.

<br>

To retrieve IANA ID of the inlanefreight.com we will use whois:

<br>

```bash
whois inlanefreight.com
```

<br>

<img width="1053" height="421" alt="image" src="https://github.com/user-attachments/assets/42b5d5a1-55e9-4ef4-9818-f4e456c96e03" />

<br>

<br>

To inspect server headers we will use banner grabbing technique:

<br>

```bash
curl -s -I http://inlanefreight.htb:30490
```

<br>

<img width="555" height="354" alt="image" src="https://github.com/user-attachments/assets/44107bec-fede-42be-ac01-1acddc79699d" />

<br>

<br>

## VHost Enumeration

<br>

The target host has only a title and nothing else available to interact with:

<br>

<img width="700" height="344" alt="image" src="https://github.com/user-attachments/assets/0de1c802-dcf7-4ef1-8061-91f6d44fc3e2" />

<br>

<br>

Robots.txt file does not exist on the target host:

<br>

<img width="704" height="109" alt="image" src="https://github.com/user-attachments/assets/d69b3407-ea8e-4d38-bc5f-6f677402b255" />

<br>

<br>

We need to find the API key in the hidden admin directory. /admin directory yields nothing to us, so what can we do next? The answer is Vhost fuzzing.

<br>

*Why VHost Fuzzing?*

<br>

When dealing with target environments we connect directly via an IP, but public DNS servers know nothing about domains like `.htb`. Running traditional DNS queries (`dig`, `nslookup`) will yield nothing. When multiple websites / subdomains are hosted on a single IP address and port - we are dealing with Name-Based Virtual Hosting. 

<br>

The web server uses the HTTP `Host:` header to determine which site to serve. Since standard tools don't know these subdomains exist, we must use VHost fuzzing — systematically brute-forcing the `Host:` header using a wordlist to discover valid virtual hosts.

<br>

We can use gobuster to find hidden subdomains associated with the main application:

<br>

```bash
gobuster vhost -u http://inlanefreight.htb:30490 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 100 --append-domain
```

<br>

<img width="634" height="237" alt="image" src="https://github.com/user-attachments/assets/f1d1ace2-a90b-44b7-8425-2a0a06bb5bb9" />

<br>

<br>

We find the hidden virtual host `web1337.inlanefreight.htb`

<br>

Add this Vhost to /etc/hosts:

<br>

<img width="633" height="274" alt="image" src="https://github.com/user-attachments/assets/3d7cf90b-059d-422a-b314-093645dcc464" />

<br>

<br>

Now we can dig deeper

<br>


## Directory Enumeration & Hidden Content

<br>

Accessing `http://web1337.inlanefreight.htb`:

<br>

<img width="704" height="142" alt="image" src="https://github.com/user-attachments/assets/3aa27420-b297-4515-9eb4-a5f79b2e8990" />

<br>

<br>

We can read robots.txt now:

<br>

```bash
curl web1337.inlanefreight.htb:30490/robots.txt
```

<br>

The file revealed a hidden administrative directory:

<br>

<img width="607" height="172" alt="image" src="https://github.com/user-attachments/assets/9b7545d3-6d2e-4ec1-aeee-d0bb8b24b680" />

<br>

<br>

And the API key:

<br>

<img width="1263" height="126" alt="image" src="https://github.com/user-attachments/assets/31c15de6-0383-497b-9fc8-2dbf97b47698" />

<br>

<br>

Pages `/index.html`, `/index-2.html` and `/index-3.html` do not exist on the target vhost web1337.inlanefreight.htb. There are no other clues for us, so we can keep fuzzing:

<br>

```bash
gobuster vhost -u http://web1337.inlanefreight.htb:30965 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 100 --append-domain
```

<br>

Deeper enumeration exposed an additional developer subdomain:

<br>

<img width="630" height="229" alt="image" src="https://github.com/user-attachments/assets/c7ab23c5-50e6-4e27-99d2-f4f8e90f74b8" />

<br>

<br>

We add this domain to our `/etc/hosts` file and we will explore this vhost

<br>

## Automation & Crawling

<br>

The `dev` subdomain presented identical, visually empty pages. Manual inspection is unfeasible here.

<br>

<img width="704" height="155" alt="image" src="https://github.com/user-attachments/assets/52c0f9d2-e0d4-4f9a-ad9d-5566ffb67d6a" />

<br>

<br>

We are looking for email addresses and the new API keys. 

<br>

To efficiently parse the entire directory structure without manual clicking, we will use `ReconSpider` tool to crawl the target and extract comments, emails, and links:

<br>

```bash
python3 ReconSpider.py http://dev.web1337.inlanefreight.htb:30965
```

<br>

<img width="628" height="217" alt="image" src="https://github.com/user-attachments/assets/3897b0bb-67e0-4a13-9df2-fe0a0733573b" />

<br>

<br>

`results.json` file contains our findings. We can access the fields we are interested in using `jq`:

<br>

```bash
jq ".emails" results.json
```

<br>

<img width="387" height="123" alt="image" src="https://github.com/user-attachments/assets/7dbf9726-8630-4bc0-9a60-f382fb483d6e" />

<br>

<br>

The API key is in the `comments` section of our file:

<br>

```bash
jq ".comments" results.json
```

<br>

<img width="892" height="124" alt="image" src="https://github.com/user-attachments/assets/78fe432e-6598-4066-a7e8-a8fc9f9ba855" />

<br>

<br>


## Key Takeaways:

<br>

* *Verify Your Wordlists:* During the VHost enumeration phase, initial scans yielded nothing because the default packaged wordlist lacked the specific entries. Recognizing this discrepancy, I searched for a more comprehensive upstream version `subdomains-top1million-110000.txt`, applied it, and successfully resolved the target. Always verify your wordlist scope if scans return empty results.

<br>

* *Embrace automation for scale:* Do not rely only on manual browsing. Automated crawlers can save you hours of manual work and reduce human prone errors.

<br>


* *Reconnaissance is everything:* As this lab demonstrated, if we perform thorough reconnaissance and gather sufficient intelligence, the actual exploitation takes just a couple of commands. If we know **what we are looking for** and **where to find it**. This is why high-quality recon is always the most critical phase.

<br>

* **The Golden Rule of penetration testing process:**

<br>

> If you are stuck, return to information gathering. If you don't know what to do next, return to information gathering and dig deeper.

<br>

<img width="2048" height="771" alt="image" src="https://github.com/user-attachments/assets/4443e8bb-670c-4b4c-8cbc-3d14d342e8dc" />

<br>

<br>

Thorough enumeration and proper tooling solve 90% of blocked paths in web assessments.
