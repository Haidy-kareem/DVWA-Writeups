## 1. Objective

Test and document the Open Redirect vulnerability in DVWA at the **High** security level, recording the full process: initial attempts, failures, source code analysis, and final exploitation.

---

## 2. Initial Attempt — Redirecting to Google

I set the DVWA security level to **High** and attempted a basic Open Redirect test by pointing the `redirect` parameter at an external site:

```
http://127.0.0.1/DVWA/vulnerabilities/open_redirect/source/high.php?redirect=https://google.com
```

**Result:** The redirect was blocked. The application did not send the browser to Google, and no redirect occurred.

This showed that, unlike the Low/Medium levels, High applies some form of input validation before performing the redirect.

---

## 3. Source Code Analysis

To understand why the request was blocked, I opened the page source via DVWA's "View Source" feature:

<img width="807" height="411" alt="image" src="https://github.com/user-attachments/assets/046681e3-0d8a-46e6-9754-dc8f8d686960" />


**Analysis:**

- The application uses PHP's `strpos()` function, which only checks whether the substring `info.php` exists **anywhere** inside the supplied `redirect` value.
- It does **not** validate the domain, the full URL structure, or whether the destination is trusted.
- As long as the string `info.php` appears somewhere in the parameter, the check passes and the `header("location: ...")` redirect is executed with the raw, attacker-controlled value.

---

## 4. First Bypass Attempt — Appending `info.php` to Google's URL

Based on the source code, I tested whether simply adding the required substring to an external URL would satisfy the check:

```
http://127.0.0.1/DVWA/vulnerabilities/open_redirect/source/high.php?redirect=https://google.com/info.php
```

**Result:** The validation passed (the string `info.php` was present), and the application executed the redirect. The browser was sent to:

```
https://google.com/info.php
```

Since that path does not exist on Google's servers, Google returned:

<img width="960" height="1057" alt="image" src="https://github.com/user-attachments/assets/8c14250d-2ee4-4eae-93bb-5af74b703cdf" />


**Observation:** This was not a failure of the exploit — it confirmed the vulnerability. The DVWA filter was successfully bypassed and the redirect was fully under my control; the 404 only reflects that the *target site* doesn't have that page, not that the Open Redirect flaw failed. This step is important evidence for the write-up because it proves the filter can be defeated using only the required keyword, regardless of the actual destination domain.

---

## 5. Building a Full Exploitation Scenario — Local Attacker Server

To demonstrate the real-world impact (redirecting a victim to a page an attacker fully controls, rather than to a 404), I set up a local server to simulate an attacker-hosted page via the terminal 

**Steps taken:**

```bash
mkdir ~/server
cd ~/server
nano info.php
```

Inside `info.php`, I added a simple HTML page:

```html
<h1>open redirect test by haidy</h1>
```

I then started PHP's built-in development server:

```bash
php -S 127.0.0.1:8000
```

**Explanation of the command:**

- `php -S` — starts PHP's built-in web server.
- `127.0.0.1` — binds the server to localhost only (simulating an external attacker-controlled host for lab purposes).
- `8000` — the port the server listens on.

The page became reachable at:

```
http://127.0.0.1:8000/info.php
```

<img width="946" height="1021" alt="image" src="https://github.com/user-attachments/assets/4fb13962-c800-43d8-87f5-f4302a645546" />


## 6. Final Exploitation

With the local "attacker" server running, I submitted the final payload:

```
http://127.0.0.1/DVWA/vulnerabilities/open_redirect/source/high.php?redirect=http://127.0.0.1:8000/info.php
```

**Result:** The validation check passed because the string `info.php` was present in the URL. The application executed:

```php
header("location: " . $_GET['redirect']);
```

The browser was successfully redirected to the attacker-controlled page:

```
http://127.0.0.1:8000/info.php
```

This confirms full exploitation of the Open Redirect vulnerability: a trusted DVWA URL was used to send the browser to an arbitrary, attacker-chosen destination.

<img width="952" height="970" alt="image" src="https://github.com/user-attachments/assets/10b0853f-d244-4733-a553-d1ae5661f5bf" />


<img width="1254" height="1052" alt="image" src="https://github.com/user-attachments/assets/a6cc0044-d04e-4375-8018-f6778ede86a9" />


## 7. Impact

An attacker could:

- Host a malicious page (e.g., a phishing login form) at a URL containing the string `info.php`.
- Send victims a link that starts with the trusted DVWA domain, increasing the likelihood they will click it.
- Silently redirect victims to the malicious page once the link is clicked, potentially harvesting credentials or delivering malware.

Because the initial link visibly belongs to a trusted domain, victims are far more likely to trust and click it than a direct link to an unknown site — the core danger of Open Redirect flaws in phishing campaigns.

---

## 8. Root Cause

The vulnerability stems from validating user input using **substring matching** (`strpos()`) instead of validating the **complete destination URL**. Checking only for the presence of `info.php` anywhere in the value allows any attacker-controlled URL to pass the filter, as long as that substring is present.

---
