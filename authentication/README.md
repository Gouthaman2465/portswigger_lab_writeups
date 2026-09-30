# Authentication

PortSwigger Web Security Academy — authentication bypass labs.

## 1. Username enumeration via different responses
[Lab link](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)

**Vulnerability:** The login endpoint returns a different response length depending on whether the username exists.
**Goal:** Brute-force a valid username and password from provided candidate lists.
**Steps:**
- Intercepted a login request and sent it to Intruder with the username field marked as the payload position (sniper attack).
- Loaded the candidate usernames and ran the attack; one response was a noticeably different length — that was the valid username.
- Repeated the attack with that username fixed and the password field as the payload position; one response returned `302` instead of the usual status, revealing the correct password.

**Why it works:** A response that differs by length or content between "user not found" and "wrong password" leaks which usernames are valid, turning a 2-factor guess (user + pass) into two independent, much cheaper brute-force steps.
**Fix:** Return identical responses (status, timing, body) for both "unknown user" and "wrong password" cases.

---

## 2. 2FA simple bypass
[Lab link](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)

**Vulnerability:** The post-login 2FA step is not enforced server-side; only the client-side flow navigates to it.
**Goal:** Access the victim's account without their 2FA code.
**Steps:**
- Logged in with valid credentials, which redirected to a "enter your 2FA code" page while an email with the code was sent.
- Instead of entering the code, navigated directly to `/my-account`.
- The account page loaded without ever submitting a valid 2FA code.

**Why it works:** The server considered the user "logged in" after the first factor and only relied on the client to request the second factor next — it never checked, server-side, that the 2FA step had actually been completed before granting access to protected pages.
**Fix:** Track 2FA completion as explicit server-side session state, and require it before serving any authenticated page.

---

## 3. Password reset broken logic
[Lab link](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)

**Vulnerability:** The password-reset flow checks that two token values match each other, but never validates that the token belongs to the username being reset.
**Goal:** Reset another user's (`carlos`) password.
**Steps:**
- Triggered a password reset for the attacker's own account and captured the reset request, which included the reset token both in the URL and in a hidden form field.
- Resent the final "set new password" request, replacing only the username with `carlos` while leaving both token values (which matched each other) unchanged.
- The server accepted the reset because the two token copies matched, without checking they were actually issued for `carlos`.

**Why it works:** The check "do these two token values match" is not the same as "does this token belong to this specific user." As long as both copies matched each other, the server assumed the request was legitimate for whichever username was supplied.
**Fix:** Bind the reset token server-side to a specific user ID at generation time, and verify that binding on submission — not just self-consistency between two client-supplied copies.

---

## 4. Username enumeration via subtly different responses
[Lab link](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses)

**Vulnerability:** Same class as lab 1, but the differences between "invalid username" and "invalid password" responses are too subtle to spot by eye.
**Goal:** Brute-force a valid username and password.
**Steps:**
- Ran the same Intruder sniper attack as lab 1, but configured a Grep - Extract rule on the exact error text ("incorrect username or password") to surface it as its own results column.
- Compared that extracted text across responses; the username whose error text differed slightly was the valid one.
- Repeated with that username fixed to brute-force the password, using status code `302` as the success indicator.

**Why it works:** Even minor wording or formatting differences in an error response are enough to distinguish valid from invalid usernames once you diff responses systematically (Burp's Grep-Extract makes subtle differences easy to spot at scale).
**Fix:** Use one static, identical error message and response shape for every authentication failure case.

---

## 5. Broken brute-force protection, IP block
[Lab link](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block)

**Vulnerability:** The IP-based lockout only counts consecutive failed attempts, resetting the counter after any request that isn't 3-in-a-row wrong.
**Goal:** Brute-force the password for a known username without triggering the IP block.
**Steps:**
- Determined that the lockout triggered after 3 consecutive wrong attempts, and that a successful-looking (or differently-timed) request in between reset that counter.
- Built an Intruder attack using a repeating pattern: 2 wrong credentials followed by the attacker's own valid login, cycling through the password candidate list this way (a pitchfork attack pairing candidate passwords with the reset pattern).
- Found the correct password once the pattern completed a full pass through the candidate list.

**Why it works:** Rate-limiting based only on a short streak of consecutive failures can be reset by interleaving one "clean" request, letting an attacker brute-force indefinitely in small batches.
**Fix:** Base lockout on a rolling window of total failed attempts per account/IP over time, not on an easily-reset consecutive-failure streak.

---

## 6. 2FA broken logic
[Lab link](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-broken-logic)

*Attempted but not yet completed — will be added once solved.*
