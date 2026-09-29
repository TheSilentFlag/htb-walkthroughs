# Command Injection: Bypassing Blacklists & Executing Code

<br>

## Objective & Scope

<br>

The objective of this assessment is to analyze and evaluate the security posture of the File Manager interface. The primary goal is to identify and exploit command injection vulnerabilities to gain unauthorized command execution and retrieve sensitive data from the underlying system.

---

## Initial Reconnaissance & Information Gathering

<br>

We are provided with `username` and `password` to authenticate to the app:

<br>

<img width="699" height="363" alt="image" src="https://github.com/user-attachments/assets/86dde788-57df-4aee-abc4-abde2ac959d6" />

<br>

<br>

Once logged in, we can perform some actions with available files. We can `copy`, `move`, `view` and `download`:

<br>

<img width="1396" height="694" alt="image" src="https://github.com/user-attachments/assets/0ba775e0-4596-4075-8e43-875604b73447" />

<br>

<br>

tmp directory is empty. If we look closely at the URL, we will notice a GET parameter`to=`. When we want to move any file to `tmp` folder, `to=` appears again:

<br>

<img width="696" height="214" alt="image" src="https://github.com/user-attachments/assets/59e64b58-54e2-4fc6-9757-c310d6e24cd9" />

<br>

<br>

<img width="1398" height="462" alt="image" src="https://github.com/user-attachments/assets/1027ee25-7ded-4175-9900-7c35606af09d" />

<br>

<br>

The `move` request looks as follows:

<br>

<img width="959" height="362" alt="image" src="https://github.com/user-attachments/assets/98b77691-ee48-481d-88ed-30bc8b83692a" />

<br>

<br>

Now `tmp` folder contains this file:

<br>

<img width="699" height="214" alt="image" src="https://github.com/user-attachments/assets/04f5f2f8-18f9-4f08-a2f4-6a7b36ba0fa4" />

<br>

<br>

At this point pay attention to the text we see before we move any file to `tmp` folder and the URL:

<br>

```plaintext
Copying

Source path: /var/www/html/files/605311066.txt
Destination folder: /var/www/html/files/tmp 
```

<br>

<img width="1012" height="363" alt="image" src="https://github.com/user-attachments/assets/b0b5af64-9290-4410-ab48-268020a622ef" />

<br>

<br>


It helps us to understand the logic of the command. This is what happens under the hood:

<br>

```bash
mv  /var/www/html/files/<from> /var/www/html/files/<to>
```

<br>

Even though the `to` parameter comes first in the URL request, it will be the last piece of the `mv` command as shown before. It is always easier to inject our command in an input going at the end of the command, rather than in the middle of it, so we will use `to` parameter for command injection attack

<br>


## Vulnerability Assessment

<br>

Once we gathered enough information about the application and how it works, we can start our assessment. At this point we have to understand the difference between:

<br>

* Was our payload filtered by the WAF (if the application has this protection)?
* Was our payload filtered by the back-end filters?
* Did our payload break the original command?

<br>

When we try to move the file we have already moved, an error occurs:

<br>

<img width="1915" height="724" alt="image" src="https://github.com/user-attachments/assets/0768f812-36a4-4513-a5cf-50b07ce4ee0d" />

<br>

<br>

Changing the payload to `&&` or `%26%26` URL-Encoded caused a different error:

<br>

<img width="959" height="332" alt="image" src="https://github.com/user-attachments/assets/7ed14d26-62de-4733-8478-9741f25e8ffd" />

<br>

<br>

Our input goes directly to the `mv` command and now we have to escape from `mv` and inject any arbitrary command to see how it works. Let's start with `ls` command and `\n` as a command separator, because `;` symbol might be blacklisted:

<br>

<img width="959" height="335" alt="image" src="https://github.com/user-attachments/assets/d5bc81b6-97ee-4655-90cd-fc86247607f4" />

<br>

<br>

`\n` is not blacklisted. But when we try `ls` we see the different response:

<br>

<img width="1918" height="700" alt="image" src="https://github.com/user-attachments/assets/f6ed5dd8-4a07-44ba-9d98-50a107d39bae" />

<br>

<br>

We got `Malicious request denied` and understand that there is a blacklist commands filter in place on the back-end. This is why symbol by symbol enumeration is crucial. If we injected the entire command `\n ls` we would not understand what is filtered and what we have to work with.

<br>

The easiest way we can try to bypass a command blacklist is obfuscation:

<br>

<img width="959" height="336" alt="image" src="https://github.com/user-attachments/assets/04cc5de6-a405-42f8-ae16-c393c6e894c2" />

<br>

<br>

We confirmed the existence of the command injection vulnerability and can proceed to exploitation phase

<br>


## Exploitation

<br>

### Manual Filter Bypass Methodology

<br>

To successfully execute arbitrary command and read `/flag.txt`, a multi-layered evasion strategy was constructed to bypass each component of the filter without triggering the blacklist:

<br>

* Context Termination (Newline Separator) - The URL-encoded newline character `%0a` allowed terminating the original `mv` command line and starting a new command
* Whitespace Evasion (Brace Expansion) - Commas inside curly braces are interpreted by the shell as argument separators without requiring spaces
* Command Name Obfuscation - The application blocked reading utilities like `cat`. Inserting single quotes within the string allowed bypassing string-matching filters while remaining valid syntax for the Bash
* Path Separator Bypass via Environment Variables - To reference files in the root directory without using the forward slash `/`, the first character of the system `$PATH` variable was extracted using string slicing

<br>


### Proof of Concept

<br>

The payload successfully bypassed backend filters and executed on the host system. The server response returned both the standard error of the failed `mv` command and the contents of the target file:

<br>

<img width="1917" height="694" alt="image" src="https://github.com/user-attachments/assets/79ce4c6c-0dd0-4a56-a055-5dc92013f7ca" />

<br>

<br>

Full system compromise and unauthorized file access were successfully demonstrated

<br>


## Remediation Recommendations

<br>

To effectively mitigate Command Injection vulnerabilities, the following controls should be implemented:

<br>

* Avoid system shell calls - Replace system calls with native PHP functions. Completely eliminate external shell execution `system()`, `exec()`, `passthru()` for file operations. Instead, use native PHP functions `copy()`, `rename()`
<br>

* Implement input sanitization and whitelisting - If external commands are necessary, enforce strict whitelist validation instead of blacklists. Ensure input parameters match expected patterns and reject any request that does not match

<br>

* Principle of Least Privilege - Ensure the web server process `www-data` operates with minimal system permissions and limit the scope accessible by the web application to its folder `open_basedir = '/var/www/html'` 

<br>

