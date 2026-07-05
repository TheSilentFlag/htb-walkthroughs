We have the following task:

<br>
<img width="614" height="158" alt="image" src="https://github.com/user-attachments/assets/59da5fea-8fe6-4ea4-98a1-e5507f4946c4" />

<br>

And the given IP address: 154.57.164.82:32215 

Note: The address might change during the writeup due to the target respawn

--------------------------------------



Let's open this address in a browser to see what we will get:

<br>
<img width="855" height="173" alt="image" src="https://github.com/user-attachments/assets/e3923342-3ec6-4b61-b9a5-f60158fdc545" />
<br>

<br>

# Directory Fuzzing

We do not see anything interesting. Now let's start recursive fuzzing with ffuf:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -ic -v -u http://154.57.164.82:32215/FUZZ -recursion -rate 500

```

<br>

This is what we get:

<br>

```bash
[Status: 301, Size: 323, Words: 20, Lines: 10, Duration: 60ms]
| URL | http://154.57.164.82:32215/<secret_directory>
| --> | http://154.57.164.82:32215/<secret_directory>/
    * FUZZ: <secret_directory>

[INFO] Adding a new job to the queue: http://154.57.164.82:32215/<secret_directory>/FUZZ

<snip>

[Status: 200, Size: 13, Words: 2, Lines: 1, Duration: 59ms]
| URL | http://154.57.164.82:32215/<secret_directory>/index.php
    * FUZZ: index.php

```

<br>

Now we can perform file searching in /<secret_directory> directory to see what we can find. We will use ffuf again:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://154.57.164.82:32215/<secret_directory>/FUZZ -e .php,.html,.txt,.bak,.js -v 
```

<br>

We got a `<target_file>.php` file that's worth to check it:

<br>

```bash
[Status: 200, Size: 13, Words: 2, Lines: 1, Duration: 61ms]
| URL | http://154.57.164.82:32215/<secret_directory>/index.php
    * FUZZ: index.php

[Status: 200, Size: 13, Words: 2, Lines: 1, Duration: 61ms]
| URL | http://154.57.164.82:32215/<secret_directory>/index.php
    * FUZZ: index.php

[Status: 200, Size: 58, Words: 8, Lines: 1, Duration: 61ms]
| URL | http://154.57.164.82:32215/<secret_directory>/<target_file>.php
    * FUZZ: <target_file>.php

:: Progress: [28500/28500] :: Job [1/1] :: 666 req/sec :: Duration: [0:00:46] :: Errors: 0 ::
```
<br>

Let's curl this URL to see the server's response:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ curl http://154.57.164.82:32215/<secret_directory>/<target_file>.php

Invalid parameter, please ensure ac<...> is set correctly
```

<br>

Here there is a hint from the server: it expects from the client (from us) the parameter ac<...> to be set correctly.
We can fuzz this parameter's value to find the one that exists. Let's try GET fuzzing first:

<br>

```bash
[user@parrot]─[~]
└──╼ $ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -fc 404 -u "http://154.57.164.61:31046/<secret_directory>/<target_file>.php?ac<...>=FUZZ"
```

<br>

We get this:

<br>

```bash
...<snip>

~operator               [Status: 200, Size: 58, Words: 8, Lines: 1, Duration: 60ms]
~mail                   [Status: 200, Size: 58, Words: 8, Lines: 1, Duration: 61ms]
~nobody                 [Status: 200, Size: 58, Words: 8, Lines: 1, Duration: 62ms]
~lp                     [Status: 200, Size: 58, Words: 8, Lines: 1, Duration: 63ms]
~sysadm                 [Status: 200, Size: 58, Words: 8, Lines: 1, Duration: 60ms]

...<snip>
```
<br>

We see that the default response has Size 58, so we can filter them out:

<br>

```bash
[user@parrot]─[~]
└──╼ $ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs 58 -u "http://154.57.164.61:3
1046/<secret_directory>/<target_file>.php?ac<...>=FUZZ"
```

<br>

Now we have the parameter's value:

<br>

```bash
ge<...>               [Status: 200, Size: 68, Words: 12, Lines: 1, Duration: 61ms]
:: Progress: [4750/4750] :: Job [1/1] :: 649 req/sec :: Duration: [0:00:09] :: Errors: 0 ::
```

<br>

Let's curl the full URL to get the flag:

<br>

```bash
[user@parrot]─[~]
└──╼ $curl http://154.57.164.61:31046/<secret_directory>/<target_file>.php?ac<...>=ge<...>
```

<br>

And we get the response:

<br>

<img width="483" height="37" alt="image" src="https://github.com/user-attachments/assets/540624ff-1296-473f-9696-55b3ac9edc38" />

<br>

<br>

```bash
[user@parrot]─[~]
└──╼ $Head on over to the <target_vhost>.htb vhost for some more fuzzing fun!
```

<br>

# Vhost Fuzzing

We haven't found the flag yet. We need to fuzz vhost. First, we need to add this DNS name to our /etc/hosts file by this command:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $echo "<target_IP> <target_vhost>.htb" | sudo tee -a /etc/hosts
```
<br>

Let's fuzz vhost using gobuster and grep only statuses 200 OK:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $gobuster vhost -u http://<target_vhost>:<PORT> -w /usr/share/seclists/Discovery/Web-Content/common.txt --append-domain | grep "Status: 200"
```

<br>

We get:

<br>

<img width="555" height="49" alt="image" src="https://github.com/user-attachments/assets/71caa894-7a38-4a41-911a-0606b3de6608" />

<br>

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $gobuster vhost -u http://<target_vhost>:<PORT> -w /usr/share/seclists/Discovery/Web-Content/common.txt --append-domain | grep "Status: 200"

Found: <hidden_subdomain>.<target_domain>:<PORT> Status: 200 [Size: 45]
Progress: 4750 / 4750 (100.00%)
```

<br>

Now we need to add this vhost <hidden_subdomain>.<target_domain>.htb to our /etc/hosts using the same IP address:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $echo "<target_IP> <hidden_subdomain>.<target_domain>.htb" | sudo tee -a /etc/hosts
```

<br>

Let's curl this: 

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $curl <hidden_subdomain>.<target_domain>.htb:<PORT>
```

<br>

We get:

<br>

<img width="251" height="38" alt="image" src="https://github.com/user-attachments/assets/40dfd46a-6574-47b3-bf86-19b1406d0bfb" />

<br>

<br>

Let's add `/godeep` to our request:

<br>

<img width="262" height="36" alt="image" src="https://github.com/user-attachments/assets/b1648b7c-f9b6-4877-96c4-a3f40459d9ab" />

<br>

<br>

Keep fuzzing the new directory:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://<hidden_subdomain>.<target_domain>.htb:<PORT>/godeep/FUZZ
```

<br>

We get:

<br>

```bash
.htaccess               [Status: 403, Size: 290, Words: 20, Lines: 10, Duration: 62ms]
.hta                    [Status: 403, Size: 290, Words: 20, Lines: 10, Duration: 64ms]
.htpasswd               [Status: 403, Size: 290, Words: 20, Lines: 10, Duration: 3897ms]
index.php               [Status: 200, Size: 13, Words: 2, Lines: 1, Duration: 61ms]
<new_target_directory>               [Status: 301, Size: 352, Words: 20, Lines: 10, Duration: 59ms]
:: Progress: [4750/4750] :: Job [1/1] :: 657 req/sec :: Duration: [0:00:10] :: Errors: 0 ::
```

<br>

Let's curl it again:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $curl <hidden_subdomain>.<target_domain>.htb:<PORT>/godeep/<new_target_directory>

Almost there...
```
<br>

<img width="323" height="36" alt="image" src="https://github.com/user-attachments/assets/f7b146ec-e81e-4b63-a11e-f0cf637c28ae" />

<br>

<br>

Keep fuzzing:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://<hidden_subdomain>.<target_domain>.htb:<PORT>/godeep/<new_target_directory>/FUZZ
```

<br>

We get:

<br>

```bash
.hta                    [Status: 403, Size: 290, Words: 20, Lines: 10, Duration: 3646ms]
<target_directory_2>                 [Status: 301, Size: 360, Words: 20, Lines: 10, Duration: 62ms]
.htpasswd               [Status: 403, Size: 290, Words: 20, Lines: 10, Duration: 4648ms]
.htaccess               [Status: 403, Size: 290, Words: 20, Lines: 10, Duration: 4656ms]
index.php               [Status: 200, Size: 15, Words: 2, Lines: 1, Duration: 60ms]
:: Progress: [4750/4750] :: Job [1/1] :: 655 req/sec :: Duration: [0:00:10] :: Errors: 0 ::
```
<br>

<img width="506" height="85" alt="image" src="https://github.com/user-attachments/assets/dfa1df05-ce5d-4e02-87e0-e1fff0b7f56f" />

<br>

<br>

Curl it again:

<br>

```bash
curl http://<hidden_subdomain>.<target_domain>.htb:<PORT>/godeep/<new_target_directory>/<target_directory_2>/

Just a bit more...
```

<br>

<img width="358" height="35" alt="image" src="https://github.com/user-attachments/assets/1faaf23f-c8aa-4317-b005-ce733c9f9613" />

<br>

<br>

Keep fuzzing:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://<hidden_subdomain>.<target_domain>.htb:<PORT>/godeep/<new_target_directory>/<target_directory_2>/FUZZ
```

<br>

Finally we get:

<br>

```bash
.htaccess               [Status: 403, Size: 290, Words: 20, Lines: 10, Duration: 62ms]
.hta                    [Status: 403, Size: 290, Words: 20, Lines: 10, Duration: 3037ms]
.htpasswd               [Status: 403, Size: 290, Words: 20, Lines: 10, Duration: 5061ms]
index.php               [Status: 200, Size: 18, Words: 4, Lines: 1, Duration: 61ms]
<final_directory>                   [Status: 301, Size: 366, Words: 20, Lines: 10, Duration: 60ms]
:: Progress: [4750/4750] :: Job [1/1] :: 649 req/sec :: Duration: [0:00:11] :: Errors: 0 ::
```

<br>

<img width="506" height="82" alt="image" src="https://github.com/user-attachments/assets/2520cfbe-e0bf-4eec-859e-8c1f5ae3c604" />

<br>

<br>

Let's curl it again:

<br>

```bash
┌─[user@parrot]─[~]
└──╼ $curl http://<hidden_subdomain>.<target_domain>.htb:<PORT>/godeep/<new_target_directory>/<target_directory_2>/<final_directory>

HTB{...}
```

<br>

And we get the flag:

<br>

<img width="393" height="39" alt="image" src="https://github.com/user-attachments/assets/5ed56768-7353-459a-9b5d-61a31c1c4aa7" />

<br>

