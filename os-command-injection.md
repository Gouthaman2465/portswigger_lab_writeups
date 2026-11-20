# OS Command Injection

> PortSwigger Web Security Academy · **3/5** labs completed

OS command injection lets an attacker run operating system commands on the server by abusing an application that passes user input to a shell.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. OS command injection, simple case](#1-os-command-injection-simple-case)
- [2. Blind OS command injection with time delays](#2-blind-os-command-injection-with-time-delays)
- [3. Blind OS command injection with output redirection](#3-blind-os-command-injection-with-output-redirection)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Find input that is plausibly passed to a system command: file conversion tools, ping/traceroute/diagnostic utilities, image processing, PDF generation, export features.
- Inject shell metacharacters one at a time and watch for behavior changes: `;`, `|`, `&&`, `||`, backticks, `$()`, newline.
- Try a simple visible side effect first (`; sleep 10` for timing, or `| whoami` if output is reflected) before attempting real exploitation.
- Check if output is returned in the response (visible) or not (blind) — blind needs time delays or out-of-band callbacks (DNS/HTTP) to confirm.
- If some characters are blocked, test alternate separators/spacing (`%0a`, `${IFS}` instead of a space, or encoding) to bypass a weak filter.
- Check the OS: Windows vs Linux changes available commands and metacharacter behavior.

## Lab write-ups

### 1. OS command injection, simple case

**Lab:** [OS command injection, simple case](https://portswigger.net/web-security/os-command-injection/lab-simple)

- This lab is vuln to os command injection in stock check endpoint
- There are 2 variables in stock check so add |whoami in storeID so you will  be able to get the username

---

### 2. Blind OS command injection with time delays

**Lab:** [Blind OS command injection with time delays](https://portswigger.net/web-security/os-command-injection/lab-blind-time-delays)

- In this lab the vuln exists in the feedback page where the vuln is blind one so we need to make sure we are manipulating the response
- To do that go to feedback section and submit a feedback
- Send the feedback to the repeater and add this payload in email bar- ``& ping -c 10 127.0.0.1 &``
- We need to url encode it to put it in the values and send the request
- Final line `csrf=40HmWko6MEGaUnhbh5039ee3SS1odP2A&name=hel&email=g%40g.com%26%20ping%20-c%2010%20127.0.0.1%20%26&subject=gweg&message=wgrgw`
- Now we get the response exactly after 10 seconds confirming the vuln exists and we do control the response time

---

### 3. Blind OS command injection with output redirection

**Lab:** [Blind OS command injection with output redirection](https://portswigger.net/web-security/os-command-injection/lab-blind-output-redirection)

- First we need to analyze whether the endpoint is vuln to blind os command injection
- To do that submit a feedback and then modify the vuln variable which is email to sleep 10 # so that we can confirm that whether it is actually taking 10 seconds to sleep
- This is the payload we need to put in email - `&sleep 10 #` to url encoded `+%26sleep+10+%23`
- Now we confirmed that this is vuln to blind command injection
- Now we will try to put the results of whoami in a .txt file in /var/www/images/output.txt
- The payload is `& whoami > /var/www/images/output.txt #`. url encoded is `%26+whoami+>+/var/www/images/output.txt+%23`
- The full payload is `csrf=0DsZlhff9poT9OxsOfOlafcOTIGN368f&name=hello&email=gg%40g.com%26+whoami+>+/var/www/images/output.txt+%23&subject=hellllll&message=wegw%0A%5C%0A`
- When we send the request it is giving us `500 internal error` means it is not able to save it. it throws 500 error because there are 2 important things that is happening, executing the command and saving the feedback response
- So when the application runs the command it is not facing any issue but when saving the feedback it sees that something is not right so it throws an error
- Now we should confirm that the output.txt is there are not. to do it load the page and you will see that the images of product pages are seen in the http history page. change the 15.jpg to output.txt to get the username

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Avoid calling the shell. Use language APIs for the task (for example, a library instead of `ping` or `convert`).
- If a command is needed, call it with an argument array and no shell, so metacharacters have no special meaning.
- Validate input with a strict allowlist (for example, digits only for an ID).
- Run the application with least privilege and, where possible, in a sandbox or container.
- Escaping by hand is fragile. Do not make it your only defense.
