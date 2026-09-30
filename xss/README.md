# Cross-Site Scripting (XSS)

PortSwigger Web Security Academy — XSS labs.

## 1. Reflected XSS into HTML context with nothing encoded
[Lab link](https://portswigger.net/web-security/cross-site-scripting/reflected/lab-html-context-nothing-encoded)

**Vulnerability:** Search input is reflected into the HTML response with no encoding at all.
**Goal:** Trigger `alert(1)`.
**Steps:** Submitted `<script>alert(1)</script>` in the search box; it was reflected and executed unmodified.

**Why it works:** No output encoding means any HTML/JS in the input becomes live markup in the page.
**Fix:** HTML-encode all user input before reflecting it into an HTML context; apply a Content-Security-Policy as defense in depth.

---

## 2. Stored XSS into HTML context with nothing encoded
[Lab link](https://portswigger.net/web-security/cross-site-scripting/stored/lab-html-context-nothing-encoded)

**Vulnerability:** Comment content is stored and rendered with no encoding.
**Goal:** Trigger `alert(1)` for any visitor viewing the comment.
**Steps:** Posted `<script>alert(1)</script>` as a comment; it executed for every subsequent viewer of that page.

**Why it works:** Same missing-encoding issue as lab 1, but persisted — every viewer of the page becomes a victim, not just the submitter.
**Fix:** Encode stored content on output, regardless of what was stored; sanitize/validate on input as well where rich content isn't needed.

---

## 3. DOM XSS in `document.write` sink using source `location.search`
[Lab link](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink)

**Vulnerability:** The page reads the search query from `location.search` and writes it into the DOM via `document.write` inside an `<img>` tag, unescaped.
**Goal:** Trigger `alert(1)`.
**Steps:**
- Observed the page builds `<img src="/tracker.gif?search=<value>">` using the raw search term.
- Closed the `img` tag early with `">` and injected a new element that supports an event handler: `"><svg onload=alert(1)>`.
- The resulting markup broke out of the `img` tag and the injected `<svg>` fired its `onload` handler.

**Why it works:** `document.write` inserts raw markup with no encoding, so any HTML the attacker gets into the source string becomes live DOM.
**Fix:** Avoid `document.write` with untrusted data; use safe DOM APIs (`textContent`, attribute setters) or encode the value for the specific context before insertion.

---

## 4. DOM XSS in `innerHTML` sink using source `location.search`
[Lab link](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-innerhtml-sink)

**Vulnerability:** Search term is written into a specific element via `innerHTML`, which silently strips `<script>` tags but not other event-bearing elements.
**Goal:** Trigger `alert(1)`.
**Steps:** Injected either `<img src=0 onerror=alert(1)>` or `<svg onload=alert(1)>` — both avoid the `<script>` tag entirely and fire via an event handler instead.

**Why it works:** `innerHTML` parses and strips executable `<script>` elements as a browser safety measure, but any other tag with an inline event handler (`onerror`, `onload`, etc.) still executes normally.
**Fix:** Don't assign untrusted data to `innerHTML`; use `textContent` for plain text, or a sanitizer library (e.g. DOMPurify) if HTML must be rendered.

---

## 5. DOM XSS in jQuery anchor `href` attribute sink using `location.search` source
[Lab link](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-href-attribute-sink)

**Vulnerability:** A jQuery call (`$('#back').attr('href', location.search)`) assigns an untrusted query value directly as a link's `href`.
**Goal:** Get a `javascript:` URL to execute via that link.
**Steps:** Set the page's `returnPath=` query value to `javascript:alert(document.cookie)`, so the "back" link's `href` became that JavaScript URI; clicking it executed the script.

**Why it works:** Browsers execute `javascript:` URIs when a link using that scheme is activated; assigning attacker-controlled input directly as an `href` allows this scheme to be injected.
**Fix:** Validate that URLs assigned to `href` use an expected scheme (`http`/`https`) before setting them; never pass untrusted input directly into a URL attribute.

---

## 6. DOM XSS in jQuery selector sink using a hashchange event
[Lab link](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-selector-hash-change-event)

**Vulnerability:** A jQuery selector built from `location.hash` is evaluated whenever the URL hash changes, and can be made to run injected markup, but the attacker can't trigger the hash change themselves — it must be delivered to a victim.
**Goal:** Deliver a working exploit to a victim via the lab's exploit server.
**Steps:**
- Reviewed the page's inline script, which reacted to a `hashchange` event and matched blog post headings against `decodeURIComponent(location.hash.slice(1))`.
- Built a payload using an image error handler: `<img src=0 onerror=print()>`.
- Since the attacker can't directly cause a hash change on the victim's browser, wrapped the delivery in an iframe that sets its own `src` after load, forcing a hash change on the target page: `<iframe src="<lab-url>" onload='this.src+="<img src=0 onerror=print()>"'>`.
- Hosted this on the exploit server and delivered it to the victim.

**Why it works:** A DOM sink can be exploitable even when the attacker can't invoke it directly, as long as they can engineer another page (or event) to trigger it on the victim's behalf.
**Fix:** Sanitize/encode `location.hash` before using it in any DOM query or manipulation, regardless of how indirectly the sink is triggered.

---

## 7. Reflected XSS into attribute with angle brackets HTML-encoded
[Lab link](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-attribute-angle-brackets-html-encoded)

**Vulnerability:** Angle brackets are HTML-encoded, but the input still lands inside an unquoted-context attribute value, and event-handler attributes aren't blocked.
**Goal:** Trigger `alert(1)`.
**Steps:** Injected `"onclick="alert(1)` as the search value. Since angle brackets are encoded but quotes aren't, this closed the `value` attribute and added a new `onclick` attribute executing on click.

**Why it works:** Encoding only `<` and `>` doesn't stop attribute-breakout attacks that work purely with quote characters and additional attribute names.
**Fix:** HTML-encode quote characters as well (or use attribute-safe encoding libraries), and avoid reflecting user input into HTML attributes at all where possible.

---

## 8. Stored XSS into anchor `href` attribute with double quotes HTML-encoded
[Lab link](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-href-attribute-double-quotes-html-encoded)

**Vulnerability:** A stored value (comment author URL) is placed into an `href` attribute; double quotes are encoded, but the `javascript:` scheme itself isn't blocked.
**Goal:** Trigger `alert(1)` when a victim clicks the comment author's name.
**Steps:** Set the comment's URL/website field to `javascript:alert(1)`; when rendered as the author link's `href` and clicked, it executed.

**Why it works:** Encoding quotes prevents attribute breakout, but doesn't stop a dangerous URL scheme from being accepted as a valid attribute value in the first place.
**Fix:** Validate that URL-type fields only accept `http`/`https` schemes before storing or rendering them.

---

## 9. Reflected XSS into a JavaScript string with angle brackets HTML encoded
[Lab link](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-html-encoded)

**Vulnerability:** Search input is reflected inside a JavaScript string literal that's then passed to `document.write`; angle brackets are encoded but quotes are not.
**Goal:** Trigger `alert(1)`.
**Steps:**
- Identified the reflection point: `var searchTerms = '<input>'; document.write(...)`.
- Closed the string literal with a single quote, added `alert(1)`, and commented out the rest of the line with `//` to avoid a syntax error from the original trailing quote: `'alert(1)//`.

**Why it works:** HTML-encoding doesn't protect a JavaScript string context — a raw quote character can still terminate the string and let injected JavaScript execute.
**Fix:** Use JavaScript-string-safe encoding (escape quotes and special characters) for any value embedded inside a `<script>` block, or avoid embedding user input in inline scripts entirely.

---

## 10. DOM XSS in `document.write` sink using source `location.search` inside a select element
[Lab link](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink-inside-select-element)

**Vulnerability:** Same `document.write`/`location.search` sink as lab 3, but the injection point is inside a `<select>` element, where event-handler attributes don't fire.
**Goal:** Trigger `alert(1)` despite the restrictive `<select>` context.
**Steps:**
- Closed out of the `<select>` element first with `</select>`, since event handlers on elements inside `<select>`/`<option>` don't execute.
- Followed with a working payload: `<img src=0 onerror=alert(1)>`.
- Combined: `</select><img src=0 onerror=alert(1)>`, URL-encoded and added as a `storeId` parameter in the product URL (which wasn't present by default and had to be added manually).

**Why it works:** Escaping the restrictive parent element first restores a normal HTML context where standard event-handler payloads work again.
**Fix:** Context-aware output encoding for wherever user input lands, including inside form elements like `<select>`; don't assume a specific parent tag limits what an attacker can inject around it.
