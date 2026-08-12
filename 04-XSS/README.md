# Project 04 — Cross-Site Scripting (XSS) Testing

## Overview

This project is part of my hands-on web application penetration testing practice.

For this assessment, I tested **OWASP Juice Shop** for Cross-Site Scripting (XSS) vulnerabilities.

The main goal was to understand how XSS works in a real web application, identify an injection point, understand how the application processes user-controlled input, determine the injection context, construct appropriate test payloads, confirm JavaScript execution, and finally demonstrate the impact of the vulnerability.

I focused on understanding the reasoning behind each step instead of only relying on automated scanners or pre-built payloads.

### Target

- **Application:** OWASP Juice Shop
- **Environment:** Local lab
- **Target URL:** `http://127.0.0.1:3000`
- **Vulnerability:** Reflected Cross-Site Scripting (XSS)

### Tools Used

- Burp Suite Community Edition
- Web Browser
- Chrome DevTools
- OWASP Juice Shop
- Kali Linux
- Burp Repeater

---

# Reconnaissance

I started the assessment by running OWASP Juice Shop locally and exploring the application normally.

While browsing the application, I looked for functionality that accepted user-controlled input and reflected that input back into the web page.

During the initial reconnaissance, I identified the product search functionality.

The search functionality accepted a user-controlled parameter:

```http
GET /rest/products/search?q=apple
```

<img width="784" height="260" alt="recon" src="https://github.com/user-attachments/assets/35fd453e-a40b-4d8b-bc9f-1b29e6f4888b" />

I intercepted the request using Burp Suite Repeater so I could modify the parameter and observe how the application responded.

The main things I wanted to determine were:
- Where user input was being accepted
- Whether the input was reflected in the response
- How the application handled HTML characters
- Whether HTML tags were interpreted
- Whether JavaScript could be executed
- Where the input was inserted into the DOM
- Which XSS context was being used

---

# Methodolgy

I used a manual black-box testing approach.

Instead of immediately using automated XSS scanners, I first tried to understand how the application processed the q parameter.

My testing process was:
```text
Discover → Intercept → Modify → Observe → Understand → Test → Validate → Document
```

The XSS testing was performed in several stages:
- Test normal search behavior
- Test input reflection
- Identify the injection point
- Test basic HTML interpretation
- Test JavaScript execution
- Identify the HTML injection context
- Validate reflected XSS
- Document the security impact

---

# XSS Testing

**1. Normal Search Request**

I first captured a normal product search request.
```text
GET /rest/products/search?q=apple
```
The application returned the expected product search results.

This provided a baseline that I could compare against when modifying the q parameter.

<img width="1915" height="841" alt="normal-search" src="https://github.com/user-attachments/assets/46df197d-6a1c-4033-925d-d723fa8b9f75" />

**2. Input Reflection Testing**

Before testing JavaScript, I first used a harmless marker:
```text
TEST123
```

The request became:
```text
GET /rest/products/search?q=Test
```

The value was reflected by the application.

I then inspected the page using Chrome DevTools.

The value appeared inside:
```http
<span id="searchValue">TEST123</span>
```
<img width="1602" height="837" alt="TEST123-search" src="https://github.com/user-attachments/assets/b794193b-0fdd-4470-a328-bd6c0090acfb" />

This was important because it showed that the search input was being placed into the HTML page.

At this point, I knew that the parameter was worth testing for HTML injection and XSS.

**3. Basic XSS Testing**

I then tested a basic JavaScript payload:
```text
<script>alert(1)</script>
```

<img width="1232" height="611" alt="alert-script-search" src="https://github.com/user-attachments/assets/fd79c067-5074-42a7-bfc9-d05e065c8017" />

The purpose of this payload was to determine whether the application would interpret the supplied input as HTML and execute JavaScript.

The payload successfully triggered a JavaScript alert in the browser.

**4. Alternative XSS Payload**

I also tested an alternative HTML-based payload:
```text
<img src=x onerror=alert(1)>
```

<img width="1916" height="842" alt="alert-check-search" src="https://github.com/user-attachments/assets/ce1e1cc6-ab86-4717-bad2-2127563ee2ed" />

The purpose of this payload was to create an HTML element and use an event handler to execute JavaScript.

The browser successfully displayed the JavaScript alert.

**5. Identifying the Injection Context**

After confirming JavaScript execution, I wanted to understand why the payload worked.

I used Chrome DevTools to inspect the resulting DOM.

The search value was located inside:
```text
<span id="searchValue">USER_INPUT</span>
```
When the XSS payload was supplied, the DOM contained:
```http
<span id="searchValue">
    <img src="x" onerror="alert(1)">
</span>
```

<img width="1599" height="812" alt="identify" src="https://github.com/user-attachments/assets/31efdaa7-a6df-4c6b-a0e0-52af1c274430" />

This showed that the input was being interpreted as HTML rather than being safely treated as plain text.

The injection context was therefore an HTML body context.

---

# Findings

XSS-01 — Reflected Cross-Site Scripting

Severity: Medium

Status: Confirmed

Affected Endpoint:
```text
GET /rest/products/search?q=
```

Affected Parameter: q

Vulnerability Type: Reflected Cross-Site Scripting (XSS)

The product search functionality was vulnerable to Reflected Cross-Site Scripting because user-controlled input was reflected into the application's HTML without being safely handled for the output context.

I was able to provide HTML containing a JavaScript event handler and successfully execute JavaScript in the browser.

The vulnerability was confirmed using:
```text
<img src=x onerror=alert(1)>
```

The successful JavaScript alert confirmed that the supplied input was interpreted as HTML and that JavaScript execution was possible.

**Proof of Concept**

The following payload was used:
```http
<img src=x onerror=alert(1)>
```

<img width="1916" height="842" alt="alert-check-search" src="https://github.com/user-attachments/assets/cf533071-a507-4237-8e4f-70015073cecf" />

---

# Impact

- Successful XSS allows attacker-controlled JavaScript to execute in a victim's browser within the vulnerable application's context.
  Modify webpage content
- Display fake or malicious content
- Perform actions through application functionality available to the victim
- Read information exposed to JavaScript within the application context
- Target authenticated users
- Potentially perform phishing attacks through modified page content

---

# Remediation

- User-controlled input should be properly encoded before being inserted into HTML.
- When HTML rendering is not required, developers should use safe DOM APIs.
- Input validation can provide an additional layer of protection.
- A strong Content Security Policy (CSP) should be implemented as an additional layer of defense.

- 
