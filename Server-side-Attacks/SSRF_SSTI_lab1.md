# Chaining SSRF & SSTI to Achieve RCE

<br>

## Objective and Scope

<br>

The primary goal of this assessment is to evaluate the backend infrastructure, configuration, and server logic for weaknesses that an attacker could exploit, following a systematic black box methodology using modern tools and manual testing techniques.

<br>

---

## Initial Reconnaissance & Information Gathering

<br>

The web site contains information about the company:

<br>

<img width="1512" height="894" alt="image" src="https://github.com/user-attachments/assets/f273eef2-bca2-4441-932e-1a434d32621c" />

<br>

<br>

The front-end does not provide us options to send custom inputs to the server, so we will use proxy to inspect the HTTP response deeply:

<br>

<img width="1512" height="769" alt="image" src="https://github.com/user-attachments/assets/16dd840f-333d-4b0f-9d71-cbff6e724dbf" />

<br>

<br>

There are three `POST` requests that the web application performs using the `api` parameter in the body. This web site fetches data from the `truckapi.htb` server and renders it to the end users. If user input is not properly sanitized, we may be able to exploit the `api` parameter making arbitrary requests to the `truckapi.htb` server through SSRF attack or we can enumerate the `localhost` itself or read its files through `file://` URL scheme if it is not disabled.

<br>


### Port Enumeration

<br>

First we will enumerate ports on the `localhost` and `truckapi.php`. When the port is open, we get `Please, specify a truck ID`:

<br>

<img width="986" height="418" alt="image" src="https://github.com/user-attachments/assets/2989524d-bab4-4cd9-be01-10c473d20c76" />

<br>

<br>

When the port is closed, we get an error:

<br>

<img width="1107" height="412" alt="image" src="https://github.com/user-attachments/assets/312b4c1b-55cf-44f2-880a-def55ac7cd94" />


<br>

<br>

Setting up `ffuf` filters:

<br>

```shell
seq 1  65535 > ports.txt
```

<br>

```shell
ffuf -w ports.txt -u http://154.57.164.80:39915 -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "api=http://truckapi.htb:FUZZ" -fr "Error"
```

<br>


<img width="891" height="352" alt="image" src="https://github.com/user-attachments/assets/36b854f1-7fff-4a92-9808-3da32efdd2df" />

<br>

<br>

<br>

Enumerating the `localhost`:

<br>

<img width="1107" height="756" alt="image" src="https://github.com/user-attachments/assets/f337dacf-2dd6-4b60-9c4b-81c0f3f029f3" />

<br>

<br>

<img width="1104" height="429" alt="image" src="https://github.com/user-attachments/assets/9449e630-9878-459d-a1f8-11783e48d3a1" />

<br>

<br>

```shell
ffuf -w ports.txt -u http://154.57.164.80:39915 -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "api=http://127.0.0.1:FUZZ" -fr "Error"
```

<br>

<img width="890" height="354" alt="image" src="https://github.com/user-attachments/assets/d23e1865-3974-4e09-9ad9-4987859d6710" />

<br>

<br>

Now we know that only port `80` is open on both servers.

<br>

`file://` URL scheme is disabled on the `localhost`:

<br>

<img width="1104" height="417" alt="image" src="https://github.com/user-attachments/assets/54e65d2e-a060-4019-bf07-545cf48e25f5" />

<br>

<br>

### Inspecting `ID` parameter

<br>

The initial `api` request was:

<br>

```http
api=http://truckapi.htb/?id%3DFusionExpress01
```

<br>

If we change it to 900, we get the following response:

<br>

<img width="1106" height="406" alt="image" src="https://github.com/user-attachments/assets/2f291dee-5e94-4523-8ea9-b6338f53be6c" />


<br>

<br>


We can assume that server templates are being used.

<br>

## Vulnerability Assessment

<br>

To confirm `SSTI` vulnerability we can provide a payload `${{<%[%'"}}%\.`that consists of all special characters that have a particular semantic purpose in popular template engines:

<br>

<img width="1106" height="418" alt="image" src="https://github.com/user-attachments/assets/0b1b5da4-88fc-4f30-9fce-547bcfbe78d2" />

<br>

<br>

This time, the server returned an error about `ID`. Such a response may indicate an internal server error `HTTP 500 code`. We need to find the exact characters that caused the error:

<br>

<img width="1104" height="369" alt="image" src="https://github.com/user-attachments/assets/f80b7548-b776-4d30-ac45-05de92e0c9f7" />

<br>

<br>

<img width="1103" height="352" alt="image" src="https://github.com/user-attachments/assets/8ede71e5-4e47-451e-9980-a4bf48abcfde" />

<br>

<br>

<img width="1103" height="411" alt="image" src="https://github.com/user-attachments/assets/b129f3ba-5d48-48d5-80b6-c427141e1a10" />


<br>

<br>

The `{{` payload causes an error, confirming that the `ID` parameter we provide goes directly to the template, not to its value. Thus confirms `SSTI` vulnerability.

<br>

### Identifying the Template Engine

<br>

There are several techniques we can use to identify the template engine:

<br>

<img width="1105" height="401" alt="image" src="https://github.com/user-attachments/assets/0a637fd4-c463-441a-8252-a4fa198d2b7e" />

<br>

<br>

<img width="1106" height="352" alt="image" src="https://github.com/user-attachments/assets/2a07dbc0-6f9c-41e4-882f-06f65859855b" />

<br>

<br>

By analyzing these responses we see that the server uses PHP `Twig` template engine. It uses implicit type conversion resulting in `5 * '5'` as 25 and performs the `{{_self}}` command:

<br>

<img width="1105" height="411" alt="image" src="https://github.com/user-attachments/assets/83b8804b-6987-4354-b720-27b49affb3df" />

<br>

<br>


## Exploitation

<br>

To execute system commands, we leveraged Twig's filter functionality to invoke system-level execution functions, such as `filter('system')`. Initial tests with basic commands like `pwd` successfully returned the current working directory:

<br>

<img width="1140" height="547" alt="image" src="https://github.com/user-attachments/assets/9a4dc98d-9ddb-4824-871c-bd96df192f3b" />

<br>

<br>

To list all the files in `/` root directory we have to implement bash internal field separator `$IFS` or double URL-encoding technique `%2520`, ensuring the space character reached the template engine intact:

<br>


<img width="1105" height="718" alt="image" src="https://github.com/user-attachments/assets/a3ca6ba9-99ce-4e70-99ec-c62e63abc1b1" />

<br>

<br>

Reading the sensitive file:

<br>

<img width="1105" height="433" alt="image" src="https://github.com/user-attachments/assets/bab04d92-942d-4309-800f-49af4d527cd6" />

<br>

<br>

By chaining these findings, we successfully evaluated the risk impact, traversed system directories, and retrieved the target file, demonstrating a critical exposure in the backend architecture.

<br>


## Remediation Recommendations

<br>

To address the identified vulnerabilities and mitigate the risk of SSRF, SSTI, and Remote Code Execution, the following remediation measures should be implemented:

<br>

* Implement strict input validation and allow lists for all URLs or domains accepted by the application. Enforce strict type validation and restrict the `ID` parameter to numeric values only (SSRF prevention).

<br>

* Secure template engine configuration. Never pass user input directly into template source code or evaluation functions. Restrict access to dangerous global objects, methods, and functions such as `system`, `exec`, `passthru`. Separate the execution environment in which the template engine runs entirely from the web server, for instance, by setting up a separate execution environment such as a Docker container (SSTI prevention).

<br>

* Run the web application  under an account with minimal permissions to limit the impact of potential command execution (Least Privilege Principle).

<br>

* Utilize a Web Application Firewall (WAF) - Deploy and configure a WAF to detect and block malicious patterns associated with template injection, command separators, and SSRF payloads.

<br>

