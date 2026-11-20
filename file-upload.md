# File Upload Vulnerabilities

> PortSwigger Web Security Academy · **7/7** labs completed

Insecure file upload lets an attacker place a malicious file (such as a web shell) on the server, or use the upload feature to bypass other protections.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Remote code execution via web shell upload](#1-remote-code-execution-via-web-shell-upload)
- [2. Web shell upload via Content-Type restriction bypass](#2-web-shell-upload-via-content-type-restriction-bypass)
- [3. Web shell upload via path traversal](#3-web-shell-upload-via-path-traversal)
- [4. Web shell upload via extension blacklist bypass](#4-web-shell-upload-via-extension-blacklist-bypass)
- [5. Web shell upload via obfuscated file extension](#5-web-shell-upload-via-obfuscated-file-extension)
- [6. Remote code execution via polyglot web shell upload](#6-remote-code-execution-via-polyglot-web-shell-upload)
- [7. Web shell upload via race condition](#7-web-shell-upload-via-race-condition)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Identify what the upload is used for and where the file ends up (is it served back, and from where).
- Test uploading a file with a dangerous extension directly (`.php`, `.jsp`, `.asp`) to see if any validation exists at all.
- If the extension is checked, test case variations, double extensions (`shell.php.jpg`), null bytes, trailing characters, and alternate executable extensions the stack actually runs (`.phtml`, `.php5`).
- If content-type/magic-byte checks exist, test making the file pass that check while keeping a payload inside it (e.g. a polyglot, or image metadata containing code for a server that processes images unsafely).
- Check whether the upload directory executes scripts at all — if not, look for a way to move/rename the file into an executable location (path traversal in the filename field).
- Test overwriting an existing, sensitive file (e.g. a config file) via a traversal sequence in the filename.
- Test race conditions: can the file be accessed/executed in the short window between upload and server-side validation/deletion?

## Lab write-ups

### 1. Remote code execution via web shell upload

**Lab:** [Remote code execution via web shell upload](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-web-shell-upload)

- In this lab the vuln exists in the profile avatar upload file
- First we will upload a file and see the http history
- There is a POST request `/my-account/avatar` which uploads the file and now go to http history filter
- Under MIME types enable images and now you will see a GET request `/files/avatar/abc.png` which retrieves the file you have uploaded
- Now remove the image content from `/my-account/avatar` and rename the filename to hi.php and write a simple php code to say HI
- The code is `<?php echo "hi"; ?>`
- Now send the request and go to `/files/avatar` endpoint and change the filename from abc.png to hi.php and now you will see the HI message there
- Now write the php code to retrieve the secret from carlos/secret.
- The php code is `<?php echo file_get_contents('/home/carlos/secret');?>` and this will retrieve the content from the specified endpoint and send the request
- Now go to `/files/avatar/hi.php ` and send the request and now you got the secret
- The secret is `OKrZnBgNrU0oMEAwNO9UaM8QTFy9DFtP`

---

### 2. Web shell upload via Content-Type restriction bypass

**Lab:** [Web shell upload via Content-Type restriction bypass](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass)

- This lab has the vuln where it only checks for the whitelisted filetype but not the actual file uploaded in there
- So to exploit it, odm the same thing as previous lab
- The secret is `R4NHpwwLZnfEzZJNVPOkHmCOyXED75ov`

---

### 3. Web shell upload via path traversal

**Lab:** [Web shell upload via path traversal](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-path-traversal)

- In this lab the current directory which saves the images are not allowed to execute the php files but it allows to upload the file
- When we try to upload a php file by replacing the already uploaded file, we are able to upload it but when we visit the file, it is printing it as plain text
- So this directory is not allowed to execute the php file
- The payload is `<?php echo "hi"; ?>`
- Now change the filename from hi.php to `../hi.php` and try to upload to the parent directory of avatar
- Now we click the send button and notice that the file is still uploading in `/files/avatar` and this confirms that the server is removing the `/`
- Now url encode the `/ to %2f` so rename the file to `..%2fhi.php` and now we can see that the file has been uploaded in /files/../avatar, means it is uploaded in `/files`
- Now visit the page by changing the filename from `/files/avatar/abc.png` to `/files/avatar/../hi.php` and we now be able to see the HI  message
- Now change the php code to get the secret.
- Payload is `<?php echo get_file_contents('/home/carlos/secret'); ?>` and send the request
- Visit the same directory which is `../hi.php` and you will get the secret

---

### 4. Web shell upload via extension blacklist bypass

**Lab:** [Web shell upload via extension blacklist bypass](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-extension-blacklist-bypass)

- In this lab the php files are blacklisted
- When we try to upload hi.php it is blocking the file from uploading it
- We can see that this is using apache by seeing the response headers `Server: Apache/2.4.41 (Ubuntu)`
- Now we need to specify any arbitrary extension that maps to running a php code
- To do that we need to change the filename to `.htaccess` and content-type to text/plain
- The payload will be ``AddType application/x-httpd-php .abc`` this payload tells that whenever you see a filename with extension .abc you run it as php file
- Now change the filename to hi.abc and add the payload to retrieve the secret
- Payload is `<?php echo file_get_contents('/home/carlos/secret'); ?>`
- Now visit hi.abc in `/files/avatar/hi.abc` to get the secret and submit it

---

### 5. Web shell upload via obfuscated file extension

**Lab:** [Web shell upload via obfuscated file extension](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-obfuscated-file-extension)

- In this lab, the php is again blacklisted but this could be bypassed by classic methods
- The methods to try are
	- Check whether it is case sensitive- php to pHp
	- Trailing dots like abc.php.
	- Double extensions like abc.php.png
	- url encoding like abc%2ephp
	- Null byte like abc.php%00.png
- The first 4 methods didn't work and the fifth method which is null byte works
- Change the filename to `hi.php%00.png` which eliminates everything after %00
- Now visit hi.php in file retrieving endpoint to get the secret key using the below payload
- `<?php echo file_get_contents('/home/carlos/secret'); ?>`

---

### 6. Remote code execution via polyglot web shell upload

**Lab:** [Remote code execution via polyglot web shell upload](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-polyglot-web-shell-upload)

- In this lab there is a new concept called polyglot.
- First check whether we are able to upload any php file and use techniques we used before and none of them doesn't work
- Polyglot is a file where it is considered as one or more file format depending on the application server parsing and validating the file
- So in this lab we need to create a polyglot file which contains our php code in it with a image.
- We need to build a polyglot image with the code and the image we already have
- To do that we need to download an cli app to do it
- `sudo apt install libimage-exiftool-perl -y`
- Then verify the version of it using `exiftool -ver`
- Now create the polyglot file using a pre existing image and that should be .jpg
- To convert it from png to jpg we need to install another tool
- `sudo apt install imagemagick -y`
- `magick abc.png abc.jpg`
- Now build the polyglot image `exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" abc.jpg -o polyglot.php`
- Now upload this file and look at the `/files/avatar/polyglot.php` file and look for something like start. the secret will be present in between START and the END
- The secret is `38jAxM4zMgfbLrKSKz6IFzHBXut0Cm5K`

---

### 7. Web shell upload via race condition

**Lab:** [Web shell upload via race condition](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-race-condition)

- In this lab the vuln exists in race condition
- Race condition is a phenomenon in cybersecurity where when 2 or more threads look forward to perform an action and the timing of it collides
- A **race condition** is a flaw in a concurrent system where the **output or behavior depends on the exact sequence or timing of uncontrollable events** (such as the execution order of multiple threads or processes).
- So here the system does many things
	- First it needs to collect the input
	- Then it should store the input in a temp folder
	- Then it should validate the input
	- After passing the input validation, it needs to store it permanently
- So what we are going to try is, we are going to add a php file as a input, it will get stored temp and we need to access and run the php file before it disappears. to do that we need to build the php file with the payload `<?php echo file_get_contents('/home/carlos/secret'); ?>` and upload it
- Then we will deliberately try to visit the page which shows the php file which is `/files/avatars/hi.php`
- Now we send both the requests to repeater and create a group.
- Now we will try to send the group requests in parallel so that at one point of time we will be able to access and execute the php file before it gets deleted from the temp folder and we did able to get the code in 5th time
- The secret is `uTe3AUHpFCVDtE3tlKA5sIbARReLoNj1`

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Check the file extension against an allowlist and check the real content (magic bytes) on the server. Never trust the `Content-Type` header or the client-supplied file name.
- Rename uploaded files and store them outside the web root, or on separate storage that cannot execute code.
- Disable script execution in upload directories.
- Set size limits, and validate the file fully before moving it to its final location (to avoid race conditions).
- Remove metadata and re-process images where possible.
