# Path Traversal

PortSwigger Web Security Academy — file path traversal labs.

## 1. File path traversal, simple case
[Lab link](https://portswigger.net/web-security/file-path-traversal/lab-simple)

**Vulnerability:** An image-loading endpoint concatenates user input directly into a filesystem path.
**Goal:** Read `/etc/passwd`.
**Steps:**
- Found the request that loads product images and sent it to Repeater.
- Replaced the filename with `../../../etc/passwd`, walking up from the images directory until the traversal reached the filesystem root.
- The response returned the contents of `/etc/passwd`.

**Why it works:** The application trusted the filename parameter completely and passed it straight to a file-read call with no path normalization or restriction to a base directory.
**Fix:** Resolve the requested path, canonicalize it, and verify it still falls inside the intended base directory before reading; reject any path containing `../`.

---

## 2. File path traversal, traversal sequences blocked with absolute path bypass
[Lab link](https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass)

**Vulnerability:** Relative traversal sequences (`../`) are blocked, but absolute paths are not.
**Goal:** Read `/etc/passwd`.
**Steps:**
- Confirmed `../` sequences were stripped or rejected.
- Supplied the absolute path `/etc/passwd` directly as the filename value.
- The file was returned.

**Why it works:** The filter only accounted for one bypass technique (relative traversal) and didn't consider that many file-read APIs will honor an absolute path just as readily.
**Fix:** Validate that the resolved path is a *descendant* of the intended directory, rather than pattern-matching against `../`.

---

## 3. File path traversal, traversal sequences stripped non-recursively
[Lab link](https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively)

**Vulnerability:** The filter strips `../` sequences, but only once per occurrence rather than recursively.
**Goal:** Read `/etc/passwd`.
**Steps:**
- Submitted `....//....//....//etc/passwd`.
- After a single non-recursive strip of `../`, this collapses to `../../../etc/passwd`.
- The file was returned.

**Why it works:** A single-pass string replacement can be defeated by nesting the pattern so that removing the first occurrence produces a new instance of the same pattern.
**Fix:** Strip traversal sequences in a loop until no further changes occur (or, better, canonicalize the path and check containment rather than filtering strings).

---

## 4. File path traversal, traversal sequences stripped with superfluous URL-decode
[Lab link](https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode)

**Vulnerability:** The application URL-decodes input twice before the traversal filter runs.
**Goal:** Read `/etc/passwd`.
**Steps:**
- Recognized that normal single URL-encoding (`../` → `%2e%2e%2f`) would be caught by the filter after one decode pass.
- Double-encoded the payload instead, so the first decode pass only reveals another encoded layer: `%252e%252e%252f...` → decodes once to `%2e%2e%2f...`, then decodes again (server-side) to `../`.
- Supplied `%252e%252e%252f%252e%252e%252f%252e%252e%252fetc%252fpasswd`, which reached the filesystem call as `../../../etc/passwd` only after the filter had already run.

**Why it works:** The traversal filter ran on the input after only one decode pass, but the server actually decoded the value twice before using it — so a payload encoded twice arrives at the filesystem call in fully traversed form, after the filter has already passed it.
**Fix:** Decode input fully before any security filtering runs, and only decode once — don't apply decoding at multiple layers of the stack without validating in between.

---

## 5. File path traversal, validation of start of path
[Lab link](https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path)

**Vulnerability:** The application only checks that the supplied path *starts with* an expected base directory, then uses the rest of the string unchanged.
**Goal:** Read `/etc/passwd`.
**Steps:**
- Confirmed the filter required the path to begin with a valid directory name such as `var` or `images`.
- Supplied `var/www/images/../../../etc/passwd` — it starts with a valid-looking prefix, satisfying the check, but still fully traverses out via the trailing `../` sequences.

**Why it works:** Validating only the start of a string says nothing about what the rest of the string does; the traversal sequences after the valid prefix were never inspected.
**Fix:** Canonicalize the entire path and verify the final resolved location is inside the allowed directory — don't validate on a prefix match alone.

---

## 6. File path traversal, validation of file extension with null byte bypass
[Lab link](https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass)

**Vulnerability:** The application requires the path to end in an image extension, but the underlying (C-based) file-handling layer treats a null byte as a string terminator.
**Goal:** Read `/etc/passwd` while satisfying the ".jpg/.png" extension check.
**Steps:**
- Confirmed the application layer validated only that the string ended in a valid image extension.
- Appended a URL-encoded null byte before a fake extension: `../../../etc/passwd%00.jpg`.
- The application-layer check saw the required `.jpg` suffix and passed the value through; the underlying OS/C file API stopped reading the filename at the null byte, actually opening `/etc/passwd`.

**Why it works:** Two layers of the stack disagreed about where the string ends — the validation layer read the whole string (extension check passes), while the file-open layer stopped at the embedded null byte (actual path becomes `/etc/passwd`).
**Fix:** Reject any input containing null bytes outright, and perform extension/type validation on the same fully-resolved, canonicalized value that will actually be used to open the file.
