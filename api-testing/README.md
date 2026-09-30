# API Testing

PortSwigger Web Security Academy — API testing labs.

## 1. Exploiting an API endpoint using documentation
[Lab link](https://portswigger.net/web-security/api-testing/lab-exploiting-api-endpoint-using-documentation)

**Vulnerability:** An undocumented-to-the-UI but publicly reachable API endpoint exposes a destructive admin action.
**Goal:** Delete the user `carlos`.
**Steps:**
- Located the API documentation, which revealed an `/api` endpoint set beyond what the UI exposed.
- Found a delete-user operation in the docs and called it directly with `carlos` as the target.

**Why it works:** The endpoint had no access control of its own; it was only "hidden" by not being linked from the UI, while still being fully documented and reachable.
**Fix:** Enforce authorization on every API endpoint regardless of whether it's linked from a UI, and don't expose destructive operations in public documentation without auth requirements clearly gated.

---

## 2. Exploiting server-side parameter pollution in a query string
[Lab link](https://portswigger.net/web-security/api-testing/server-side-parameter-pollution/lab-exploiting-server-side-parameter-pollution-in-query-string)

**Vulnerability:** User input is concatenated into an internal backend query string without proper encoding, letting an attacker inject additional parameters the backend will parse.
**Goal:** Trigger a password reset for `administrator` and capture the reset token.
**Steps:**
- Submitted `administrator` as the reset username and observed the request forwarded to `/forgot-password`, with an associated `forgot-password.js` revealing a `reset_token` field.
- Injected a URL-encoded `&` (`%26`) after the username to add an extra parameter; the app returned "parameter not supported," confirming the frontend passes the raw username through without stripping injected parameters.
- Injected a URL-encoded `#` (`%23`) to terminate the backend query early, which surfaced a "field not specified" error — confirming the backend expects a `field` parameter.
- Brute-forced the `field` parameter's name via Intruder using a list of common backend variable names, finding `email` and `username` as valid, then supplied `field=reset_token` (found in `forgot-password.js`) to leak the reset token in the response.
- Used the leaked token to reset the password and log in as administrator.

**Why it works:** The frontend validated its own known parameters but the backend query builder trusted whatever additional parameters arrived after the injected separator, letting the attacker control which internal field got returned.
**Fix:** Use parameterized queries / structured APIs internally instead of string concatenation for backend requests, and strictly validate and allow-list all parameters at every layer, not just the outermost one.

---

## 3. Finding and exploiting an unused API endpoint
[Lab link](https://portswigger.net/web-security/api-testing/lab-exploiting-unused-api-endpoint)

**Vulnerability:** A PATCH-capable price-update endpoint exists but isn't used by the UI, and has no authorization check.
**Goal:** Purchase a product for free.
**Steps:**
- Noticed an `api/products/1/price` endpoint referenced while browsing normally.
- Sent a POST to it — rejected with "only GET and PATCH allowed."
- Sent a PATCH with a JSON body `{"price": 0}` and correct `Content-Type`/`Content-Length` headers; the price updated to zero.
- Completed the purchase at no cost.

**Why it works:** The endpoint accepted state-changing PATCH requests from any authenticated (or possibly any) user with no authorization or business-logic validation on the new value.
**Fix:** Restrict write access to pricing endpoints to trusted internal services/admin roles, and validate that submitted values fall within sane bounds.

---

## 4. Exploiting a mass assignment vulnerability
[Lab link](https://portswigger.net/web-security/api-testing/lab-exploiting-mass-assignment-vulnerability)

**Vulnerability:** A checkout endpoint accepts a `chosen_discount` object it wasn't designed to expose to client control.
**Goal:** Apply a 100% discount and get a product for free.
**Steps:**
- Noticed the checkout response included a field not present in the request: `"chosen_discount": {"percentage": 0}`.
- Added that same field to the outgoing request body, setting `"percentage": 100`.
- The discount applied, reducing the price to zero.

**Why it works:** The backend bound the entire request body to an internal object that included fields (like discount percentage) never meant to be client-writable.
**Fix:** Use explicit allow-listed DTOs for request binding; never deserialize client input directly into an internal model that has more fields than intended.

---

## 5. Exploiting server-side parameter pollution in a REST URL
[Lab link](https://portswigger.net/web-security/api-testing/server-side-parameter-pollution/lab-exploiting-server-side-parameter-pollution-in-rest-url)

**Vulnerability:** User input is inserted into an internal path-style API URL without sanitizing path-traversal or terminator characters.
**Goal:** Reset the administrator's password via an internal API path and delete `carlos`.
**Steps:**
- Determined the vulnerable parameter is used in a path-style URL (not a query string) by testing how `#` and `?` characters affected the response — path segments treat these specially, query strings don't.
- Used `../` sequences to traverse upward and discover the internal API root, eventually exposing an `openapi.json` schema that revealed the real internal path template: `/api/internal/v1/users/{username}/field/{field}`.
- Crafted an input value of `../../v1/users/administrator/field/passwordResetToken%23`, which — once inserted into the `{username}` placeholder — traversed back out to the real internal endpoint and requested the `passwordResetToken` field for `administrator`.
- Retrieved the reset token from the response and used it to set a new password for `administrator`, then deleted `carlos`.

**Why it works:** Because the vulnerable value was substituted directly into a path template, path-traversal sequences let the attacker escape the intended placeholder and construct an entirely different internal API call — including one requesting a sensitive field the outward-facing API never exposes directly.
**Fix:** Never build internal request paths by string-substituting untrusted input; validate and canonicalize path segments before use, and reject traversal sequences outright.
