# PortSwigger Web Security Academy — Lab Write-ups

Notes and write-ups from working through PortSwigger's Web Security Academy,
covering the Apprentice and Practitioner tiers. Each lab includes the
vulnerability class, the goal, the exploitation steps, why the bug exists,
and a suggested fix — since detection without remediation is only half the job.

## Topics

| Topic | Labs completed | Link |
|---|---|---|
| Access Control | 11 | [access-control/](./access-control/) |
| API Testing | 5 | [api-testing/](./api-testing/) |
| Authentication | 5 (of 6) | [authentication/](./authentication/) |
| CSRF | 2 mechanisms (3 PoCs) | [csrf/](./csrf/) |
| GraphQL | 1 (of 2) | [graphql/](./graphql/) |
| JWT | 5 (of 8) | [jwt/](./jwt/) |
| Path Traversal | 6 | [path-traversal/](./path-traversal/) |
| SQL Injection | 10 | [sql-injection/](./sql-injection/) |
| XSS | 10 | [xss/](./xss/) |

**Total: ~55 labs.**

## Methodology

Manual testing with Burp Suite (Repeater, Intruder, and the GraphQL
extension for introspection), browser DevTools for inspecting client-side
logic, and — for a couple of the JWT labs — small Python scripts (PyJWT,
`jwcrypto`) to forge and re-sign tokens once a weakness in verification
was found.

## Notes on scope

A few labs are listed as "attempted but not yet completed" in their topic
files — those are left in deliberately rather than deleted, so the
progress here stays honest. They'll be filled in as they're solved.
