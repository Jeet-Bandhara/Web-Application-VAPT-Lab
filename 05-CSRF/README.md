# Project 05 — Cross-Site Request Forgery (CSRF) Testing

## Overview

This project is part of my hands-on Web Application Penetration Testing practice.

For this assessment, I tested **OWASP Juice Shop** for Cross-Site Request Forgery (CSRF) vulnerabilities.

The main goal was to understand how CSRF works in a real web application, identify a state-changing endpoint, determine whether CSRF protections were implemented, test whether cross-origin requests were accepted, and finally demonstrate the impact using a working CSRF Proof of Concept.

I focused on understanding the complete attack flow instead of only relying on automated scanners.

### Target

- **Application:** OWASP Juice Shop
- **Environment:** Local Lab
- **Target URL:** `http://127.0.0.1:3000`
- **Vulnerability:** Cross-Site Request Forgery (CSRF)
- **Affected Endpoint:** `POST /profile`
- **Affected Function:** Username modification

### Tools Used

- Burp Suite Community Edition
- Burp Proxy
- Burp HTTP History
- Burp Repeater
- Google Chrome
- Kali Linux
- Python HTTP Server
- OWASP Juice Shop


---

# Reconnaissance

I started the assessment by running OWASP Juice Shop locally and exploring the application normally.

The objective during reconnaissance was to identify functionality that could modify user information or perform other state-changing actions.

While browsing the application, I identified the **User Profile** functionality.

The profile page allowed an authenticated user to modify their username.

The relevant endpoint was:

```http
POST /profile
```

<img width="1257" height="616" alt="recon" src="https://github.com/user-attachments/assets/360749f6-71c3-44f4-a18b-a9f79876528b" />

The request used:
```text
Content-Type: application/x-www-form-urlencoded
```

and the request body contained the username:
```text
username=tester
```

I captured the request using Burp Suite.

The endpoint became interesting from a CSRF perspective because it performed a state-changing operation and accepted a simple form-encoded POST request.

<image>

---

# Methodology

I used a manual black-box testing methodology.
```text
Reconnaissance
      ↓
Identify State-Changing Functionality
      ↓
Capture Request
      ↓
Analyze Authentication
      ↓
Check CSRF Token
      ↓
Test Origin Validation
      ↓
Test Cross-Origin Request
      ↓
Create CSRF PoC
      ↓
Validate Unauthorized State Change
      ↓
Document Impact
```

---

# CSRF Testing

## **1. Identify a State-Changing Endpoint**

During normal application usage, I identified the profile username modification functionality.

The application sends a request similar to:
```text
POST /profile HTTP/1.1
Host: 127.0.0.1:3000
Content-Type: application/x-www-form-urlencoded

username=tester
```

The endpoint modifies the authenticated user's profile.

This makes it a potential CSRF target because an attacker may attempt to cause a victim's browser to submit the same request without the victim intentionally performing the action.

<img width="1219" height="609" alt="testing-csrf" src="https://github.com/user-attachments/assets/ba157d48-ab20-43fa-9ae4-783a17b37a4b" />


## **2. Analyze the Request**

I examined the request headers and body in Burp Suite.

The request contained:
```text
POST /profile
```

The request also contained the application's authentication cookie

No dedicated CSRF token was visible in the request.

This was an important observation because CSRF tokens are commonly used to ensure that state-changing requests originated from the legitimate application.

However, the absence of a visible CSRF token alone does not prove a vulnerability.

I therefore continued with active testing.

## **3. Test Origin Validation**

The next step was to determine whether the application validated the Origin header.

The legitimate request normally contained:
```text
Origin: http://127.0.0.1:3000
```

I modified the request to use an attacker-controlled origin:
```text
Origin: http://test.example
```

The rest of the request was kept unchanged.

The server still responded with:
```text
HTTP/1.1 302 Found
Location: /profile
```

<img width="1227" height="579" alt="origin-change" src="https://github.com/user-attachments/assets/ebb80c60-7cc7-49bb-92b7-92c9ae9203f3" />

## **4. Verify the State Change**

The response alone was not enough to prove CSRF.

I therefore checked whether the profile was actually modified.

After sending the request with the attacker-controlled origin, the profile page showed:
```text
Username: tester3
```

<img width="1027" height="655" alt="change-username-tester3" src="https://github.com/user-attachments/assets/4d04a228-e155-485e-bfce-8c58952997be" />


The username had successfully changed.

This demonstrated that the server accepted the state-changing request even though the request contained an unexpected origin.

## **Proof of Concept**

After confirming that the endpoint accepted the cross-origin request, I created a simple HTML page containing an auto-submitted form.

```http
<!DOCTYPE html>
<html>
<head>
    <title>CSRF Test</title>
</head>

<body>

<h2>CSRF Test</h2>

<form action="http://127.0.0.1:3000/profile" method="POST">
    <input type="hidden" name="username" value="csrf-test">
    <input type="submit" value="Change Username">
</form>

</body>
</html>
```

The form sends a POST request to:
```text
POST /profile
```

The important point is that the request is generated by a page hosted on a different origin.

## **Hosting the CSRF PoC**

I hosted the malicious HTML page using Python's built-in HTTP server.

Command:
```text
python3 -m http.server 8000
```

The PoC was then accessible at:
```text
http://127.0.0.1:8000/csrf.html
```

These use different ports and therefore represent different origins.
```text
Attacker Origin
http://127.0.0.1:8000
        |
        | Cross-Origin POST
        ↓
Victim Application
http://127.0.0.1:3000
```

## **Executing the CSRF Attack**

I kept the victim user authenticated to OWASP Juice Shop.

I then opened the attacker-controlled page:
```text
http://127.0.0.1:8000/csrf.html
```

I clicked:
```text
Change Username
```

<img width="1915" height="795" alt="change-username-form" src="https://github.com/user-attachments/assets/1c49c049-2e89-4f06-897d-371d5dfb2f68" />

The browser generated the cross-origin POST request.

## **CSRF Request Captured in Burp**

Burp Suite captured the request generated by the malicious page.

The request was similar to:
```text
POST /profile HTTP/1.1
Host: 127.0.0.1:3000
Content-Type: application/x-www-form-urlencoded
Origin: http://127.0.0.1:8000

username=csrf-test
```

The important observation was that the request originated from:
```text
http://127.0.0.1:8000
```

while targeting:
```text
http://127.0.0.1:3000
```

<img width="1225" height="579" alt="burp-check" src="https://github.com/user-attachments/assets/19758934-7cfd-4332-92b5-ab2c754731ac" />

## **Final Verification**

After submitting the CSRF PoC, I returned to the Juice Shop profile page.

The username had changed to:
```text
csrf-test
```
<img width="1909" height="804" alt="verification" src="https://github.com/user-attachments/assets/bf2db380-e91c-4952-b628-2fecba7575bd" />

This confirmed that the malicious cross-origin request successfully modified the authenticated user's profile.

---

# Findings

CSRF-01 — Cross-Site Request Forgery

Severity: Medium

Status: Confirmed

Affected Endpoint:
```text
POST /profile
```
Affected Parameter: username

Affected Functionality: User Profile / Username Modification

---

# Impact

CSRF does not normally allow an attacker to directly read arbitrary responses from the vulnerable application.

Instead, the primary impact is that an attacker can cause an authenticated victim to perform an action without their intention.

The overall impact could become more serious if similar CSRF weaknesses existed in other sensitive state-changing functionality such as:
- Password changes
- Email address changes
- Security-question changes
- Two-factor authentication settings
- Account recovery settings
- Payment-related actions
- Other account-management operations

--- 

# Remediation

- Implement CSRF Tokens
- Validate the Origin Header
- Validate the Referer Header
- Use SameSite Cookies
- Avoid State-Changing GET Requests

---

