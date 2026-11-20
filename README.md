# PortSwigger Web Security Academy — Lab Write-ups

My step-by-step notes and write-ups for the [PortSwigger Web Security Academy](https://portswigger.net/web-security) labs, one file per vulnerability class.

## Progress

**132 completed** across 21 topics.

### Injection

| Topic | Completed | Progress | Write-up |
|---|:---:|---|---|
| NoSQL Injection | 4 / 4 | ██████████ 100% | [nosql-injection.md](./nosql-injection.md) |
| OS Command Injection | 3 / 5 | ██████░░░░ 60% | [os-command-injection.md](./os-command-injection.md) |
| SQL Injection | 10 / 18 | ██████░░░░ 56% | [sqli.md](./sqli.md) |
| Server-Side Template Injection (SSTI) | 5 / 7 | ███████░░░ 71% | [ssti.md](./ssti.md) |
| XML External Entity (XXE) Injection | 6 / 9 | ███████░░░ 67% | [xxe.md](./xxe.md) |

### Client-side

| Topic | Completed | Progress | Write-up |
|---|:---:|---|---|
| Clickjacking | 4 / 5 | ████████░░ 80% | [clickjacking.md](./clickjacking.md) |
| Cross-Origin Resource Sharing (CORS) | 2 / 3 | ███████░░░ 67% | [cors.md](./cors.md) |
| Cross-Site Request Forgery (CSRF) | 10 / 11 | █████████░ 91% | [csrf.md](./csrf.md) |
| Cross-Site Scripting (XSS) | 10 / 30 | ███░░░░░░░ 33% | [xss.md](./xss.md) |
| DOM-Based Vulnerabilities | 6 / 7 | █████████░ 86% | [dom-based.md](./dom-based.md) |

### Server-side

| Topic | Completed | Progress | Write-up |
|---|:---:|---|---|
| File Upload Vulnerabilities | 7 / 7 | ██████████ 100% | [file-upload.md](./file-upload.md) |
| Path Traversal | 6 / 6 | ██████████ 100% | [path-traversal.md](./path-traversal.md) |
| Server-Side Request Forgery (SSRF) | 5 / 7 | ███████░░░ 71% | [ssrf.md](./ssrf.md) |

### Authentication & Access

| Topic | Completed | Progress | Write-up |
|---|:---:|---|---|
| Access Control Vulnerabilities | 11 / 13 | ████████░░ 85% | [access-control.md](./access-control.md) |
| Authentication Vulnerabilities | 5 / 14 | ████░░░░░░ 36% | [authentication.md](./authentication.md) |
| JWT Attacks | 6 / 8 | ████████░░ 75% | [jwt.md](./jwt.md) |

### API & Logic

| Topic | Completed | Progress | Write-up |
|---|:---:|---|---|
| API Testing | 5 / 5 | ██████████ 100% | [api-testing.md](./api-testing.md) |
| Business Logic Vulnerabilities | 10 / 12 | ████████░░ 83% | [business-logic.md](./business-logic.md) |
| GraphQL API Vulnerabilities | 5 / 5 | ██████████ 100% | [graphql.md](./graphql.md) |
| Information Disclosure | 5 / 5 | ██████████ 100% | [information-disclosure.md](./information-disclosure.md) |
| Insecure Deserialization | 7 / 10 | ███████░░░ 70% | [insecure-deserialization.md](./insecure-deserialization.md) |

## What is in each write-up

- A short description of the vulnerability class.
- Every lab I solved, in order, with a link to the lab and the steps I followed (payloads, requests and scripts).
- A **Prevention** section at the end with how the bug is fixed.

## Tools and method

Manual testing with Burp Suite (Proxy, Repeater, Intruder, Decoder), browser DevTools for client-side logic, and small Python scripts where automation helped (for example brute force through GraphQL aliases, blind data extraction, and JWT forging).

## Notes

- Only labs I solved are written up. Labs I have not finished are not listed, so the numbers above stay honest. Some of the remaining labs need Burp Suite Professional.
- Lab IDs and exploit-server addresses in the notes are replaced with placeholders, because PortSwigger generates a new one every time a lab starts.
- These write-ups contain solutions. Try each lab on your own first.
- Everything here was done against PortSwigger's own intentionally vulnerable labs. Do not test systems you do not have permission to test.

## Contact

GitHub: [Gouthaman2465](https://github.com/Gouthaman2465) · LinkedIn: [gouthaman-g](https://www.linkedin.com/in/gouthaman-g-0296b1288) · Portfolio: [gouthaman.me](https://gouthaman.me)
