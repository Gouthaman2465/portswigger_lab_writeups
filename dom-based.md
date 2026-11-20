# DOM-Based Vulnerabilities

> PortSwigger Web Security Academy · **6/7** labs completed

DOM-based vulnerabilities happen when client-side JavaScript takes data from an attacker-controlled source (URL, `postMessage`, storage) and passes it to a dangerous sink.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [Concept notes](#concept-notes)
- [1. DOM XSS using web messages](#1-dom-xss-using-web-messages)
- [2. DOM XSS using web messages and a JavaScript URL](#2-dom-xss-using-web-messages-and-a-javascript-url)
- [3. DOM XSS using web messages and `JSON.parse`](#3-dom-xss-using-web-messages-andjsonparse)
- [4. DOM-based open redirection](#4-dom-based-open-redirection)
- [5. DOM-based cookie manipulation](#5-dom-based-cookie-manipulation)
- [6. Exploiting DOM clobbering to enable XSS](#6-exploiting-dom-clobbering-to-enable-xss)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- List every client-side source an attacker can influence: URL (`location.search`, `location.hash`, `location.href`), `document.referrer`, `window.name`, `postMessage`, local/session storage.
- List every dangerous sink in the page's JS: `innerHTML`, `outerHTML`, `document.write`, `eval`, `setTimeout`/`setInterval` with a string, `location`/`location.href` assignment, `.src` on script/iframe tags.
- Trace whether any source reaches a sink without sanitization, by reading the JS (view-source, DevTools, or Burp's DOM Invader).
- For `postMessage` handlers, check whether `event.origin` is validated at all, and if so, whether it's an exact match or a weak check (`indexOf`, `startsWith`, `includes` without anchoring).
- Check whether the message's data is validated by type/shape before being used, not just its origin.
- Check client-side routing/navigation logic for open-redirect-style DOM vulnerabilities (`location = userInput`).
- Check for DOM-based variants of standard bugs: DOM XSS, DOM-based open redirect, DOM-based cookie manipulation, and client-side prototype pollution feeding into a sink.

## Concept notes

- event.data- tells what data comes in
- event.origin- tells the origin it came from
- event.source- tells the  reference window

#### mistakes for validating the origin
- Using indexOf- which checks whether the given sting is there in a string
- Using startsWith- it checks whether the string starts with the value given
- endsWith- checks whether the string endswith the given value

#### secure way to do it

- Use includes to check t
- First add a list with the whitelisted origins
- Then check whether the incoming origin includes in the whitelisted one

#### source- place where the data came from

- location.search
- location.hash
- location.href

#### sinks- the potential endpoint where the data processing could be dangerous

- .innerhtml
- .outerhtml
- eval()
- document.write()
- document.cookie()
#### document.domain is used to bypass the SOP by the parent page
this could be done when the parent and child page sets their `document.domain= "example.com"`  and now parent page will be able to access the DOM of the child page and vice versa

the above method is blocked everywhere in modern sites and the new way is to use postMessage
postMessage is used to communicate between cross origin sites to transfer some info that depends on the further operation on the main site
eg- `window.parent.postMessage(dataToSend, "https://example.com");`

### web manipulation

#### 1. HTML injection- the attacker sends the malicious html code like alert and triggers it using innerhtml sink
#### 2. js execution- using eval sink so that the user controlled input just runs as the js code

#### 3. navigation control- the site validates the user controlled input so that the app may redirect to the malicious site

#### 4. application state change- the attacker tries to change the role from user to admin but this is not enough to change the privileges in the server but could be a dangerous one if the server validates the authorization using the client side permission

### steps in validating the messages

- Origin validation
- Data type validation
- Operation validation
- Safe DOM operation like using textcontent instead of innerhtml

### steps to test the message manipulation

- Identify the message listeners
- See the data processing methods
- Look for an vulnerable source or sinks
- Inspect the origin validation
- Check the data type validation
- Check the safe DOM operation
- Check whether the payload is reaching the dangerous endpoint so that we could attack there

DOM data manipulation occurs when JavaScript modifies DOM-related data in a way that can affect application behaviour or security.

## Lab write-ups

### 1. DOM XSS using web messages

**Lab:** [DOM XSS using web messages](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages)

- This lab has the eventlistener and has innerhtml in postmessages.
- So to exploit it we need to build an iframe which implements the postMessage where it should have the img tag with the print called in there
- To do that first we need to build an iframe where the syntax is `<iframe src="" onload="this.contentWindow,postMessage('vuln code', '*')">`
- The payload is `<iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('<img src=1 onerror=print()>', '*')"> `

---

### 2. DOM XSS using web messages and a JavaScript URL

**Lab:** [DOM XSS using web messages and a JavaScript URL](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages-and-a-javascript-url)

- In this lab the vuln exists in the eventlistener where it used indexOf with the condition of occurrence of http: || https: so we need to craft the iframe accordingly using JS
- So the payload is `<iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload ="this.contentWindow.postMessage('javascript:print()//http:', '*')">`
- We are giving `javascript:print//http:` because we need to ensure 2 things, the url should contain http and the print payload
- So to do that we are using javascript keyword where it tells the browser to execute it as js code
- So when we give this line in postmessage, the javascript converts the line into print() because // is used for single line comment in js. we have added the http: and the payload so that's the reason we are using javascript keyword to change the text into js code

---

### 3. DOM XSS using web messages and `JSON.parse`

**Lab:** [DOM XSS using web messages and `JSON.parse`](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages-and-json-parse)

- In this lab json parse is parsing the input and directly feeding the input in the innerhtml sink
- So to exploit it we need to write the json file
- First we will visit the source code of the page and we can see that under eventlistener there are 2 switch cases and one is load-channel and under that the input is getting directly fed, so we need to write the json file in such a way that, we are going to use load-channel and then add the print function under the key called url
- To do that this is the payload

```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>
```

- The reason for using `\` in json file is because in js, the quotes will be misjudged. here there is a quote at the starting of ht json right before the `{` so it thinks that the first quote starts  before`{` and the next time it sees the quotes, it thinks that we should close the quotes so the result will be `"{"`  but this is not we want, so to do that, we are using `\` to escape that scenario and usage of `\` makes it as a literal quote not to finish the previous quote

---

### 4. DOM-based open redirection

**Lab:** [DOM-based open redirection](https://portswigger.net/web-security/dom-based/open-redirection/lab-dom-open-redirection)

- In this lab the vuln exists in redirection.first go to any of the posts and view the source code. there you can see the code to return to the home page has the redirection vuln

```html
<a href='[#](https://YOUR-LAB-ID.web-security-academy.net/post?postId=9#)' onclick='returnUrl = /url=(https?:\/\/.+)/.exec(location); location.href = returnUrl ? returnUrl[1] : "/"'>Back to Blog</a>
```

- The above code is the vuln code
- This is vuln because the url is just checking whether the given url has http: or https: and that's it
- If there is a url then redirect to that page else go to the home page
- So now the attacker can send any url and make the DOM to redirect to his page
- So the payload is

```
https://YOUR-LAB-ID.web-security-academy.net/post?postId=9&url=https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/exploit
```

- Now visit the above payload on the browser, since exploit server is the server you control so you are giving the malicious redirect as your server

---

### 5. DOM-based cookie manipulation

**Lab:** [DOM-based cookie manipulation](https://portswigger.net/web-security/dom-based/cookie-manipulation/lab-dom-cookie-manipulation)

- In this lab the site is storing the previously view product as cookie and the cookie value is the url of it
- It is just appending the url to it so we need to make sure the inject print() with something the site doesn't do it previously.
- The payload is `<iframe src="https://YOUR-LAB-ID.web-security-academy.net/product?productId=1&'><script>print()</script>" onload="if (!window.x)this.src='https://YOUR-LAB-ID.web-security-academy.net/';window.x=1">`
- The payload tells us that it is setting the cookie value to the malicious url which was given as the scr for iframe, in this case it is /productId=1 and we have successfully poisoned the cookie
- Then we have created the print statement and successfully injected it
- The use of `&` here is after & everything is considered as the next parameter so after that the cookie value is over and the script is executed
- And when in on loading the iframe, we are redirecting it to the root directory because that's where the cookie functionality gets executed and its there
- The condition for onload is checking whether the variable x on window is not true and initially it will be undefined so the condition becomes true and we are adding the root directory to the src and after that to avoid repeating the same condition we are making x=1 to avoid the loop

---

### 6. Exploiting DOM clobbering to enable XSS

**Lab:** [Exploiting DOM clobbering to enable XSS](https://portswigger.net/web-security/dom-based/dom-clobbering/lab-dom-xss-exploiting-dom-clobbering)

- In this lab we are going to clobber a already pre existing svg file which is defaultAvatar
- To do that the below line is the payload
- `<a id=defaultAvatar><a id=defaultAvatar name=avatar href="cid:&quot;onerror=alert(1)//">`
- After posting the comment with the above payload now submit another comment with random value and return to the post page and you will see the error pop up
- We use two `<a>` tags with the same `id="defaultAvatar"` because the browser groups elements sharing the same ID into a DOM collection and exposes that collection through `window.defaultAvatar`. The first anchor helps establish this collection, while the second anchor introduces the `name="avatar"` attribute, which creates a named property within the collection. Since the second anchor also contains our malicious `href`, we can access that element through `window.defaultAvatar.avatar` and make the application use our attacker-controlled value instead of its default avatar object. This manipulation of the expected JavaScript object through HTML elements is called DOM clobbering.
- The reason behind using cid is it is a protocol where it doesn't url encode the double quotes, so we can add the encoded double quote
- The href part is interesting because when we try to add `cid:"onerror=alert(1)//`, this will get passed to a lets say img tag. this becomes `<img src="cid:"onerror=alert(1)//>` and now the `img src="cid:"` and there wont be any src called cid right, so now the onerror fires up and it gets executed, and after the alert everything comments out because of `//`

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Never pass data from `location`, `document.referrer` or `postMessage` into sinks like `eval()`, `innerHTML` or `document.write()`.
- For `postMessage`, always check `event.origin` against an exact allowlist using strict equality. Do not use `indexOf`, `startsWith` or `endsWith`.
- Validate the type and shape of incoming message data before using it.
- Use `textContent`, `setAttribute` with safe values, and Trusted Types where supported.
- A strict Content Security Policy limits the damage when something slips through.
