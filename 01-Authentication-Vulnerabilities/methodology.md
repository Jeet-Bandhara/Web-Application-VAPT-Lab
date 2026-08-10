##  Methodology

After identifying the authentication-related functionality, I started testing each area individually.

### 1. Registration Testing

I first tested the registration functionality to understand how the application handled passwords.

I tried using weak and commonly used passwords to check whether the application had proper password-strength requirements.

The application accepted a very weak password and successfully created the account.

This became the main confirmed finding from this project.

### 2. Login Testing

I then looked at the login process and captured the authentication requests using Burp Suite.

I reviewed the requests and responses to understand how authentication was being handled and what information was returned after successful authentication.

### 3. JWT Testing

The application uses JWT tokens for authentication, so I inspected the JWT structure and decoded the header and payload.

I tested whether changing values inside the payload, including the user's role, could be used to obtain unauthorized privileges.

Although the modified token was accepted at the HTTP level in some requests, it did not result in successful privilege escalation. The application returned an empty user object instead.

Because I could not demonstrate actual unauthorized access, I did not report JWT manipulation as a confirmed vulnerability.

### 4. Password Reset Testing

I also tested the forgot-password and password-reset functionality.

I inspected the requests related to security questions and password changes and replayed modified requests through Burp Suite.

The invalid reset attempts returned:

`HTTP 401 Unauthorized`

Since I could not bypass the authentication or reset another user's password, I did not classify this as a confirmed vulnerability.

### Testing Approach

I followed a simple approach throughout the assessment:

**Discover → Intercept → Modify → Replay → Verify → Document**

I only considered an issue a confirmed vulnerability when I could reproduce it and demonstrate a meaningful security impact.
