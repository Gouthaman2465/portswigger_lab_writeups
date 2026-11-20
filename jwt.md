# JWT Attacks

> PortSwigger Web Security Academy · **6/8** labs completed

JWT attacks abuse weak token handling: unverified signatures, the `none` algorithm, weak secrets, and attacker-controlled header parameters such as `jwk`, `jku` and `kid`.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. JWT authentication bypass via unverified signature](#1-jwt-authentication-bypass-via-unverified-signature)
- [2. JWT authentication bypass via flawed signature verification](#2-jwt-authentication-bypass-via-flawed-signature-verification)
- [3. JWT authentication bypass via weak signing key](#3-jwt-authentication-bypass-via-weak-signing-key)
- [4. JWT authentication bypass via jwk header injection](#4-jwt-authentication-bypass-via-jwk-header-injection)
- [5. JWT authentication bypass via jku header injection](#5-jwt-authentication-bypass-via-jku-header-injection)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Decode the JWT and check: is the signature actually verified by the server at all? Try modifying a claim and resending unmodified-looking signature.
- Test the `none` algorithm: strip the signature and set `alg: none`.
- Check if the signing algorithm can be swapped between asymmetric and symmetric (RS256 to HS256), signing with the public key as an HMAC secret, if the server trusts the `alg` header from the token itself.
- If HMAC-signed, try cracking the secret offline against a wordlist.
- Check header fields that might be trusted for key lookup without validation: `jwk` (embedded key), `jku` (URL to a key set you could host), `kid` (path/injection into key lookup, e.g. pointing at a predictable file or using SQLi in the lookup).
- Check claim validation: is `exp` enforced, is `aud`/`iss` checked, can an expired or wrong-audience token still be used?
- Check if a token issued for one purpose/endpoint is accepted on another (token confusion between services).

## Lab write-ups

### 1. JWT authentication bypass via unverified signature

**Lab:** [JWT authentication bypass via unverified signature](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature)

**Vulnerability:** The server decodes the JWT payload but never verifies the signature.

**Goal:** Access the admin panel and delete `carlos`.

**Steps:**
- Logged in normally and captured the issued JWT.
- In Burp Repeater, edited the payload's `sub` claim from `wiener` to `administrator`, changed the request path to `/admin`, and sent it — no re-signing needed.
- Located the delete link exposed on the admin page and requested it directly.

**Why it works:** Since the signature is never checked, any claim in the payload can be edited freely and the server will still trust it.

**Fix:** Always verify the JWT signature server-side against a known key before trusting any claim in the payload.

---

### 2. JWT authentication bypass via flawed signature verification

**Lab:** [JWT authentication bypass via flawed signature verification](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification)

**Vulnerability:** The server accepts tokens with `alg` set to `none` and no signature.

**Goal:** Access the admin panel and delete `carlos`.

**Steps:**
- Took a valid JWT and set the header's `alg` to `none`.
- Set `sub` to `administrator`, then removed the signature entirely, leaving a trailing period after the payload.
- Sent the token and reached the admin delete function.

**Why it works:** Some JWT libraries treat `alg: none` as a legitimate "unsecured" token type; if the server doesn't explicitly reject it, signature verification is skipped altogether.

**Fix:** Explicitly reject the `none` algorithm server-side and enforce an allow-list of accepted algorithms.

---

### 3. JWT authentication bypass via weak signing key

**Lab:** [JWT authentication bypass via weak signing key](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key)

**Vulnerability:** The HMAC secret used to sign tokens is a common, guessable string.

**Goal:** Forge a valid admin token and delete `carlos`.

**Steps:**
- Captured a valid JWT from a logged-in session.
- Cracked the signing secret offline against a wordlist using `hashcat -a 0 -m 16500 <jwt> <wordlist>`.
- Used the recovered secret to sign a new token with `sub: administrator` in Python (PyJWT), then submitted it and reached `/admin`.

```python
import jwt

payload = {
    "iss": "portswigger",
    "exp": 1782852024,
    "sub": "administrator"
}

token = jwt.encode(payload, "<recovered-secret>", algorithm="HS256")
print(token)
```

**Why it works:** HMAC-signed JWTs are only as strong as the secret. A weak or wordlist-guessable secret lets an attacker forge arbitrary valid tokens once cracked.

**Fix:** Use a long, high-entropy signing secret (or move to asymmetric signing), and rotate it if compromise is suspected.

---

### 4. JWT authentication bypass via jwk header injection

**Lab:** [JWT authentication bypass via jwk header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection)

**Vulnerability:** The server trusts an embedded `jwk` header to supply the public key used for verification, instead of using its own trusted key.

**Goal:** Forge a self-signed admin token that the server still accepts.

**Steps:**
- Generated an RSA key pair locally:

```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

- Converted the public key to JWK format with `jwcrypto`.
- Built a token whose header carries `alg: RS256` and a `jwk` field containing the attacker's own public key, and whose payload sets `sub: administrator`.
- Signed the token with the matching private key and submitted it to `/admin`, then reached the delete function.

```python
import jwt

jwk = {"kty": "RSA", "kid": "<kid>", "e": "AQAB", "n": "<modulus>"}
headers = {"alg": "RS256", "kid": jwk["kid"], "jwk": jwk}
payload = {"iss": "portswigger", "exp": 9999999999, "sub": "administrator"}

with open("private.pem") as f:
    private_key = f.read()

token = jwt.encode(payload, private_key, algorithm="RS256", headers=headers)
print(token)
```

**Why it works:** If the server blindly uses whatever public key is embedded in the token's own header to verify that same token's signature, the attacker can supply both halves of the key pair and the check always passes.

**Fix:** Never accept a key for verification from the token itself. Verify against a fixed, pre-configured key or a `kid` value checked against a trusted, server-side key store (JWKS allow-list).

---

### 5. JWT authentication bypass via jku header injection

**Lab:** [JWT authentication bypass via jku header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jku-header-injection)

**Vulnerability:** The server fetches the verification key from a URL supplied in the token's own `jku` (JWK Set URL) header, without restricting which hosts are trusted.

**Goal:** Host a malicious key set and get the server to trust a self-signed admin token.

**Steps:**
- Generated an RSA key pair and converted the public key to JWK format, as in lab 4.
- Wrapped it in a JWKS document (`{"keys": [ ... ]}`) and hosted it on the lab's exploit server.
- Built a token with header `jku` pointing at the hosted JWKS URL and the matching `kid`, payload `sub: administrator`, signed with the private key.
- Submitted the token; the server fetched the attacker-hosted key and verified successfully.

```python
import jwt, time

headers = {
    "alg": "RS256",
    "kid": "<kid>",
    "jku": "<attacker-hosted-jwks-url>",
    "typ": "JWT",
}
payload = {"iss": "portswigger", "sub": "administrator", "exp": int(time.time()) + 3600}

with open("private.pem") as f:
    private_key = f.read()

token = jwt.encode(payload, private_key, algorithm="RS256", headers=headers)
print(token)
```

**Why it works:** Trusting a client-supplied URL for key retrieval means an attacker only needs to host their own valid key set for the server to treat their forged tokens as legitimate.

**Fix:** Restrict `jku` to a strict allow-list of trusted hosts (ideally hardcode the JWKS endpoint), and never resolve it from unauthenticated input.

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Always verify the signature, and reject tokens with the `none` algorithm.
- Fix the expected algorithm on the server. Do not take it from the token header.
- Use long random secrets (or proper key pairs) and rotate them.
- Do not trust keys supplied in the token (`jwk`, `jku`, `kid`). Resolve them only from a trusted allowlist.
- Validate claims such as `exp`, `iss` and `aud`, and keep token lifetimes short.
