# Business Logic Vulnerabilities

> PortSwigger Web Security Academy · **10/12** labs completed

Business logic flaws come from wrong assumptions about how users will behave. Tools do not find them easily, because each request can look valid on its own.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Excessive trust in client-side controls](#1-excessive-trust-in-client-side-controls)
- [2. High-level logic vulnerability](#2-high-level-logic-vulnerability)
- [3. Inconsistent security controls](#3-inconsistent-security-controls)
- [4. Flawed enforcement of business rules](#4-flawed-enforcement-of-business-rules)
- [5. Low-level logic flaw](#5-low-level-logic-flaw)
- [6. Inconsistent handling of exceptional input](#6-inconsistent-handling-of-exceptional-input)
- [7. Weak isolation on dual-use endpoint](#7-weak-isolation-on-dual-use-endpoint)
- [8. Insufficient workflow validation](#8-insufficient-workflow-validation)
- [9. Authentication bypass via flawed state machine](#9-authentication-bypass-via-flawed-state-machine)
- [10. Infinite money logic flaw](#10-infinite-money-logic-flaw)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Map the full intended workflow of a feature (e.g. checkout: add to cart → apply coupon → pay → confirm) and ask what the server actually verifies at each step versus what it assumes.
- Test skipping steps entirely — can step 4 be called without having completed steps 1–3?
- Test repeating a step that should only be allowed once (reusing a one-time coupon, re-submitting a 'first purchase' discount, replaying a completed action).
- Test values the UI doesn't let you set but the API might accept: negative quantities/prices, prices for a different currency/region, exceeding a stated maximum.
- Test ordering tricks: changing an item, address, or amount after a validation step but before final confirmation (TOCTOU within the workflow itself).
- Test whether a provided value is trusted twice inconsistently — e.g. price is validated against the catalog at one step but taken from the client directly at another (like the final payment confirmation).
- Look for ways to abuse a feature exactly as designed but toward an unintended goal (e.g. gift cards, referral bonuses, export limits) — these bugs are about intent, not malformed input.

## Lab write-ups

### 1. Excessive trust in client-side controls

**Lab:** [Excessive trust in client-side controls](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-excessive-trust-in-client-side-controls)

- In this lab our goal is to buy a product which is very expensive and we don't have enough money to buy it
- So we will add the item to the cart and note that there is a post request which has the itemname and price
- Change the price value to 0 and send the request and the request is being accepted
- Now add the item to the cart and buy it

---

### 2. High-level logic vulnerability

**Lab:** [High-level logic vulnerability](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-high-level)

- In this lab there is no price parameter here so we will see that we can do anything about the quantity parameter
- I added 2 items to the cart and when I put -3 as the parameter the cart is going to -1 and the price is in negative. we wont be able to buy in negative amount
- So we need to have positive amount
- We will add the leather jacket and add another product
- Now we have to adjust the cart so that the amount comes within $100. to do that we have to put -something to the quantity so that the price comes down to 100 and the leather jacket cart count should be 1
- If we bring it under the credits we have and click buy we will be able to buy it

---

### 3. Inconsistent security controls

**Lab:** [Inconsistent security controls](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-inconsistent-security-controls)

- In this lab we have a email client and first we need to register a account with our mail id
- So go to email client to get the mail id and register with that mail id
- The mail id is `attacker%40YOUR-EXPLOIT-SERVER-ID.exploit-server.net`
- Now we got a email for mfa and we registered the account
- This lab states that the email which has `dontwannacry.com` domain has the admin access and we do have a email update column
- Change the mail to `attacker@dontwannacry.com`  and now you got the admin access
- Delete the carlos account

---

### 4. Flawed enforcement of business rules

**Lab:** [Flawed enforcement of business rules](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-flawed-enforcement-of-business-rules)

- In this lab there is a option to signup for newsletter and when we do that we get a ne coupon code which has 30%off
- Now add the jacket and go to the cart to apply both the coupons
- When you try to apply the same coupon twice you will get an error but when you alternate it, you will be able to apply infinite times
- Bring the total to 0 and purchase the product

---

### 5. Low-level logic flaw

**Lab:** [Low-level logic flaw](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-low-level)

- First we will add 1 jacket and see if there is any possible parameter to attack
- Now we add 100 jackets and we get the error invalid quantity and when we add 99 it accepts it
- Now we will add very large number of jackets and see is there any unexpected behaviour
- When the price value goes around $2B, it changed to negative value and after that for every entry the price is reversing like from negative to zero
- So we add more of jackets so that we try to bring the value back to a positive number
- Now the comes close to zero but not between 100 credits so we add some other product to match the value and the total of jackets are 32000 and the other product is 44
- Now click buy to finish the lab

---

### 6. Inconsistent handling of exceptional input

**Lab:** [Inconsistent handling of exceptional input](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-inconsistent-handling-of-exceptional-input)

- In this lab there is no minimum number of characters in email column for registering an account
- We have the email client which gives us all the emails from its subdomain too
- So we wil try to register a email, with 100 200 300 400 and 500 characters before @ so that we can see that whether there is a limit of not
- To do that send the request to the intruder, mark the word attacker in email(because the email given to us is `attacker@exploit.something` .
- Under the payload option, select character blocks, set the minimum length to 100 and max to 500 ad step up to 100 so there will be 5 entries with 100 200 300 400 and 500 characters
- nw we got the verification mail and the click the first mail to verify it and when you login into the account, there is no full email that we passed, it means that the value is getting truncated after a certain number of characters the the character count is 238, any letter after 238 characters is getting removed
- So our goal is to create a email with @dontwannacry.com domain within 238 characters and make our email client domain to come after 238 characters
- So to do that add 221 A's and then add @dontwannacry.com so that it comes to 238 characters, after that we add our email domain
- So the email will be `AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA@dontwannacry.com.YOUR-EXPLOIT-SERVER-ID.exploit-server.net`
- Now we got the verification email and also when we login to our account the email is just with the @justwannacry.com with the A's
- Now delete carlos account to finish the lab

---

### 7. Weak isolation on dual-use endpoint

**Lab:** [Weak isolation on dual-use endpoint](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-weak-isolation-on-dual-use-endpoint)

- In this lab there is a update password functionality
- So take the endpoint and change the username to administrator
- Now see that it is not a valid request
- Now remove the current password and send the request with your new password and username eas administrator and you have changed the password
- Now login and delete carlos account

---

### 8. Insufficient workflow validation

**Lab:** [Insufficient workflow validation](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-insufficient-workflow-validation)

- In this lab there is a vuln in workflow events validation
- To exploit this vuln we need to place an order under our limits and see what is the logic behind it
- There are 3 endpoints, the first one is `/cart` which adds the items to the cart, second one is `/cart/checkout` which is a post request to buy the order and the third one is `/order-confirmation?order-confirmed=true` which is a GET request
- The potential vuln endpoint could be not checking the status of payment and setting the product to be bought so the third endpoint may a vuln one
- So to exploit it add the leather jacket and click buy
- Now copy the third endpoint and replace the `/cart/checkout` and send the request and you get the 405 error and get is allowed
- Now change the method to get and you successfully purchased the product

---

### 9. Authentication bypass via flawed state machine

**Lab:** [Authentication bypass via flawed state machine](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-authentication-bypass-via-flawed-state-machine)

- In this lab, the vuln exists in authentication workflow sequence event
- First we will login as wiener and it asked for the role, there are 2 roles- user and contentcreator
- So we will try to visit /admin and it says only admins can view it
- Now we logout and try to login again
- Turn on intercept and login
- Pass the /login POST request but drop the role selector
- Remove /role-selector from the URL and refresh the page and you got the admin panel
- The page is checking whether this is a login but not the role so we got the admin access
- Delete the account to finish the lab

---

### 10. Infinite money logic flaw

**Lab:** [Infinite money logic flaw](https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-infinite-money)

- In this lab the vuln exists in gift card
- There is a product called gift card and we have signup section to get 30% discount
- The gift card price is 10 and with discount it is 7. when we buy the giftcard, it is giving us the code and when we redeem it we are getting 10 we can do this infinite times
- So we are spending 7 to get 10 so push the credits to 1337 to get the jacket
- After getting 1337 buy the jacket to finish the lab

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Enforce every business rule on the server: prices, quantities, roles, limits and workflow order.
- Validate each step of a multi-step process, and do not assume users follow the intended order.
- Do not trust values sent by the client for anything that affects money or privilege.
- Test unusual inputs: negative numbers, very large numbers, repeated actions and skipped steps.
- Include abuse cases in design reviews and threat modeling.
