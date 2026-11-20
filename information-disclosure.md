# Information Disclosure

> PortSwigger Web Security Academy · **5/5** labs completed

Information disclosure happens when an application reveals sensitive data it shouldn't — version numbers, debug output, source code, old commits, or internal headers — that helps an attacker understand and attack the system further.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Information disclosure in error messages](#1-information-disclosure-in-error-messages)
- [2. Information disclosure on debug page](#2-information-disclosure-on-debug-page)
- [3. Source code disclosure via backup files](#3-source-code-disclosure-via-backup-files)
- [4. Authentication bypass via information disclosure](#4-authentication-bypass-via-information-disclosure)
- [5. Information disclosure in version control history](#5-information-disclosure-in-version-control-history)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Trigger errors deliberately (invalid input, wrong types, missing parameters) and read the full error message, not just the status code — look for stack traces, file paths, and version numbers.
- Check `/robots.txt`, `/sitemap.xml`, and the site map for files and paths that aren't linked from the UI (config files, debug pages, backup endpoints).
- Look for debug or diagnostic pages left enabled in production (`phpinfo.php`, `/debug`, `/status`), which often leak secrets, environment variables, or internal paths.
- Check for backup and temporary files next to real source files (`.bak`, `.old`, `~`, `.js.bak`) that may contain credentials or logic the live version hides.
- Test unusual HTTP methods like `TRACE` on sensitive endpoints — some servers echo the full request back, revealing headers added by proxies or load balancers that are normally invisible to the client.
- Check whether an internal-only header (e.g. an IP-allowlist header) is trusted blindly, and whether you can add that header yourself with the right value to pass as an internal/local request.
- Check for an exposed `.git` directory — if present, the entire commit history (including old, removed secrets) can be downloaded and inspected offline.
- Compare responses for authenticated vs unauthenticated, and admin vs normal user, to see if extra fields or detail leak out before access control actually kicks in.

## Lab write-ups

### 1. Information disclosure in error messages

**Lab:** [Information disclosure in error messages](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-in-error-messages)

The error exposes the version number, and we have to find it using a triggered error.

- Find any product and look at its URL — it takes an integer ID.
- Replace that ID with some random, invalid value. The resulting error message leaks the application's version number.
- The version disclosed is `2 2.3.31`.

---

### 2. Information disclosure on debug page

**Lab:** [Information disclosure on debug page](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-on-debug-page)

We have to find the secret key from a debug file.

- To get the file, go to the site map. There is a PHP file called `phpinfo.php`.
- View the file at `/cgi-bin/phpinfo.php` and search the page for "secret" — the secret key is right there.

---

### 3. Source code disclosure via backup files

**Lab:** [Source code disclosure via backup files](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-via-backup-files)

We have to find a backup file to get the database password.

- Add `/robots.txt` to the URL. It may contain references to important config paths.
- It exposes where the backup file endpoint is: `/backup`.
- Go to `/backup` to find a JS file belonging to the site.
- Open the file — the database password is on the SQL connect line.
- The password is `s12gkgleb3adhewhj2m060rjnrmb97ov`.

---

### 4. Authentication bypass via information disclosure

**Lab:** [Authentication bypass via information disclosure](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-authentication-bypass)

This lab was new to me — I used the Match and Replace feature under Proxy for the first time.

- First, try to visit `/admin`. It says access is only allowed for local users, meaning the server should think the request is coming from `localhost` / `127.0.0.1`.
- So we have to make the server think our request is coming from localhost.
- To do that, first send the `/admin` request using the `TRACE` method, which echoes all the headers back in the response. This reveals a hidden header the server itself is adding to our request.
- In the echoed response there's a custom header: `X-Custom-IP-Authorization:`, set to my own IP address.
- Take that header and go to Match and Replace under Proxy settings.
- Click **Add** under HTTP match and replace rules. In the popup, leave **Match** empty, set **Type** to **Request header**, and set **Replace** to:

  ```http
  X-Custom-IP-Authorization: 127.0.0.1
  ```

- Click OK. From now on, Burp adds this header with the value `127.0.0.1` to every request.
- Refresh the `/admin` page — access is granted. Delete Carlos's account.

---

### 5. Information disclosure in version control history

**Lab:** [Information disclosure in version control history](https://portswigger.net/web-security/information-disclosure/exploiting/lab-infoleak-in-version-control-history)

Version control history is a record of every old commit — how the site evolved from scratch to now — so it can contain sensitive info like passwords that were later removed from the live code.

- We have to get access to it and find the admin password.
- To get the control history, first browse to `/.git` to confirm the directory is exposed.
- Download the whole directory using `wget`:

  ```bash
  wget -r https://YOUR-LAB-ID.web-security-academy.net/.git/
  ```

- Download a Git GUI tool — I used Git Cola to view the history.
- Open the downloaded folder in Git Cola. Under **Commit**, undo the last commit. `admin.conf` turns green in the sidebar (meaning it was removed in a later commit, but still exists in history).
- Click the file to reveal the admin password. Log in as admin and delete Carlos's account.

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Return generic error messages to users. Log full stack traces, versions, and internal paths server-side only — never render them in the response.
- Remove debug pages, diagnostic endpoints, and framework info pages (like `phpinfo.php`) before deploying to production.
- Never leave backup, temporary, or editor swap files in a web-accessible directory. Exclude them at the build/deploy level, not just by hiding the link.
- Don't trust client-supplied headers for access control, even ones that look internal (`X-Custom-IP-Authorization`, `X-Forwarded-For`). Enforce IP-based restrictions at the network layer, not by trusting a header the client can set.
- Disable the `TRACE` method (and other unused HTTP methods) at the web server level.
- Never commit secrets to version control, even temporarily. If a secret is ever committed, rotate it — removing it in a later commit does not remove it from history.
- Keep `.git` and other VCS directories out of the web root entirely, and verify this in deployment, not just in `robots.txt`.
