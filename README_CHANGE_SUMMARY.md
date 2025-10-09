# README Change Summary

## Summary of Changes
- Expanded setup instructions for enabling Identity Platform and configuring email/password provider.
- Clarified that service accounts are not required for user authentication.
- Added step-by-step guide for creating and restricting API keys in Google Cloud Console.
- Provided detailed environment variable setup for API key and project ID.
- Documented all REST API endpoints with example curl commands:
  - `/login` for user authentication
  - `/signup` for registration
  - `/validate-token` for token validation
  - `/custom-token` for custom token sign-in
  - `/secure/decode-token` for JWT decoding
  - `/public-key` for retrieving the public key from a GCP token
- Added project structure overview and licensing information.
- Included links and instructions for JWT debugging and verification.
- Added FAQ section clarifying service account and project requirements for GCP Identity Platform.

## Impact
- Improves onboarding and developer experience with clear, actionable documentation.
- Ensures all endpoints and features are discoverable and easy to test.
- Reduces confusion about service account usage and GCP project requirements.
