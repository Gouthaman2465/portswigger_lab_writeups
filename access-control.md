# Access Control Vulnerabilities

> PortSwigger Web Security Academy · **11/13** labs completed

Broken access control lets users act outside their permissions. It includes vertical privilege escalation, IDOR/BOLA, and bypasses of URL or method restrictions.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Unprotected admin functionality](#1-unprotected-admin-functionality)
- [2. Unprotected admin functionality with unpredictable URL](#2-unprotected-admin-functionality-with-unpredictable-url)
- [3. User role controlled by request parameter](#3-user-role-controlled-by-request-parameter)
- [4. User role can be modified in user profile](#4-user-role-can-be-modified-in-user-profile)
- [5. User ID controlled by request parameter](#5-user-id-controlled-by-request-parameter)
- [6. User ID controlled by request parameter, with unpredictable user IDs](#6-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)
- [7. User ID controlled by request parameter with data leakage in redirect](#7-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect)
- [8. User ID controlled by request parameter with password disclosure](#8-user-id-controlled-by-request-parameter-with-password-disclosure)
- [9. Insecure direct object references](#9-insecure-direct-object-references)
- [10. URL-based access control can be circumvented](#10-url-based-access-control-can-be-circumvented)
- [11. Method-based access control can be circumvented](#11-method-based-access-control-can-be-circumvented)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Map all roles/user tiers in the app and identify functionality restricted to a higher tier.
- Try accessing higher-tier functionality directly by URL, ignoring any hidden menu items, as a lower-privileged or unauthenticated user.
- For any object reference (order ID, invoice number, user ID) in a URL or parameter, swap it for another user's ID and see if you can access their data (IDOR/BOLA).
- Test both read and write actions on someone else's object, not just read — can you modify or delete another user's resource?
- Check if access control is enforced consistently across all methods for the same resource (GET might be checked but PUT/DELETE/PATCH not).
- Check for multi-step process flaws: can a later step in a workflow be accessed directly, skipping an authorization check that only happens earlier in the flow?
- Check whether client-controlled data decides access (a hidden form field for role, a JWT claim the client can edit, a request parameter like `isAdmin=true`).
- Check platform/URL-based bypasses: trailing slashes, case changes, extra path segments, or alternate HTTP methods that route around a restriction enforced at a specific path.

## Lab write-ups

### 1. Unprotected admin functionality

**Lab:** [Unprotected admin functionality](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)

- In this lab we have to delete carlos
- To do that first we will be visiting `/robots.txt` and know the admin directory and we found out that it is `/administrator-panel`
- Now visit that particular directory and delete it

---

### 2. Unprotected admin functionality with unpredictable URL

**Lab:** [Unprotected admin functionality with unpredictable URL](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url)

- Inspect the webpage and search for admin
- You will see something like var=admin() and in that JS function you will see the href link with the random url so that is the url and we found out that is `/admin-oocdye`
- Go to that directory and delete carlos

---

### 3. User role controlled by request parameter

**Lab:** [User role controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)

- In this lab we have the /admin page and they have given it
- Out job is to find how to change the permission of our account the admin so that we can delete the other user
- To do this first login to your user and send that response to the repeater
- There will be cookie called admin=false so change it to true and send it and now we have the admin panel and we confirmed it
- To change it in our webpage go to inspect -> application and look for cookie
- Change admin from false to true and refresh the page and now you will have the admin panel

---

### 4. User role can be modified in user profile

**Lab:** [User role can be modified in user profile](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)

- In this lab we have to delete the user carlos but it is only possible with a user whose roleid is 2
- So to do this we are going to login to our account and try to modify out email and see what type of file it is sending to the server
- When we intercept the request we was able to see it is json file so we will manually add `"roleid"=2` to that json file so forward it and we now have the admin access and we can delete the user

---

### 5. User ID controlled by request parameter

**Lab:** [User ID controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter)

- This is a basic IDOR so login to our account and change the id from wiener to carlos and get the API key
- `WSy27LFX8mo9II52FNP3rT092eEnjUPi` is the API key

---

### 6. User ID controlled by request parameter, with unpredictable user IDs

**Lab:** [User ID controlled by request parameter, with unpredictable user IDs](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)

- This is also a IDOR but identified with the GUID not the user id
- GUID is a random 32 character string used as the user id instead of 1 or 2 or 3 and it cant be guessed by the attacker and cant brute force 32 characters
- So we have to see whether carlos has interacted in this blog and we found out that one blog is written by carlos
- Click carlos and you will get his blog with the GUID in the url as id
- Copy it and now login into your account and change the GUID and now you have his API key

---

### 7. User ID controlled by request parameter with data leakage in redirect

**Lab:** [User ID controlled by request parameter with data leakage in redirect](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect)

- In this lab it has the vuln in redirection, means when you send a request by doing IDOR for carlos account, it says that it cant do it but it silently gives away the redirected page of carlos's account
- So to solve this we are going to try IDOR for carlos account and send that request to the repeater and now we are going to send it using repeater, render the response and you wil get the API key of carlos

---

### 8. User ID controlled by request parameter with password disclosure

**Lab:** [User ID controlled by request parameter with password disclosure](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure)

- In this lab we have IDOR vuln and the password is prefilled in its field.
- So first change wiener to administrator in the url and login it
- Now you have the masked password, inspect the webpage and get the password
- Now login as administrator and use the obtained password and then delete carlos

---

### 9. Insecure direct object references

**Lab:** [Insecure direct object references](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)

- In this lab we have a new section called live chat and the vuln is under that section
- This section changes the url static and it has a option of view transcript and when I click it, it is giving me a 2.txt which is a static url endpoint
- So I sent the particular request to repeater and changed from 2.txt to 1.txt and we got the password for carlos
- Now we logged in as carlos with the password `t37411xglsjmcdr5zm2v`

---

### 10. URL-based access control can be circumvented

**Lab:** [URL-based access control can be circumvented](https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented)

- In this lab the vuln is in the server end which blindly believes the header `X-Original-URL` which is a header used to specify the exact path that the browser should take it to the server and this lab blindly believes it without checking
- To do this intercept the request and change the GET method path from `/admin` to `/` and add `X-Original-URL: /admin` at the bottom of the request
- Now you are admin and you have to delete the carlos user
- To do this again intercept the request after clicking delete and change the POST method path from `/admin/delete?username=carlos` to `?username=carlos` and add `X-Original-URL: /admin/delete` at the bottom of the request
- Now you will get a page called access denied but whe you come back to that page, the user would have got deleted

---

### 11. Method-based access control can be circumvented

**Lab:** [Method-based access control can be circumvented](https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented)

- In this lab we have the admin account and we have the access to upgrade or downgrade the user but we want to upgrade a user using a non-admin user and we have vuln in http request method like POST or GET
- So to do this first go to admin account and see how the upgrade works and what method it is taking and so on
- Upgrade carlos to admin and send that request to the repeater
- Now login as wiener and take the session cookie and replace it with the admin's cookie which changed the role of carlos from user to admin and send the request and you will get unauthorized
- Now right click in request of that particular same request which got failed due to non-admin user cookie and change the http method so now it changes from POST to GET automatically and change the username from carlos to wiener and now we have changed the role using a non-admin role because we have used the non-admin user's cookie
- Note that the all process only happens in the request which the carlos role was changed.
- We only login to wiener to get the cookie and paste it here and rest happens here (`/admin-roles request`)

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Deny by default, and enforce authorization on the server for every request.
- Check object ownership on every access (for example, that the order belongs to the caller). Do not trust IDs from the client.
- Do not rely on hidden URLs, client-side role values or request parameters to decide permissions.
- Centralize authorization logic, and test it, instead of repeating checks in each handler.
- Do not trust headers such as `X-Original-URL` or `Referer` for access decisions.
