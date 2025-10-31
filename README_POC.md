

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

### 1. Sign Up User (Email-only / Password-less)
#### Sign Up User (Email-only / Password-less) with `GCP Identity Platform`
- GCP Identity Platform does **not** support signing up users with email only.

#### Sign up with email / password (Optional)
You can create a new email and password user by issuing an HTTP POST request

**Reference** [Sign up with email / password](https://cloud.google.com/identity-platform/docs/use-rest-api#section-create-email-password)

Sample request
```sh
curl 'https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=[API_KEY]' \
-H 'Content-Type: application/json' \
--data-binary '{"email":"[user@example.com]","password":"[PASSWORD]","returnSecureToken":true}'
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

#### Send Password Reset Email (Optional)
Used to send one-time _out-of-band_ (OOB) codes (e.g., email verification, password reset) to users.

  - With Google Identity Platform (Advanced/Enterprise Firebase Auth)
  - Firebase generates a one-time OOB action code internally and sends the link to the user's email (using default/configured template)
  - The API response does **not** contain the actual OOB link (for security reasons). That’s the expected behavior (these links can directly reset a password or verify an email).

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
  "kind": "identitytoolkit#GetOobConfirmationCodeResponse",
  "email": "[user@example.com]"
}
```

**Sample Email:**

<img src="img/email_oobcode.png" alt="email_oobcode" width="50%" />

---

### 2. Sign in user (email-only / password-less, or with pre-registered credentials)

#### Sign In User (email-only / password-less) with `GCP Identity Platform`
To sign in users by email link, you must first enable the` Email/Password provider` and `Allow passwordless login` method for your GCP IP project:
1. Go to the **Identity Providers page** in the [Google Cloud console](https://console.cloud.google.com/customer-identity/providers).
2. On the left menu, click `Providers`.
3. In the `Select a provider` list, Add and enable a new provider(`Email/Password`). 
4. In the same section, enable `Allow passwordless login` (Sign-in occurs via confirmation email link).
5. Click Save.

Reference: [Sign in a user with an email by using Identity Platform](https://docs.cloud.google.com/identity-platform/docs/sign-in-user-email)

---

#### 1. Send the email sign-in link
You can sign in a user with an email by issuing an HTTP POST request.

**Reference** [Send email](https://cloud.google.com/identity-platform/docs/use-rest-api#section-send-email-verification)

Sample curl:

```sh
curl -X POST "https://identitytoolkit.googleapis.com/v1/accounts:sendOobCode?key=[API_KEY]" \
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

#### 2. Signing in with the email link from the previous step.
---

Sample curl:
```sh
curl -X POST "https://identitytoolkit.googleapis.com/v1/accounts:signInWithEmailLink?key=[API_KEY]" \
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

#### 3. Sign In With Email + Password
Sign in a user with an email and password by issuing an HTTP POST request to the `signInWithPassword` endpoint.

**Reference:** [Sign in pre-registered user (Google Docs)](https://cloud.google.com/identity-platform/docs/use-rest-api#section-sign-in-email-password)

**Sample curl:**
```sh
curl 'https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=[API_KEY]' \
  -H 'Content-Type: application/json' \
  --data-binary '{"email":"[user@example.com]","password":"[PASSWORD]","returnSecureToken":true}'
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


#### 4. Sign In Anonymously
Sign in a user anonymously by issuing an HTTP POST request to the `signUp` endpoint.
Anonymous Sign-in must be enabled in your project settings and also included as a sing-in provider.

**Reference:** [Sign in anonymously (Google Docs)](https://cloud.google.com/identity-platform/docs/use-rest-api#section-sign-in-anonymously)

**Sample curl:**
```sh
curl 'https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=[API_KEY]' \
  -H 'Content-Type: application/json' \
  --data-binary '{"returnSecureToken":true}'
```

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
  - `idToken`: JWT you can verify on your backend
  - `refreshToken`: use to renew session without re-login

---

### 3. Receive Access Token (JWT) and Refresh Token
When you sign in with a pre-registered user (`signInWithPassword`) or passwordless sign-in, you’ll receive both an Access Token (JWT) and a Refresh Token. In either case, the response will include the following fields:

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


### 5. Get a New JWT Access Token Using the Refresh Token
You can refresh an Identity Platform ID token by issuing an HTTP POST request to the `token` endpoint.

**Reference:** [Exchange a refresh token for an ID token (Google Docs)](https://cloud.google.com/identity-platform/docs/use-rest-api#section-refresh-token)

**Sample curl:**
```sh
curl 'https://securetoken.googleapis.com/v1/token?key=[API_KEY]' \
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

# Proof of Concept: Firebase API Integration

## Setup Instructions

You’ll need:
  - A [Firebase project](https://console.firebase.google.com/) for signing-in users.
  - A configured email/password provider with email link.
  - An API key.

### Firebase Auth [REST API](https://firebase.google.com/docs/reference/rest/auth?utm_source=chatgpt.com#section-api-usage)

Where can I find my Firebase api-key
1. Go to the `Project overview` page in the `Firebase console`.
2. Click the `+ Add App` in the top bar.
3. Click `</>` button to add a web app.

You'll get a pop up with the values you need.

---

### 1. Sign Up User (Email-only / Password-less)

#### Sign Up User (Email-only / Password-less) with `Firebase`
- Firebase does **not** support signing up users with email only.

#### Sign up with email / password (Optional)
You can create a new email and password user by issuing an HTTP POST request

**Reference** [Sign up with email / password](https://firebase.google.com/docs/reference/rest/auth?utm_source=chatgpt.com#section-create-email-password)

Sample request
```sh
curl 'https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=[API_KEY]' \
-H 'Content-Type: application/json' \
--data-binary '{"email":"[user@example.com]","password":"[PASSWORD]","returnSecureToken":true}'
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

#### Sign In User (email-only / password-less) with `Firebase`
To sign in users by email link, you must first enable the` Email provider` and `Email link sign-in` method for your Firebase project:
1. In the [Firebase console](https://console.firebase.google.com/?authuser=0&_gl=1*xyhjy*_ga*MjExNjcwNDMxMi4xNzYwMDM1MzMx*_ga_CW55HF8NVT*czE3NjExNTY0MDMkbzQkZzEkdDE3NjExNTY0MDYkajU3JGwwJGgw), open the `Authentication` section.
2. On the `Sign in` method tab, add and enable a new provider(Email/Password). 
3. In the same section, enable Email link (passwordless sign-in).
4. Click Save.

Reference: [Enable Email Link sign-in for your Firebase project](https://firebase.google.com/docs/auth/web/email-link-auth?authuser=0&_gl=1*12lyr8t*_ga*MjExNjcwNDMxMi4xNzYwMDM1MzMx*_ga_CW55HF8NVT*czE3NjExNDY1OTUkbzMkZzEkdDE3NjExNDc1MjUkajYwJGwwJGgw#enable_email_link_sign-in_for_your_firebase_project)

---

#### 1. Send the email sign-in link
You can sign in a user with an email by issuing an HTTP POST request.

**Reference** [Send email](https://firebase.google.com/docs/reference/rest/auth?utm_source=chatgpt.com#section-send-email-verification)

Sample curl:

```sh
curl -X POST "https://identitytoolkit.googleapis.com/v1/accounts:sendOobCode?key=YOUR_FIREBASE_API_KEY" \
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

If firebase successfully queued the email to send. The user will soon receive an email with a link that looks like:
```sh
https://yourapp.com/finishSignIn?apiKey=AIzaSy...&oobCode=ABC123XYZ&mode=signIn
```

<img src="img/sign-in.png" alt="sign-in" width="50%" />

**Note:**
When you use `continueUrl` in Firebase, you’re not performing a sign-in operation directly. You’re passing along a URL to which Firebase will redirect the user after a particular action (like email verification, password reset, or a sign-in link in this case).

The email is not automatically included in that continue URL because:

- Security & privacy: Firebase intentionally omits the user’s email from the URL to avoid leaking it through browser history, logs, analytics, or referral headers.

- Stateless flow: The continue URL is meant to be a neutral redirect destination. Firebase doesn’t embed identifying user info there unless you explicitly choose to.

#### 2. Signing in with the email link from the previous step.
---

Sample curl:
```sh
curl -X POST "https://identitytoolkit.googleapis.com/v1/accounts:signInWithEmailLink?key=YOUR_FIREBASE_API_KEY" \
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
idToken → The Firebase Authentication ID token (JWT). You can verify it on your backend using the Firebase Admin SDK.
refreshToken → Used to get new ID tokens when expired.
expiresIn → Lifetime of the ID token (in seconds).
localId → The user’s Firebase UID.
email → The user’s email.
```
---

#### 3. Sign In User (Pre-Registered Credentials)
You can sign in a user with an email and password by issuing an HTTP POST request.

**Reference:** [Sign in with email / password (Firebase Docs)](https://firebase.google.com/docs/reference/rest/auth?utm_source=chatgpt.com#section-sign-in-email-password)

Sample request
```sh
curl 'https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=[API_KEY]' \
-H 'Content-Type: application/json' \
--data-binary '{"email":"[user@example.com]","password":"[PASSWORD]","returnSecureToken":true}'
```

Sample response
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

### 3. Receive access token (JWT) and refresh token
When you sign in with a pre-registered user (signInWithPassword) or passwordless sign-in, you’ll receive both an Access Token (JWT) and a Refresh Token. In either case, the response will include the following fields:

Sample Response:
```json
{
  "idToken": "[ID_TOKEN]",
  "email": "",
  "refreshToken": "[REFRESH_TOKEN]",
  "expiresIn": "3600",
  "localId": "Jws4SVjpT..."
}
```

### 4. Get the public key to verify JWT signature
You can get the Google Public Signing Keys used by Firebase in JWK format using this JWKS Endpoint. [https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com](https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com)

Sample Output:

```sh
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "9d2130fef02587fd88618869822734ef76a11165",
      "n": "viEkDYFahRPiz5LeAq1cXC46HAYZN9oTm9EZQKSagxvWzypsZ4xgYALtw4LVcHXAGMMtWLmCtTO0d5-5ZPf75PI7siqMsMOlrqV2WSLH3eqawY5JGKCFpjSqRwGAzJxAwLe3RfbPO8NyE3bsHVevrRgpkuzOfy-yweNxlQFJWj22mvfznnJoo463wyCjd-mboq_wgrkfsBh09upufh5Q1e-W_NtdrzxcAQ4-NJFkVP6XAYrmi_LRXGL3Zb1dVp6V-pk3BM1qqV5ZTL46h5F58IYIZ7Q4BB1ZuWvT5EX5mjbo206HWJvoLuVMcNzkYjMIZstlBxU0RniyhoLS4r7I4Q",
      "alg": "RS256",
      "e": "AQAB"
    },
    {
      "kid": "7ea09d057262e63e62ff73f03e04ad29d89f892a",
      "kty": "RSA",
      "n": "3RtsHBKQFOIQkuhAD-EJKXy0fw3bErHqI7VlLgejwFxelx_dpO-zhGGMEW5nfTDUulPk-aDBQtQdTgj-64_Gf87cK2pjRkf_9SxSXCMpTSnHIUZsF4ZrOXpk_1RSPASm9luTURM-92iF2ZpQo7tN8Anm_eAaaXl6IvuUB5c7pSqq6rXFJ7gunGaOJPeaycv25V5MVsJZxbl37nEjkxXluuXMu1Uu4HUfzxVAEOyLtLSOXNxlgA0yNz2VrSwjHVnatFWDTkrDxui4Uduec5YGjO06tLDIPydX0rsjZen7oQ_6HstMwyuiU16KcpifAvdVRMs-JyFdqrz2_TF7nOzhiQ",
      "e": "AQAB",
      "alg": "RS256",
      "use": "sig"
    }
  ]
}
```

Decode, verify, and generate [JSON Web Tokens](https://www.jwt.io), which are an open, industry standard.

#### JWT Signature Verification (Optional)
**Reference** [Get user data](https://firebase.google.com/docs/reference/rest/auth?utm_source=chatgpt.com#section-get-account-info)

Sample curl:

```sh
curl -X POST "https://identitytoolkit.googleapis.com/v1/accounts:lookup?key=YOUR_FIREBASE_API_KEY" \
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

### 5. Get a new JWT access token using the refresh token

**Reference** [Refresh the ID token](https://firebase.google.com/docs/reference/rest/auth?utm_source=chatgpt.com#section-refresh-token)

Sample curl:
```sh
curl -X POST "https://securetoken.googleapis.com/v1/token?key=YOUR_FIREBASE_API_KEY" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=refresh_token&refresh_token=YOUR_REFRESH_TOKEN"
```

Example Response:
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Ij...",
  "expires_in": "3600",
  "token_type": "Bearer",
  "refresh_token": "AEu4IL0Z6H...",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Ij...",
  "user_id": "cHJ9JbqT8YF8zPj3bD3K4D",
  "project_id": "630549311453"
}
```

#### Summary of all firebase endpoints
- Send sign-in link	POST	https://identitytoolkit.googleapis.com/v1/accounts:sendOobCode?key=API_KEY
- Complete sign-in	POST	https://identitytoolkit.googleapis.com/v1/accounts:signInWithEmailLink?key=API_KEY
- Verify ID token	POST	https://identitytoolkit.googleapis.com/v1/accounts:lookup?key=API_KEY
- Refresh token	POST	https://securetoken.googleapis.com/v1/token?key=API_KEY

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

#### Feature evolution & deprecation
- Some of the `email-link` flows used to rely on legacy systems such as Firebase Dynamic Links.
- Currently, There is a deprecation warning in the console:

```sh
The following Authentication features will stop working when Firebase Dynamic Links shuts down soon: email link authentication for mobile apps, as well as Cordova OAuth support for web apps.
Since the Firebase Dynamics links deprecation window has finished, you should take action immediately to avoid breaking your app's Authentication flow.
```

- Email link authentication previously relied on Firebase Dynamic Links … which will be shut down on August 25, 2025.
- Because of such shifts, documentation may have been trimmed or reorganised, and enum values like EMAIL_SIGNIN might not be emphasised in the higher-level documentation.
- Be aware of quotas: when using many sends of email link flows. you might get a warning like this:
"QUOTA_EXCEEDED : Exceeded daily quota for email sign-in."