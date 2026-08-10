## 🚨 Findings

### AUTH-01 — Weak Password Policy

**Severity:** Medium  
**Status:** Confirmed

#### Description

During the registration testing, I found that the application allowed users to create accounts with extremely weak passwords.

For example, I was able to register an account using a password such as:

`123`

The application accepted the request instead of rejecting the password because of its weak strength.

The server responded with:

`HTTP/1.1 201 Created`

This indicates that password-strength validation is either missing or not being properly enforced on the server side.

#### Proof of Concept

I intercepted the registration request using Burp Suite and submitted a weak password.

Example:

```json
{
  "email": "test11@gmail.com",
  "password": "123",
  "passwordRepeat": "123"
}
