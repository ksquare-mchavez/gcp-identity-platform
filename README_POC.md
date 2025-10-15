
# Proof of Concept: Identity Platform API Integration

## Document Purpose
This document provides:
- **Setup instructions**
- **API endpoints for key operations**
- **Sample curl commands**

The goal is to demonstrate:
1. Sign up user (email-only / password-less)
2. Sign in user (email-only / password-less, or with pre-registered credentials)
3. Receive access token (JWT) and refresh token
4. Get the certificate/public key to verify JWT signature
5. Get a new JWT access token using the refresh token

This guide is intended for iOS engineers implementing client-side logic.

---

## Key Operations & Endpoints

### 1. Sign Up User (Email-only / Password-less)

#### Send Password Reset Email
Used to send one-time `out-of-band` (OOB) codes (e.g., email verification, password reset) to users.

- With Google Identity Platform (advanced Firebase Auth)
- Firebase generates a one-time OOB action code internally and sends the link to the user's email (using default/configured template)
- The API response does **not** contain the actual OOB link (for security reasons). That’s the expected behavior. These links can directly reset a password or verify an email.

**Reference:** [Send Password Reset Email (Google Docs)](https://cloud.google.com/identity-platform/docs/use-rest-api#section-send-password-reset-email)

**Sample curl:**
```sh
curl 'https://identitytoolkit.googleapis.com/v1/accounts:sendOobCode?key=[API_KEY]' \
   -H 'Content-Type: application/json' \
   --data-binary '{"requestType":"PASSWORD_RESET","email":"[user@example.com]"}'
```

**Sample Response:**
```json
{
   "email": "[user@example.com]"
}
```

---

## Notes on HIPAA Compliance

- If your project is under Firebase Auth (Firebase Console), it is **not HIPAA compliant**.
- If it is a Google Cloud Identity Platform project with a signed BAA, it **can be HIPAA compliant**.

### Email Links (sendOobCode)
Google’s built-in email delivery system sends OOB links.

**Do NOT include PHI in:**
- Email subject or body templates
- Action URLs or query parameters

Note: These emails are not intended for PHI transport; including PHI could break HIPAA compliance.

**For HIPAA-safe email delivery:**
- Disable automatic OOB emails
- Use your own secure mail provider
- Generate and deliver the link yourself via your secure system (no PHI in the URL)
