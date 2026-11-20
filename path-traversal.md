# Path Traversal

> PortSwigger Web Security Academy · **6/6** labs completed

Path traversal lets an attacker read files outside the intended directory by manipulating file paths with sequences like `../`.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. File path traversal, simple case](#1-file-path-traversal-simple-case)
- [2. File path traversal, traversal sequences blocked with absolute path bypass](#2-file-path-traversal-traversal-sequences-blocked-with-absolute-path-bypass)
- [3. File path traversal, traversal sequences stripped non-recursively](#3-file-path-traversal-traversal-sequences-stripped-non-recursively)
- [4. File path traversal, traversal sequences stripped with superfluous URL-decode](#4-file-path-traversal-traversal-sequences-stripped-with-superfluous-url-decode)
- [5. File path traversal, validation of start of path](#5-file-path-traversal-validation-of-start-of-path)
- [6. File path traversal, validation of file extension with null byte bypass](#6-file-path-traversal-validation-of-file-extension-with-null-byte-bypass)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Find any parameter that looks like it maps to a file: `file=`, `path=`, `doc=`, `image=`, `template=`, `lang=`, a download filename.
- Test the basic traversal sequence `../../../etc/passwd` (and the Windows equivalent) and see if it's blocked, stripped, or works.
- If stripped once, test nested sequences like `....//....//` which survive a naive single-pass `../` removal.
- Test URL-encoded and double-encoded traversal (`%2e%2e%2f`, `%252e%252e%252f`) if the app decodes more than once.
- Test absolute paths and path truncation, and whether a required file extension can be bypassed with a null byte or similar legacy trick.
- If traversal to arbitrary files isn't possible, check if the app still lets you read files within intended directories you shouldn't access (e.g. other users' uploads via sequential/guessable paths).

## Lab write-ups

### 1. File path traversal, simple case

**Lab:** [File path traversal, simple case](https://portswigger.net/web-security/file-path-traversal/lab-simple)

- In this lab we have to access the etc/passwd files and we know that we have vuln in viewing images
- To do this open burp and load the lab web page
- Go to http history in proxy and look for images file
- Send the file to the repeater and try to change the filename to /etc/passwd
- If it doesn't work then add the relative path in front of it like `../etc/passwd`
- Do this until you get it
- The answer is `../../../etc/passwd` because images usually present in the folder `var/www/images/1.jpg` so to get the /etc file we have to come out of the /var directory

---

### 2. File path traversal, traversal sequences blocked with absolute path bypass

**Lab:** [File path traversal, traversal sequences blocked with absolute path bypass](https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass)

- This lab is as same as the previous one. the only difference is relative path is blocked but they forgot to block the absolute path
- So change the filename to `/etc/passwd`

---

### 3. File path traversal, traversal sequences stripped non-recursively

**Lab:** [File path traversal, traversal sequences stripped non-recursively](https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively)

- In this lab the traversal relative path is stripped once means if you give `../../../etc/passwd`, it will strip away the dots and slash and return `/etc/passwd` and it only does once for every sequence
- So what if we give like this `....//....//....//etc/passwd` , if it strips one set we will end up with `../../../etc/passwd` and we will get our path

---

### 4. File path traversal, traversal sequences stripped with superfluous URL-decode

**Lab:** [File path traversal, traversal sequences stripped with superfluous URL-decode](https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode)

- In this lab, the input gets url-decode not once but twice. the vuln lies there
- Normal webpages has only one decoding but here it is having 2.
- The word superfluous means extra so here extra decoding
- Normally the character gets decoded into the following  values
- `. decodes to %2e`
- `/ decodes to %2f`
- `% decodes to %25`
- So if I want to give it as decoded url then `../../../etc/passwd` becomes `%2e%2e%2f%2e%2e%2f%2e%2e%2fetc@2fpasswd`
- But here it has double decoding so to solve it decode the character `%` so that it gets removed in the first decoding so the final answer will become `%252e%252e%252f%252e%252e%252f%252e%252e%252fetc%252fpasswd`

---

### 5. File path traversal, validation of start of path

**Lab:** [File path traversal, validation of start of path](https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path)

- In this lab, in states that we have to start from a valid path or a valid directory(with a name like /var or /etc)
- So to traverse to our desired directory we can just start with `var/www/images/../../../etc/passwd` because it does follow our rule that we have to start it with a directory and we are starting with it but afterwards the server reads the whole input and goes to our desired directory

---

### 6. File path traversal, validation of file extension with null byte bypass

**Lab:** [File path traversal, validation of file extension with null byte bypass](https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass)

- In this lab, they have told that the path should end with a file extension and here it is an image so it should with a .jpg or .png
- But this check only happens at the application layer but the server is C-based so we can have a null terminator like %00 and then the extension so that in application layer we can show them that is is ending with a extension but the server removes the null terminator
- So the final answer will be `../../../etc/passwd%00.jpg`

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Avoid passing user input to filesystem APIs. Use IDs mapped to files instead of file names.
- If a path is needed, canonicalize it and verify the result starts with the intended base directory.
- Do not rely on stripping `../` or on blocklists. Nested sequences, encodings and null bytes bypass them.
- Run the application with a user that can read only what it needs.
