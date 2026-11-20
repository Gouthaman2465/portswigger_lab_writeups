# Cross-Site Scripting (XSS)

> PortSwigger Web Security Academy · **10/30** labs completed

Cross-site scripting lets an attacker run JavaScript in another user's browser, in the context of the vulnerable site. It can steal sessions, act as the victim, or rewrite the page.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Reflected XSS into HTML context with nothing encoded](#1-reflected-xss-into-html-context-with-nothing-encoded)
- [2. Stored XSS into HTML context with nothing encoded](#2-stored-xss-into-html-context-with-nothing-encoded)
- [3. DOM XSS in `document.write` sink using source `location.search`](#3-dom-xss-indocumentwritesink-using-sourcelocationsearch)
- [4. DOM XSS in `innerHTML` sink using source `location.search`](#4-dom-xss-ininnerhtmlsink-using-sourcelocationsearch)
- [5. DOM XSS in jQuery anchor `href` attribute sink using `location.search` source](#5-dom-xss-in-jquery-anchorhrefattribute-sink-usinglocationsearchsource)
- [6. DOM XSS in jQuery selector sink using a hashchange event](#6-dom-xss-in-jquery-selector-sink-using-a-hashchange-event)
- [7. Reflected XSS into attribute with angle brackets HTML-encoded](#7-reflected-xss-into-attribute-with-angle-brackets-html-encoded)
- [8. Stored XSS into anchor `href` attribute with double quotes HTML-encoded](#8-stored-xss-into-anchorhrefattribute-with-double-quotes-html-encoded)
- [9. Reflected XSS into a JavaScript string with angle brackets HTML encoded](#9-reflected-xss-into-a-javascript-string-with-angle-brackets-html-encoded)
- [10. DOM XSS in `document.write` sink using source `location.search` inside a select element](#10-dom-xss-indocumentwritesink-using-sourcelocationsearchinside-a-select-element)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Map every place user input is reflected or stored: URL params, form fields, headers, file names, profile fields, comments, search results.
- Identify the sink context for each: HTML body, HTML attribute, inside a `<script>` block, a JS string, a URL, or inside an event handler.
- Probe each context with a context-breaking string (e.g. `"><svg onload=alert(1)>` for HTML, `'-alert(1)-'` for a JS string) and see what survives encoding.
- Check what characters are filtered or encoded (`<`, `>`, quotes, parentheses) and whether the filter can be bypassed with case, encoding, or alternate tags/attributes.
- Distinguish reflected (in the response to this request) from stored (saved and shown to other users later, e.g. comments, usernames, support tickets) from DOM-based (never touches the server, only client-side JS).
- For DOM-based XSS, trace sources (`location`, `document.referrer`, `window.name`, `postMessage`) to sinks (`innerHTML`, `eval`, `document.write`, `setTimeout` with a string).
- Check the Content-Security-Policy header: is it present, and can it be bypassed (JSONP endpoints, allowed CDNs, missing `object-src`/`base-uri`)?
- Check cookie flags (`HttpOnly`, `SameSite`) to judge real-world impact of a working payload.
- If angle brackets are blocked, look for attribute-based or event-handler-based injection, or markdown/template injection that renders to HTML.

## Lab write-ups

### 1. Reflected XSS into HTML context with nothing encoded

**Lab:** [Reflected XSS into HTML context with nothing encoded](https://portswigger.net/web-security/cross-site-scripting/reflected/lab-html-context-nothing-encoded)

- This is a lab t demonstrate the reflected XSS into HTML.
- Here we are adding the js script in the search box and when it executes the HTML page, it executes our script
- Answer: ``<script>alert(1)</script>

---

### 2. Stored XSS into HTML context with nothing encoded

**Lab:** [Stored XSS into HTML context with nothing encoded](https://portswigger.net/web-security/cross-site-scripting/stored/lab-html-context-nothing-encoded)

- This is the lab to solve stored XSS.
- Here we are adding our script in the comment section and after that whenever a user uses that particular site, they will execute out script
- Answer: add the script in any of the comment section of a post

---

### 3. DOM XSS in `document.write` sink using source `location.search`

**Lab:** [DOM XSS in `document.write` sink using source `location.search`](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink)

- Whenever we search for anything, it is being written in the HMTL using document.write and t retrieve the data to be written using location.search.
- This html page uses <img> tag for every search.
- So to bypass this we are using a new svg element which supports events like buttons
- **so the script is- ``"><svg onload=alert(1)>**
- Whenever we search something on the search bar, it is adding to the img tag like this- ``document.write('<img src="/tracker.gif?search + location.search + ">"')
- So to attack this we are starting the script with "> because the page thinks that the img tag got closed and after that we are adding our desired script to attack this site
- For eg- ``<img src="/tracker.gif?search=shoes">`` with the starting "> we are just closing the img tag and after that SVG tag comes in
- ``<img src="/tracker.gif?search="><svg onload=alert(1)>">

---

### 4. DOM XSS in `innerHTML` sink using source `location.search`

**Lab:** [DOM XSS in `innerHTML` sink using source `location.search`](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-innerhtml-sink)

- There is a difference between the previous lab and this lab. last lab we were using document.write() to write it to the HTML page which could write the whole page but here we are using innerHTML which modifies a specific body which we wanted
- Here we cant use ``<script>`` tag because innerHTML removes it
- So there are 2 answers for this lab
	1.``<img src=0 onerror=alert(1)>
	2.``<svg onload=alert(1)>
- Both the answers work because in the first script we are deliberately giving a error tag and try to trigger the onerror feature and executing out alert
- The second is the same one as the previous lab

---

### 5. DOM XSS in jQuery anchor `href` attribute sink using `location.search` source

**Lab:** [DOM XSS in jQuery anchor `href` attribute sink using `location.search` source](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-href-attribute-sink)

- In this lab we are using href to get the cookie of the site
- Cookie contains many things like the name, session id, httponly(this one cant be seen by JS), expiry date, value(the data) domain which it belongs to, path, secure(only sent over https), samesite
- So the aims to get the document.cookie which returns the name=value pair which a JS cant see it-  for eg: name=session123, domain=example.com
- So it wont return the values that the JS cant see like httponly
- So here the lab uses jQuery which is a JS library which uses $
- $ ('#back').attr('href',location.search)
- The above command simply means that assign the value you get it for location.search to the back page
- So here our aim is to assign the document.cookie to the "back" link using alert so that we can see the values of it
- The answer is go to the feedback page and remove everything after returnPath= in the url and add ``javascript:alert(document.cookie)`` . we don't add href:... because the jQuery has already assigned href and what we get it will be assigned to href, we don't need to explicitly type href
- Now when you click back, the document.cookie will get executed. in this lab we didn't see anything because there is no info that a JS can see it like httponly

---

### 6. DOM XSS in jQuery selector sink using a hashchange event

**Lab:** [DOM XSS in jQuery selector sink using a hashchange event](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-selector-hash-change-event)

- In this lab we are going to see how to send the exploit to the victim using exploit server
- The objective of this lab is to find the vuln in auto scroll in the page which uses the hash function by location.hash
- We have to send the print function to the victim
- To achieve this first we need to understand what is the js script for hashchange
- To get this right click and view page source
- There I got this script

```html
<script>
    $(window).on('hashchange', function(){
    var post = $('section.blog-list h2:contains(' + decodeURIComponent(window.location.hash.slice(1)) + ')');
    if (post) post.get(0).scrollIntoView();});
    </script>
```

- The above script can be broken down into parts
- `$window` means look for the window moves
- Then this jquery looks for hashchange, if there is any change then run the function in that script
- `section.blog-list h2:contains` means the h2 elements which matches the exact value which was given as hash because in this webpage h2 separates every sections
- decodeuricomponent decodes the character and changes to the url form
- `window.location.hash.slice(1)) + ')'` takes the hash value and removes the # using slice(1 ) means take it from the 1st index's element
- If the #value is there in the blog as h2 tag then post will get scrolled using `scrollintoview` function and get is used to retrieve that particular h2 tag(here get(0) means go to the first occurrence of that particular hash value)
- So now we understood that script so this site takes whatever we give it as hash values
- By using this we are going to exploit this
- We can do it using `<img src=0 onerror=print()>` but we have to send the exploit to the victim
- To do this we are going to use iframe which is used to embed another HTML page
- This is the answer for this lab:`<iframe src="website url" onload= 'this.src+="<img src=0 onerror=print()>"'>`
- Post the above command in the body of the exploit server and send it to the victim

---

### 7. Reflected XSS into attribute with angle brackets HTML-encoded

**Lab:** [Reflected XSS into attribute with angle brackets HTML-encoded](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-attribute-angle-brackets-html-encoded)

- In this lab we have angle brackets encoded-means if we give `<h1>` the web page takes it as open bracket as lt and close bracket as gt. it doesn't take it as the tags
- So here our script doesn't work
- The value we are giving is going in as the value in input field
- For eg if I search for hello- `<input type="text" placeholder="Search the blog..." name="search" value="hello">` here the searched value is going into the field hello.
- So our job is to close fill some value for the field value and then use alert to exploit it
- To counter this we are going to use some events handlers in JS like onclick,onmouseover, etc as the value to fill it and then we are going to call alert
- To do this- the answer will be `"onclick="alert(1)`
- In the above answer we are adding the term onclick as value for value field and closing the quotes after = so that the value field is filled, and after that we are calling alert
- The code will be like this `<input type="text" placeholder="Search the blog..." name="search" value onclick="alert(1)">`
- So in the above input field the value for value field is closed with a " and then we call onclick="alert(1)"

---

### 8. Stored XSS into anchor `href` attribute with double quotes HTML-encoded

**Lab:** [Stored XSS into anchor `href` attribute with double quotes HTML-encoded](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-href-attribute-double-quotes-html-encoded)

- The objective of this lab is to get alert when we click the author's name in the comments section
- To do that first we are going to have some random comment and then go the view page source
- We will look at which value goes to the href value and in the next comment we are going to change that field to `javascript:alert(1)`
- In this case the url gives the href so in the next comment we are going to add the above answer in the url section

---

### 9. Reflected XSS into a JavaScript string with angle brackets HTML encoded

**Lab:** [Reflected XSS into a JavaScript string with angle brackets HTML encoded](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-html-encoded)

- In this lab we have the string in JS
- First I gave a random string to check what is happening
- I got to know that it is using document.write and uses the same input which is given as the search value

```html
<script>
        var searchTerms = 'hello';
        document.write('<img src="/resources/images/tracker.gif searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

- So in the above script it takes the entire searched element to write it in the web page so first we need to close the searchitems variable by starting the search with `'`
- Then we will give our `alert(1)`
- But still we have the ending quotes `'` so we need to remove it
- We can do it by adding // at the enf of our answer to say it as a comment line to JS
- So the final answer will be `'alert(1)//`

---

### 10. DOM XSS in `document.write` sink using source `location.search` inside a select element

**Lab:** [DOM XSS in `document.write` sink using source `location.search` inside a select element](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink-inside-select-element)

- In this lab the vuln exists inside the select tag
- First we need to come out of the select tag
- To do that we can add a `</select>` so that we come out of the select attribute. why we are doing this because event handlers wont work inside tags like select or options
- Then we can add our alert by using `<img src=1 onerror=alert(1)>`
- So by combining both the things the answer is `</select><img src=0 onerror=alert(1)>`
- We have to add this to the url
- url doesn't have the storeid so we are manually going to add the storeId parameter by & separated between prod and store id
- So the final answer will be `https://YOUR-LAB-ID.web-security-academy.net/product?productId=1&storeId=%3C/select%3E%3Cimg%20src=0%20onerror=alert(1)%3E`

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Encode output for the exact context it lands in: HTML body, HTML attribute, JavaScript string or URL.
- Avoid dangerous sinks such as `innerHTML`, `document.write()` and `eval()`. Prefer `textContent` and safe DOM APIs.
- If you must allow HTML, sanitize it with a well-maintained library such as DOMPurify, not with hand-written filters.
- Deploy a strict Content Security Policy as a second layer. Set `HttpOnly` and `SameSite` on session cookies.
- Do not rely on blocklists of tags or keywords. They are bypassed in many ways.
