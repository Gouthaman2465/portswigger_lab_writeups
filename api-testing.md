# API Testing

> PortSwigger Web Security Academy · **5/5** labs completed

API testing covers finding hidden endpoints, abusing HTTP methods, mass assignment and server-side parameter pollution.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [1. Exploiting an API endpoint using documentation](#1-exploiting-an-api-endpoint-using-documentation)
- [2. Exploiting server-side parameter pollution in a query string](#2-exploiting-server-side-parameter-pollution-in-a-query-string)
- [3. Finding and exploiting an unused API endpoint](#3-finding-and-exploiting-an-unused-api-endpoint)
- [4. Exploiting a mass assignment vulnerability](#4-exploiting-a-mass-assignment-vulnerability)
- [5. Exploiting server-side parameter pollution in a REST URL](#5-exploiting-server-side-parameter-pollution-in-a-rest-url)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Enumerate endpoints beyond what the UI uses: check JS files, API docs/schema (Swagger/OpenAPI), versioned paths (`/v1`, `/v2`, `/internal`), and guess plural/singular or near-miss paths.
- For every endpoint, test every HTTP method, not just the one the UI uses — a `DELETE` or `PUT` might lack the checks the `GET` has.
- Test mass assignment: send extra fields in a request body (`role`, `isAdmin`, `price`) that the UI doesn't expose, and see if the server binds them anyway.
- Test server-side parameter pollution: does a value from the user end up unsafely concatenated into an internal API call the app makes on your behalf?
- Compare behavior between API versions — an old/internal version may lack a fix or check present in the current one.
- Check content-type handling: does the endpoint behave differently (or skip validation) for XML, form data, or alternate encodings versus the expected JSON?

## Lab write-ups

### 1. Exploiting an API endpoint using documentation

**Lab:** [Exploiting an API endpoint using documentation](https://portswigger.net/web-security/api-testing/lab-exploiting-api-endpoint-using-documentation)

- Login to `wiener:peter` and search for the API endpoint
- We found out that the endpoint is `/api`
- Now click delete and type carlos and you have deleted it

---

### 2. Exploiting server-side parameter pollution in a query string

**Lab:** [Exploiting server-side parameter pollution in a query string](https://portswigger.net/web-security/api-testing/server-side-parameter-pollution/lab-exploiting-server-side-parameter-pollution-in-query-string)

- In this lab we have to login as admin and delete carlos and we don't have any login like `wiener:peter`
- So now we only have the forgot password section and the vuln should be there
- Now give the username as administrator and you can see that the request in sent to the endpoint `/forgot-password` and you have forgot-password.js file
- Look at the file that there is a parameter called `reset-token`
- Now coming back the username, change it to some random username and it gives invalid username means the backend is actually checking for the existing username
- Now lets add a parameter to the username as a encoded url like `username=administrator%26abc=123` `%26 =& in url encoding ` here abc is a parameter and 123 is the value
- Now we got a different error called parameter not supported and this means that the frontend doesn't just send the username without sanitizing it, if it doesn't sanitize it it would have given the same invalid username error so the frontend is properly sanitizing the user input and it tells that abc is not a valid field
- Now we use # in the url which removes all the things after the # means if the actual url is username=admin&field=email and you are adding the # after admin, the url omits the field=email part
- So the command will be `username=administrator%26abc=123%23` `#=%23 in url encoding`
- Now it throws an error called field not specified which means that the backend relies on a parameter called `field` and this contains some value
- Now lets add some values to field like `field=x%23` and now we got the error called invalid field and this time also we have added # at the last but this time it doesn't throw some field is not specified so `username` and `field` are the 2 parameters here
- So now we have to brute force the field parameter's value
- Send the request to intruder and highlight `x` and in payload list select server side variables names but it is not available in basic burp version
- So get it by googling and paste the list and run it. the status code which gets 200 are the valid fields and the valid fields are email and username
- Now we got a field called reset_token from forgot-password.js now give that field in field option like this `username=administrator%26field=reset_token%23` now we got the reset token in the response
- Copy the token and go to `/forgot-password?reset_token=<token>` and now you got the screen to add the new password
- Now you've changed it and delete carlos account

---

### 3. Finding and exploiting an unused API endpoint

**Lab:** [Finding and exploiting an unused API endpoint](https://portswigger.net/web-security/api-testing/lab-exploiting-unused-api-endpoint)

- In this lab we have the buy a product without money so we need to change the price of te product to 0 so that we can buy it
- To do this login to your account and add the product to the cart
- Note that there has a api endpoint which is `api/products/1/price` which has the price of the product
- Now send that to the repeater and change the method of the http request and see what happens
- When we use POST method it states that this is not allowed only GET and PATCH allowed
- So we used patch now it shows only application/json allowed
- Now we created a json body and added its content type and length as follows

```
Content-Type: application/json
  Content-Length: 12

{"price": 0}
```

- Now we have changed the price and now we can buy it

---

### 4. Exploiting a mass assignment vulnerability

**Lab:** [Exploiting a mass assignment vulnerability](https://portswigger.net/web-security/api-testing/lab-exploiting-mass-assignment-vulnerability)

- This is same like the above lab but we have to find the hidden field
- We got a API endpoint which is `api/checkout`
- The request of that endpoint contains product id and product name but the response has a separate field called chosen discount and it has a field called percentage

```
"chosen_discount":{"percentage":0},
```

- So to exploit it intercept the request and add the above field in the json body, make the percentage to 100 and pass it . now your product has become 0 and you can buy it

---

### 5. Exploiting server-side parameter pollution in a REST URL

**Lab:** [Exploiting server-side parameter pollution in a REST URL](https://portswigger.net/web-security/api-testing/server-side-parameter-pollution/lab-exploiting-server-side-parameter-pollution-in-rest-url)

- In this lab we have to login as admin and delete carlos using the api endpoint which has a vuln in path traversal and parameter pollution in url
- Here we don't have any login credentials to login so the vuln probably lies in forgot password
- Send the username as `administrator` and send the /forgot-password request to repeater and we also has a js file called forgot-password.js
- The first thing we are going to see is whether this is a query string or path url
- To do this we are going to pass 2 characters `#=%23 and ?=%3F` after administrator
- In query string these values are treated as a normal character so these char wont affect it
- But in path url these values means something
- ### Query string vs path URL:
	- Query string url takes the value in key:value pairs
	- `example.com/products?id=5&color=blue` is an example, here id is taken as 5 and color is taken as blue
	- In path url the value is taken by traversing to a specific parameter and that parameter has the value of it
	- `example.com/products/5` here the parameter is product and the value of it is 5 so to do it, it should traverse
- So when you pass # to path, it takes as a terminator value meaning anything that comes after it is omitted
- When you send ?, it takes it as something new is going to come so it has to take it as a separate one
- If you give both of the char and you get nay error other than invalid username, it is confirmed that it is a path url because those char are taken as a normal char it query string so `administrator` becomes `administrator#` and it will check for that user and return invalid username
- Now we confirmed that it is a path url because we got the error ` "Invalid route. Please refer to the API definition"`
- To find the depth of the  endpoint so now add `../` before  `%23` to check what error we face
- We faced a different error when we passed `../../../../%23`. it says not found that means we have navigated outside the root directory
- Now add some common api endpoints at the end of the `../`  and when we tried `openapi.json` and we got a new error

```
{
  "error": "Unexpected response from API server:\n{\n  \"openapi\": \"3.0.0\",\n  \"info\": {\n    \"title\": \"User API\",\n    \"version\": \"2.0.0\"\n  },\n  \"paths\": {\n    \"/api/internal/v1/users/{username}/field/{field}\": {\n      \"get\": {\n        \"tags\": [\n          \"users\"\n        ],\n        \"summary\": \"Find user by username\",\n        \"description\": \"API Version 1\",\n        \"parameters\": [\n          {\n            \"name\": \"username\",\n            \"in\": \"path\",\n            \"description\": \"Username\",\n            \"required\": true,\n            \"schema\": {\n        ..."
}
```

- From the above error we can see that the api endpoint is `/api/internal/v1/users/{username}/field/{field}`
- Now we have to find a valid field which is after the username
- When we send some random value it throws 400 error but email is a valid one because we know that the email is sent the email and when I give it, it returns the email with ****
- From forgot-password.js we can take a field that is used for password rest which is `passwordResetToken`
- Give the field as `passwordResetToken` and now we got a new error which states that `This version of API only supports the email field for security reasons`
- It might have used a version like v2 so now we are going to switch it to v1
- To do this we write this path `../../v1/users/administrator/field/passwordResetToken%23`
- Why are we writing exactly like this and why cant we write the full path like api/internal/....
- When we write `api/internal/v1/users/administrator/field/passwordResetToken%23` the server takes this whole string as username because the username is going to fit into the placeholder of it. the actual path is `/api/internal/v1/users/{username}/field/{field}` where the above given path will go into the place of {username} and now the actual path that goes to the server becomes `/api/internal/v1/users/{api/internal/v1/users/administrator/field/passwordResetToken%23}' which makes no sense`
- But when you give `../../v1/users/administrator/field/passwordResetToken%23` this goes into like `/api/internal/v1/users/{../../v1/users/administrator/field/passwordResetToken%23}` this one reads like this api->internal->v1->users-> now `../` goes backward so the pointer goes to v1 and another ../ moves it to internal. now the remaining part of te given input adds up so the final input will become `api/internal/v1/users/administrator/field/passwordResetToken%23`
- Now we got the token to reset the password
- Now edit the url in the browser to `/forgot-password?passwordResetToken=51oijwvdveea5ybbk0ruc4ihizm6p0ys`
- Now you got the new password page now add the password, login as admin and delete carlos

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Keep an inventory of all endpoints, versions and methods. Remove unused ones and unused documentation.
- Allow only the HTTP methods and content types each endpoint needs.
- Prevent mass assignment: bind only an allowlist of fields from the request to internal objects.
- Encode user input before putting it into internal API calls to avoid parameter pollution.
- Apply authentication and authorization to every endpoint, including internal and legacy ones.
