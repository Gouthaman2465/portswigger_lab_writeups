# Access Control

PortSwigger Web Security Academy — Broken Access Control labs.

## 1. Unprotected admin functionality
[Lab link](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)

**Vulnerability:** Admin panel has no access control — it relies on the URL being secret.
**Goal:** Delete the user `carlos`.
**Steps:**
- Checked `/robots.txt`, which disclosed a disallowed path: `/administrator-panel`.
- Visited the path directly and used the delete function with no authentication required.

**Why it works:** The application assumed an unlisted URL was equivalent to access control. `robots.txt` is public by design, so "hidden" paths listed there are not actually hidden.
**Fix:** Enforce server-side authorization checks on every admin route, independent of whether the URL is discoverable.

---

## 2. Unprotected admin functionality with unpredictable URL
[Lab link](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url)

**Vulnerability:** Admin URL is randomized but still has no real access control, and is leaked client-side.
**Goal:** Delete the user `carlos`.
**Steps:**
- Inspected the page source and found a JavaScript reference revealing the admin path (e.g. `/admin-oocdye`).
- Visited that path directly and deleted the user.

**Why it works:** "Security through obscurity" fails once the obscured value is exposed anywhere the client can read it, including inline JavaScript.
**Fix:** Never rely on an unguessable URL as the only control. Require server-side role checks on the admin route itself.

---

## 3. User role controlled by request parameter
[Lab link](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)

**Vulnerability:** User privilege is stored in a client-controlled cookie.
**Goal:** Gain admin access and delete another user.
**Steps:**
- Logged in as a normal user and inspected cookies via the browser's Application tab.
- Found a cookie `admin=false`; changed it to `admin=true` and refreshed.
- The admin panel became accessible.

**Why it works:** The server trusted a value the client can freely edit instead of checking a session-bound role on its own side.
**Fix:** Store authorization state server-side (session/DB), never in a client-editable cookie or parameter.

---

## 4. User role can be modified in user profile
[Lab link](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)

**Vulnerability:** Mass assignment — a profile update endpoint accepts a `roleid` field it shouldn't.
**Goal:** Escalate to a role (`roleid=2`) that can delete other users.
**Steps:**
- Intercepted the "update email" request and observed it was a JSON body.
- Added `"roleid": 2` to the JSON body and forwarded the request.
- Gained admin access and deleted the target user.

**Why it works:** The backend bound the entire incoming JSON object to the user model instead of allow-listing which fields a user is permitted to change.
**Fix:** Explicitly allow-list writable fields per endpoint/role; never bind untrusted input directly to a full data model.

---

## 5. User ID controlled by request parameter (IDOR)
[Lab link](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter)

**Vulnerability:** Classic IDOR — an object is fetched by an ID the client controls, with no ownership check.
**Goal:** Read another user's API key.
**Steps:**
- Logged in and located the request that fetches account data by username.
- Changed the username parameter from the logged-in user to the target user.
- The response returned the target's API key.

**Why it works:** The endpoint returned data for whatever ID was requested without verifying the requester owns that ID.
**Fix:** Check that the authenticated session actually owns the requested resource before returning it.

---

## 6. User ID controlled by request parameter, with unpredictable user IDs
[Lab link](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)

**Vulnerability:** Same IDOR pattern as above, but IDs are GUIDs instead of sequential integers.
**Goal:** Read another user's API key using their GUID.
**Steps:**
- Found a blog post authored by the target user, which exposed their GUID in the URL.
- Substituted that GUID into the account-data request in place of the logged-in user's own GUID.
- Retrieved the target's API key.

**Why it works:** An unguessable identifier is not an authorization control — once the GUID leaks anywhere (e.g. a public blog post), the missing ownership check is exploitable exactly as with sequential IDs.
**Fix:** Same as above — verify ownership server-side; don't rely on ID unpredictability as a substitute.

---

## 7. User ID controlled by request parameter with data leakage in redirect
[Lab link](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect)

**Vulnerability:** The endpoint denies direct access but leaks the target's data in a redirect response before blocking it.
**Goal:** Read another user's API key.
**Steps:**
- Requested the target's account page (IDOR attempt); the app denied it visibly.
- Sent the same request through Burp Repeater and inspected the full redirect response rather than the rendered page.
- The API key was present in the body of the redirect response itself.

**Why it works:** The access-control check happened, but only after the sensitive data had already been written into the response — the redirect fired too late.
**Fix:** Perform the authorization check before any response data is generated, not after.

---

## 8. User ID controlled by request parameter with password disclosure
[Lab link](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure)

**Vulnerability:** IDOR combined with a masked password field pre-filled in the HTML.
**Goal:** Log in as admin and delete `carlos`.
**Steps:**
- Changed the user parameter in the URL to `administrator`.
- The account page loaded with the password field pre-filled, masked with dots but present in the HTML.
- Inspected the page source to read the actual value, then logged in as administrator and deleted the target.

**Why it works:** Masking a value visually does not remove it from the DOM; anything sent to the client is retrievable.
**Fix:** Never send sensitive values to the client at all, masked or not — populate such fields blank and only submit changes, don't round-trip the secret.

---

## 9. Insecure direct object references
[Lab link](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)

**Vulnerability:** Static, sequential file references for a "live chat" transcript feature.
**Goal:** Read another user's (`carlos`) chat transcript to obtain their password.
**Steps:**
- Used the live chat "view transcript" feature, which requested a static file like `2.txt`.
- Changed the filename to `1.txt` and retrieved a different user's transcript, which contained their password.
- Logged in as that user.

**Why it works:** Transcripts were referenced by a simple sequential filename with no ownership check on which transcript the requester is allowed to view.
**Fix:** Map transcripts to opaque, per-session identifiers and verify the requester's session owns the referenced resource.

---

## 10. URL-based access control can be circumvented
[Lab link](https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented)

**Vulnerability:** The application (or a front-end proxy) trusts the `X-Original-URL` header without validation.
**Goal:** Reach `/admin` and delete `carlos`.
**Steps:**
- Sent a request with path `/` and added the header `X-Original-URL: /admin`.
- The server treated the request as if it targeted `/admin`, granting access.
- Repeated the technique on the delete action: path `?username=carlos` with header `X-Original-URL: /admin/delete`.

**Why it works:** Access control was enforced by inspecting the visible request path, while routing logic elsewhere honored the spoofable `X-Original-URL` header instead — the two disagreed, and the header won.
**Fix:** Perform authorization checks against the actual route the request will be routed to, after any header-based rewriting, and don't trust client-supplied routing headers from untrusted origins.

---

## 11. Method-based access control can be circumvented
[Lab link](https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented)

**Vulnerability:** Role-change endpoint only checks admin privilege for the `POST` method, not `GET`.
**Goal:** Promote a non-admin account to admin using only a non-admin session.
**Steps:**
- As admin, observed the request that upgrades a user's role (`POST` to a role-change endpoint).
- Replaced the session cookie with a low-privilege user's cookie and resent the same `POST` — got denied (unauthorized).
- Changed the HTTP method from `POST` to `GET` on that same request (keeping the low-privilege cookie) and resent it.
- The role change succeeded despite using a non-admin session.

**Why it works:** Access control was implemented per-method instead of per-endpoint, so an identical action reachable via a different verb bypassed the check entirely.
**Fix:** Enforce authorization at the route/action level regardless of HTTP method, and disable unused methods on sensitive endpoints.
