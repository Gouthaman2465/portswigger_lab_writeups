# Authentication Vulnerabilities

> PortSwigger Web Security Academy · **5/14** labs completed

Authentication flaws let attackers log in as other users through username enumeration, brute force, broken 2FA, or weak password reset logic.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Username enumeration via different responses](#1-username-enumeration-via-different-responses)
- [2. 2FA simple bypass](#2-2fa-simple-bypass)
- [3. Password reset broken logic](#3-password-reset-broken-logic)
- [4. Username enumeration via subtly different responses](#4-username-enumeration-via-subtly-different-responses)
- [5. Broken brute-force protection, IP block](#5-broken-brute-force-protection-ip-block)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Test username enumeration: compare responses (error text, status code, response time, response length) for a known-valid vs unknown username on login, register, and password reset.
- Test brute-forcing the login: is there rate limiting, account lockout, or CAPTCHA, and can it be bypassed (rotating IP via headers, resetting the counter via a successful attempt, trying passwords against many usernames instead of one).
- Test 2FA: can the 2FA step be skipped by going straight to the post-login page, is the 2FA code itself brute-forceable, and is it bound to the same user/session that passed step one?
- Test password reset: is the token unpredictable, single-use, and does it expire, and is it actually checked against the account it was issued for?
- Test "remember me" and session tokens for predictability or for being guessable/sequential.
- Test whether changing one's own password/email requires re-entering the current password (prevents session-riding attacks on account takeover).
- Test multi-step/multi-factor logins for logic flaws: can a step be skipped or completed out of order?

## Lab write-ups

### 1. Username enumeration via different responses

**Lab:** [Username enumeration via different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)

- In this lab they have given a set of commonly used usernames and passwords and out job is to brute force it
- First intercept the request using burp and highlight the username and send it to the intruder
- The attack type should be sniper attacks and load all the username in the right panel under payload config
- Then click attack
- Notice the length of the response and note one will be greater than all the others and that is the username
- Then `clear$, replace the username with the found username and highlight password and add$`
- Now load all the password and click attack
- Now notice the status code, everything will be 200 but one will be 302 and that is the password

---

### 2. 2FA simple bypass

**Lab:** [2FA simple bypass](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)

- First login into your account, you will get the code in your email client
- Note that the url changes to /my-account after you enter the code
- So now lets try without giving the code we will change the url manually to /my-account and now we are into the account without the code
- Do the same for victim's login

---

### 3. Password reset broken logic

**Lab:** [Password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)

- In this lab we have to gain the access by using the reset password section
- When we click the reset password, it is asking for a username and we have our username and password
- When we give our username, it is sending us the reset password email
- The reset password token is taken twice-one in the url and the other one in the bottom. so its just checking both are equal and if it is then it is allowing to change any user's password and that the vuln
- So to exploit it we are just sending the particular request(which is the request when clicking reset password after giving new password and confirm pass) to the repeater
- Now we are changing the username to carlos and sending the request and we got 302 and now we click the redirection
- One more way is to use the proxy. when the particular request which is used to change the new password arrives, change the username to carlos and pass it
- It is taking us to the home page confirming that it is working
- This lab only checks whether the tokens are equal no what what the token is and what the username is
- Now we are into carlos account

---

### 4. Username enumeration via subtly different responses

**Lab:** [Username enumeration via subtly different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses)

- In this lab we have to brute it like lab 1.
- The method is the same but we have to add a grep so that we can see a new column if the username or password is incorrect
- We are doing this because in one response the error message will be a bit off to find that we are adding the grep extract
- To do that we have to click the settings which is there right panel and click add in grep-extract panel
- Now click response and find `incorrect username or password` and highlight it and click ok
- Now run the sniper attack for username and take the username which gives that different error and that is the username, now replace the username with the found  username  and run password sniper attack and find the password with status code 302

---

### 5. Broken brute-force protection, IP block

**Lab:** [Broken brute-force protection, IP block](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block)

- In this lab there is a logic flaw in brute force. I found out that for every 3 wrong credentials, the ip is being blocked so I tried 2 wrong and then my credentials and again a wrong one and it worked
- So the flaw here is I can try 2 incorrect credentials and then use mine and continue this until I find the correct password
- So I created the username with my username and 2 carlos and this pattern repeats
- For password list also this is the same my password and 2 passwords from the list and so on
- I ran a pitchfork attack and found out that the password is **qwerty**

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Return the same response (and take about the same time) for wrong username and wrong password.
- Rate limit by account and by IP, in a way that cannot be bypassed by headers like `X-Forwarded-For`.
- Verify 2FA server-side and bind it to the user and session that passed the first step.
- Make password reset tokens random, single-use, time-limited and tied to the user. Check the token on both the page and the submit step.
- Enforce strong passwords and check them against known breached passwords.
