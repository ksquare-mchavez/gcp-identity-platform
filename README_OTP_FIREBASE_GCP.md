

# Proof of Concept: Firebase/GCP IP API Integration

### Sign up or sign in with your phone number using SMS OTP verification.
Phone Authentication in Firebase is a secure method of verifying user identities using their phone numbers. 
It simplifies the sign-in process by sending verification codes(OTP) via SMS and ensuring that only users with access to the phone number can sign in.

#### Enable Phone Authentication
1. Navigate to the Authentication section in the Firebase Console.
2. Select the Sign-in method tab.
3. Enable Phone as a sign-in provider.

#### Implementing Phone Authentication
1. Adding Firebase SDK to Your Project
For a web application, include the Firebase SDK in your HTML file:
```sh
<!-- Firebase App (the core Firebase SDK) -->
<script src="https://www.gstatic.com/firebasejs/9.12.1/firebase-app-compat.js"></script>
<!-- Firebase Auth (for authentication) -->
<script src="https://www.gstatic.com/firebasejs/9.12.1/firebase-auth-compat.js"></script>
```

2. Initialize Firebase
Initialize Firebase in your JavaScript file using the configuration details from your Firebase project settings:

```sh
const firebaseConfig = {
    apiKey: "AIzaSyC90ZauZKIOUltCNFpl_R5ThCPm90FPkIU",
    authDomain: "biolinq-test.firebaseapp.com",
    projectId: "biolinq-test",
    storageBucket: "biolinq-test.firebasestorage.app",
    messagingSenderId: "788107264953",
    appId: "1:788107264953:web:470779c0d9f5ba0b98ec68"
};

// Initialize Firebase
firebase.initializeApp(firebaseConfig);
```

3. Setting Up Recaptcha Verifier
Firebase requires a reCAPTCHA verifier to ensure that phone authentication requests are made by real users:
```sh
const recaptchaVerifier = new firebase.auth.RecaptchaVerifier('recaptcha-container', {
  'size': 'invisible',
  'callback': function(response) {
    // reCAPTCHA solved - allow signInWithPhoneNumber
  }
});
```
**Explanation**: This code initializes an invisible reCAPTCHA verifier (RecaptchaVerifier) for phone authentication in Firebase. When the reCAPTCHA challenge is solved, it allows the signInWithPhoneNumber function to proceed

4. Sending Verification Code
Create a function to handle sending the verification code to the user's phone number:
```sh
function sendVerificationCode(phoneNumber) {
  const appVerifier = recaptchaVerifier;
  firebase.auth().signInWithPhoneNumber(phoneNumber, appVerifier)
    .then((confirmationResult) => {
      // SMS sent. Prompt user to type the code from the message
      window.confirmationResult = confirmationResult;
    })
    .catch((error) => {
      // Handle errors here
      console.error('Error during sign-in:', error.message);
    });
}
```
**Explanation**: This function sends a verification code to the provided phone number using Firebase authentication. It uses the signInWithPhoneNumber method with the reCAPTCHA verifier (appVerifier) to initiate the verification process. If successful, it stores the confirmationResult for further verification. If there's an error, it logs the error message.

5. Verifying the Code
Create a function to verify the code entered by the user:

```sh
function verifyCode(verificationCode) {
  confirmationResult.confirm(verificationCode)
    .then((result) => {
      // User signed in successfully
      const user = result.user;
      console.log('User:', user);
    })
    .catch((error) => {
      // Handle errors here
      console.error('Error during verification:', error.message);
    });
}
```
Explanation: This function verifies the code entered by the user for phone authentication. It uses the confirm method on confirmationResult (previously obtained from signInWithPhoneNumber) to complete the authentication process. If successful, it logs the user information. If there's an error, it logs the error message.

6. Handling User Authentication State
Monitor the user's authentication state to check if they are signed in:
```sh
firebase.auth().onAuthStateChanged((user) => {
  if (user) {
    // User is signed in
    console.log('User is signed in:', user);
  } else {
    // No user is signed in
    console.log('No user is signed in');
  }
});
```
**Explanation**: This code sets up a listener for authentication state changes using Firebase Authentication. When the authentication state changes (i.e., a user signs in or signs out), the provided callback function is called. If a user is signed in, it logs the user information. If no user is signed in, it logs a message indicating that

#### Phone Authentication. 
Create a new file called index.html and insert the following code:

```sh
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Firebase Phone Authentication</title>
  <style>
    body {
      font-family: 'Segoe UI', Arial, sans-serif;
      background: linear-gradient(120deg, #f8fafc 0%, #e0e7ff 100%);
      margin: 0;
      padding: 0;
      min-height: 100vh;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
    }
    h1 {
      color: #1976d2;
      margin-bottom: 24px;
      font-size: 2rem;
      letter-spacing: 1px;
    }
    .auth-container {
      background: #fff;
      border-radius: 16px;
      box-shadow: 0 4px 24px rgba(59, 130, 246, 0.08);
      padding: 32px 24px;
      display: flex;
      flex-direction: column;
      gap: 16px;
      min-width: 320px;
      max-width: 360px;
    }
    input[type="text"] {
      padding: 10px 12px;
      border: 1px solid #d1d5db;
      border-radius: 8px;
      font-size: 1rem;
      outline: none;
      transition: border-color 0.2s;
    }
    input[type="text"]:focus {
      border-color: #3b82f6;
    }
    button {
      padding: 10px 0;
      background: #1976d2;
      color: #fff;
      border: none;
      border-radius: 8px;
      font-size: 1rem;
      cursor: pointer;
      transition: background 0.2s;
      box-shadow: 0 2px 8px rgba(59, 130, 246, 0.08);
    }
    button:hover {
      background: #2563eb;
    }
    #recaptcha-container {
      margin-bottom: 8px;
    }
  </style>
  <!-- Firebase App (the core Firebase SDK) -->
  <script src="https://www.gstatic.com/firebasejs/9.12.1/firebase-app-compat.js"></script>
  <!-- Firebase Auth SDK -->
  <script src="https://www.gstatic.com/firebasejs/9.12.1/firebase-auth-compat.js"></script>
</head>
<body>
  <h1>Sign In with Phone Number</h1>
  <div class="auth-container">
    <div id="recaptcha-container"></div>
    <input type="text" id="phone-number" placeholder="Enter phone number" required>
    <button onclick="sendVerificationCode()">Send Verification Code</button>
    <input type="text" id="verification-code" placeholder="Enter verification code" required>
    <button onclick="verifyCode()">Verify Code</button>
  </div>
  <script>
    const firebaseConfig = {
      apiKey: "AIzaSyC90ZauZKIOUltCNFpl_R5ThCPm90FPkIU",
      authDomain: "biolinq-test.firebaseapp.com",
      projectId: "biolinq-test",
      storageBucket: "biolinq-test.firebasestorage.app",
      messagingSenderId: "788107264953",
      appId: "1:788107264953:web:470779c0d9f5ba0b98ec68", 
    };

    // Initialize Firebase
    firebase.initializeApp(firebaseConfig);
    const recaptchaVerifier = new firebase.auth.RecaptchaVerifier('recaptcha-container', {
      'size': 'invisible',
      'callback': function(response) {
        console.log('reCAPTCHA solved');
      }
    });

    let confirmationResult;
    function sendVerificationCode() {
      const phoneNumber = document.getElementById('phone-number').value;
      const appVerifier = recaptchaVerifier;
      firebase.auth().signInWithPhoneNumber(phoneNumber, appVerifier)
        .then((result) => {
          confirmationResult = result;
          console.log('Verification code sent');
        })
        .catch((error) => {
          console.error('Error during sign-in:', error.message);
        });
    }

    function verifyCode() {
      const verificationCode = document.getElementById('verification-code').value;
      confirmationResult.confirm(verificationCode)
        .then((result) => {
          const user = result.user;
          console.log('User:', user);
          alert('Verification successful! User signed in.');
        })
        .catch((error) => {
          console.error('Error during verification:', error.message);
          alert('Verification failed. Please try again.');
        });
    }

    firebase.auth().onAuthStateChanged((user) => {
      if (user) {
        console.log('User is signed in:', user);
      } else {
        console.log('No user is signed in');
      }
    });
  </script>
</body>
</html>
```

Output: \
<img src="img/image.png" alt="passwordles-flow" width="40%" />

Testing data:
```
Phone Number: +16505551234
OTP Code: 123456
```

Steps
```
1. Click on Send Verification Code 
2. Click on Verify Code
```

<img src="img/image-1.png" alt="sign-in" width="40%" />

You can also retrieve user information, such as the token, through the browser console.
<img src="img/image-2.png" alt="user-info" width="50%" />

---

#### Phone Authentication(Optional)
There is also a react app for getting the token as well.
You can access the repo [here](https://github.com/ksquare-mchavez/phone-otp)

#### Step 1. Enter Phone Number
<img src="img/image-3.png" alt="sign-in" width="30%" />

#### Step 2. Enter OTP
<img src="img/image-4.png" alt="sign-in" width="30%" />


**Response**:
```json
{
  "token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Ij...",
  "token_data": {
    "auth_time": 1762282418,
    "iss": "https://securetoken.google.com/biolinq-test",
    "aud": "biolinq-test",
    "exp": 1762286018,
    "iat": 1762282418,
    "sub": "i6oJpdrt8qU7y...",
    "uid": "i6oJpdrt8qU7y...",
    "firebase": {
      "sign_in_provider": "phone",
      "tenant": "",
      "identities": {
        "phone": [
          "+16505551234"
        ]
      }
    }
  }
}
```