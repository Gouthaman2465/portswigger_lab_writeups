# GraphQL

PortSwigger Web Security Academy — GraphQL security labs.

## 1. Accessing private GraphQL posts
[Lab link](https://portswigger.net/web-security/graphql/lab-graphql-reading-private-posts)

**Vulnerability:** A hidden blog post can be read directly via the GraphQL API by ID, bypassing the intended access restriction, and the schema's introspection is enabled, revealing an undisplayed field that holds the post's password.
**Goal:** Read the hidden post and its password.
**Steps:**
- Found the `/graphql/v1` endpoint in the HTTP history; querying it normally returned posts with IDs 1, 2, 4, 5 but skipped ID 3 (the hidden one).
- Sent the request to Repeater, then used Burp's GraphQL extension (right-click → GraphQL → Set introspection query) to dump the full schema.
- Searched the introspection result for "password" and found a field named `postPassword` not used by the normal UI.
- Modified a real "view post" GraphQL request to target `id: 3` and explicitly request the `postPassword` field.
- The response returned the hidden post's content and its password.

**Why it works:** The API enforced the hidden/visible distinction only in what the frontend chose to query and display, not in what the backend would actually return if asked directly — and introspection being enabled let the extra field be discovered in the first place.
**Fix:** Enforce object-level authorization on the resolver itself (never assume a field won't be requested just because the UI doesn't request it), and disable introspection in production.

---

## 2. Accidental exposure of private GraphQL fields
[Lab link](https://portswigger.net/web-security/graphql/lab-graphql-accidental-field-exposure)

*Attempted but not yet completed — will be added once solved.*
