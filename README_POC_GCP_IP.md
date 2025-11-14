

# Proof of Concept: GCP Identity Platform API Integration

## Document Purpose
This document provides:
  - **Setup instructions**
  - **API endpoints for key operations**
  - **Sample curl commands**

**Goals:**
  1. Sign up user (email-only / password-less)
  2. Sign in user (email-only / password-less, or with pre-registered credentials)
  3. Receive access token (JWT) and refresh token
  4. Get the certificate/public key to verify JWT signature
  5. Get a new JWT access token using the refresh token

This guide is intended for iOS engineers implementing client-side logic.


**Expected Flow:**

<img src="diagrams/passwordless-flow.png" alt="passwordles-flow" width="50%" />

---

## Setup Instructions GCP Identity Platform
You’ll need:
  - A GCP project with Identity Platform enabled.
  - A configured email/password provider with `Allow passwordless` enabled (Identity Platform → Providers → Add a provider).
  - An API key(Identity Platform → Providers → Application setup details).

You don’t need a service account for any of this.

### Using the [REST API](https://cloud.google.com/identity-platform/docs/use-rest-api#before_you_begin)
To use the REST API, you'll need an Identity Platform API key. To obtain a key:

1. Go to the **Identity Providers page** in the [Google Cloud console](https://console.cloud.google.com/customer-identity/providers).
2. On the left menu, click Providers.
3. Click **Application setup details**.
4. Copy the `apiKey` field.

## Key Operations & Endpoints

### 1. Sign up user (email-only / password-less)
#### 1.1. Sign up user (email-only / password-less)
- GCP Identity Platform does **not** support signing up users with email only.

---

#### 1.2. Sign up with email / password
You can create a new email and password user by issuing an HTTP POST request

**Reference** [Sign up with email / password](https://cloud.google.com/identity-platform/docs/use-rest-api#section-create-email-password)

Sample request
```sh
curl 'https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=AIzaSyC90ZauZKIOUltCNFpl_R5ThCPm90FPkIU' \
-H 'Content-Type: application/json' \
--data-binary '{"email":"user@example.com","password":"password123","returnSecureToken":true}'
```

Sample response
```json
{
  "kind": "identitytoolkit#SignupNewUserResponse",
  "idToken": "[ID_TOKEN]",
  "email": "[user@example.com]",
  "refreshToken": "[REFRESH_TOKEN]",
  "expiresIn": "3600",
  "localId": "tRcfmLH7..."
}
```


### 2. Sign in user (email-only / password-less, or with pre-registered credentials)
To sign in users by email link, you must first enable the` Email/Password provider` and `Allow passwordless login` method for your GCP IP project:
1. Go to the **Identity Providers page** in the [Google Cloud console](https://console.cloud.google.com/customer-identity/providers).
2. On the left menu, click `Providers`.
3. In the `Select a provider` list, Add and enable a new provider(`Email/Password`). 
4. In the same section, enable `Allow passwordless login` (Sign-in occurs via confirmation email link).
5. Click Save.

Reference: [Sign in a user with an email by using Identity Platform](https://docs.cloud.google.com/identity-platform/docs/sign-in-user-email)

#### 2.1. Send the email sign-in link
You can sign in a user with an email by issuing an HTTP POST request.
Make sure your domain is allowlisted by your `Project settings`.

**Reference** [Send email](https://cloud.google.com/identity-platform/docs/use-rest-api#section-send-email-verification)

Sample curl:

```sh
curl -X POST "https://identitytoolkit.googleapis.com/v1/accounts:sendOobCode?key=AIzaSyC90ZauZKIOUltCNFpl_R5ThCPm90FPkIU" \
  -H "Content-Type: application/json" \
  -d '{"requestType": "EMAIL_SIGNIN","email": "user@example.com","continueUrl": "https://example.com/finishSignIn"}'
```

Response (HTTP 200 OK):
```json
{
  "kind": "identitytoolkit#GetOobConfirmationCodeResponse",
  "email": "user@example.com"
}
```

If GCP successfully queued the email to send. The user will soon receive an email with a link that looks like:
```sh
https://yourapp.com/finishSignIn?apiKey=AIzaSy...&oobCode=ABC123XYZ&mode=signIn
```

<img src="img/sign-in.png" alt="sign-in" width="50%" />

**Note:**
When you use `continueUrl` in GCP, you’re not performing a sign-in operation directly. You’re passing along a URL to which GCP will redirect the user after a particular action (like email verification, password reset, or a sign-in link in this case).

The email is not automatically included in that continue URL because:

- Security & privacy: GCP intentionally omits the user’s email from the URL to avoid leaking it through browser history, logs, analytics, or referral headers.

- Stateless flow: The continue URL is meant to be a neutral redirect destination. GCP doesn’t embed identifying user info there unless you explicitly choose to.

#### Signing in with the email link from the previous step.
We should get the oobCode from the email link sent in Step 2.1 by using the continueUrl endpoint.

Sample curl:
```sh
curl -X POST "https://identitytoolkit.googleapis.com/v1/accounts:signInWithEmailLink?key=AIzaSyC90ZauZKIOUltCNFpl_R5ThCPm90FPkIU" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com","oobCode": "XYZ123ABC"}'
```

Successful Response (HTTP 200 OK):
```json
{
  "kind": "identitytoolkit#EmailLinkSigninResponse",
  "idToken": "eyJhbGciOiJSUzI1NiIsImtpZCI6Ij...",
  "email": "user@example.com",
  "refreshToken": "AEu4IL1...",
  "expiresIn": "3600",
  "localId": "w3nXrQz8tH1l5H7hF8R9ZpA2y0L2",
  "isNewUser": false
}
```

Fields explained:
```sh
idToken → The Authentication ID token (JWT). You can verify it on your backend using the Firebase Admin SDK.
refreshToken → Used to get new ID tokens when expired.
expiresIn → Lifetime of the ID token (in seconds).
localId → The user’s UID.
email → The user’s email.
```
---

#### 2.2. Sign In With Email + Password
Sign in a user with an email and password by issuing an HTTP POST request to the `signInWithPassword` endpoint.

**Reference:** [Sign in pre-registered user (Google Docs)](https://cloud.google.com/identity-platform/docs/use-rest-api#section-sign-in-email-password)

**Sample curl:**
```sh
curl 'https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=AIzaSyC90ZauZKIOUltCNFpl_R5ThCPm90FPkIU' \
  -H 'Content-Type: application/json' \
  --data-binary '{"email":"user@example.com","password":"password123","returnSecureToken":true}'
```

**Sample Response:**
```json
{
  "localId": "ZY1rJK0eYLg...",
  "email": "[user@example.com]",
  "displayName": "",
  "idToken": "[ID_TOKEN]",
  "registered": true,
  "refreshToken": "[REFRESH_TOKEN]",
  "expiresIn": "3600"
}
```
  - `idToken`: JWT you can verify on your backend
  - `refreshToken`: use to renew session without re-login

### 3. Receive Access Token (JWT) and Refresh Token
When you sign in with a pre-registered user (`signInWithPassword`) or passwordless sign-in, you’ll receive an Access Token (JWT) and a Refresh Token. In either case, the response will include the following fields:

**Sample Response:**
```json
{
  "idToken": "[ID_TOKEN]",
  "email": "",
  "refreshToken": "[REFRESH_TOKEN]",
  "expiresIn": "3600",
  "localId": "Jws4SVjpT..."
}
```

---

### 4. Get the Certificate/Public Key to Verify JWT Signature
Google-managed key pairs are automatically rotated and used for signing for a maximum of two weeks. The rotation process is probabilistic; usage of the new key will gradually ramp up and down over the key’s lifetime.

You can get the Google Public Signing Keys used by GCP Identity Platform using this JWKS Endpoint:
[https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com](https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com)

**Reference:** [Oauth 2.0 JWK rotate period (Google Docs)](https://discuss.google.dev/t/oauth-2-0-jwk-rotate-period/133471)

**Sample Output:**
```json
{
  "keys": [
    {
      "kid": "a5a005e97cb1e427300e52cdd50f0b36cd80629b",
      "e": "AQAB",
      "kty": "RSA",
      "alg": "RS256",
      "n": "rYmjgQOtXjOBB9YLLmRGLopC5465C7f_RNJfsZRB0X599gUPVUZ1tgWvQKWPaQ20H8mH7HHZa9--umXHFmg8urHdNaqOP-Ox51m9817ve0mFVqz1l7gN0Dx6UXLKDiHvxa4-pCale62wnmfFJdxtNYTaHICZPiY7FFVhIcTkZxulEni_crdnMirOcNXcHUouEX0Vp_d4JycALHolANj31zrMlmp6LopxpzJTxWiIHrW4nZ9v6I1LmoUKVI0Ll-Gj_qIh-7KQk_AhP3k0R2Bk-lOAY8WkZ3-rWG2wvmERsWZZx06uSQni2NRMPXbajXD1ogXAOy9xHb6aDcXb6CR4Gw",
      "use": "sig"
    },
    {
      "kid": "9d2130fef02587fd88618869822734ef76a11165",
      "kty": "RSA",
      "n": "viEkDYFahRPiz5LeAq1cXC46HAYZN9oTm9EZQKSagxvWzypsZ4xgYALtw4LVcHXAGMMtWLmCtTO0d5-5ZPf75PI7siqMsMOlrqV2WSLH3eqawY5JGKCFpjSqRwGAzJxAwLe3RfbPO8NyE3bsHVevrRgpkuzOfy-yweNxlQFJWj22mvfznnJoo463wyCjd-mboq_wgrkfsBh09upufh5Q1e-W_NtdrzxcAQ4-NJFkVP6XAYrmi_LRXGL3Zb1dVp6V-pk3BM1qqV5ZTL46h5F58IYIZ7Q4BB1ZuWvT5EX5mjbo206HWJvoLuVMcNzkYjMIZstlBxU0RniyhoLS4r7I4Q",
      "use": "sig",
      "e": "AQAB",
      "alg": "RS256"
    }
  ]
}
```

**General Information:**
  - For Firebase Auth / Google Identity, the JWKS is here: [https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com](https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com)
  - The well-known JWKS endpoint for Google OAuth2 is: [https://www.googleapis.com/oauth2/v3/certs](https://www.googleapis.com/oauth2/v3/certs)
  - Decode, verify, and generate [JSON Web Tokens](https://www.jwt.io), which are an open, industry standard.

#### JWT Signature Verification (Optional)
**Reference** [Get user data](https://firebase.google.com/docs/reference/rest/auth?utm_source=chatgpt.com#section-get-account-info)

Sample curl:

```sh
curl -X POST "https://identitytoolkit.googleapis.com/v1/accounts:lookup?key=AIzaSyC90ZauZKIOUltCNFpl_R5ThCPm90FPkIU" \
  -H "Content-Type: application/json" \
  -d '{ "idToken": "YOUR_ID_TOKEN_HERE"}'
```
Example Response:
```json
{
  "kind": "identitytoolkit#GetAccountInfoResponse",
  "users": [
    {
      "localId": "Z5eSYloY0Kfw0DY3nGuDS8VBu023",
      "email": "user@example.com",
      "passwordHash": "UkVEQUN...",
      "emailVerified": true,
      "passwordUpdatedAt": 1761151738892,
      "providerUserInfo": [
        {
          "providerId": "password",
          "federatedId": "user@example.com",
          "email": "user@example.com",
          "rawId": "user@example.com"
        }
      ],
      "validSince": "1761151738",
      "disabled": false,
      "lastLoginAt": "1761260420637",
      "createdAt": "1761148075925",
      "emailLinkSignin": true,
      "lastRefreshAt": "2025-10-23T23:00:20.637Z"
    }
  ]
}
```

### 5. Get a New JWT Access Token Using the Refresh Token
You can refresh an Identity Platform ID token by issuing an HTTP POST request to the `token` endpoint.

**Reference:** [Exchange a refresh token for an ID token (Google Docs)](https://cloud.google.com/identity-platform/docs/use-rest-api#section-refresh-token)

**Sample curl:**
```sh
curl 'https://securetoken.googleapis.com/v1/token?key=AIzaSyC90ZauZKIOUltCNFpl_R5ThCPm90FPkIU' \
-H 'Content-Type: application/x-www-form-urlencoded' \
--data 'grant_type=refresh_token&refresh_token=[REFRESH_TOKEN]'
```

**Sample Response:**
```json
{
  "access_token": "[ACCESS_TOKEN]",
  "expires_in": "3600",
  "token_type": "Bearer",
  "refresh_token": "[REFRESH_TOKEN]",
  "id_token": "[ID_TOKEN]",
  "user_id": "tRcfmLH7o2XrNELi...",
  "project_id": "1234567890"
}
```
---

#### Notes on HIPAA Compliance
- If your project is under Firebase Auth (Firebase Console), it is **not HIPAA compliant**.
- If it is a Google Cloud Identity Platform project with a signed BAA, it **can be HIPAA compliant**.
- Google’s built-in email delivery system sends OOB links.

**Do NOT include PHI in:**
  - Email subject or body templates
  - Action URLs or query parameters

_Note: These emails are not intended for PHI transport; including PHI could break HIPAA compliance._

**For HIPAA-safe email delivery:**
  - Disable automatic OOB emails
  - Use your own secure mail provider
  - Generate and deliver the link yourself via your secure system (no PHI in the URL)

#### Daily quota for email sign-in
- Be aware of quotas: when using many sends of email link flows. you might get a warning like this:
"QUOTA_EXCEEDED : Exceeded daily quota for email sign-in."