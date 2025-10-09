# PR Summary: README Updates

## What does this PR do?
- Updates the README to document all authentication endpoints and usage examples for the GCP Identity Platform Go microservice.
- Adds step-by-step instructions for obtaining and setting up the GCP Identity Platform API key and project ID.
- Provides detailed curl examples for all endpoints, including login, signup, token validation, custom token sign-in, JWT decoding, and public key retrieval.
- Clarifies project structure and adds links to relevant Google Cloud and JWT resources.
- Improves onboarding and developer experience with clear, actionable setup and usage guidance.

## Why is this needed?
- Ensures users and contributors have up-to-date, accurate documentation for all features and endpoints.
- Reduces onboarding friction and support requests by providing clear instructions and examples.
- Supports standards compliance and best practices for GCP Identity Platform integration.

## How to test?
- Follow the README instructions to set environment variables, run the service, and call each endpoint using the provided curl commands.
- Verify that each endpoint responds as documented.

## Additional Notes
- No breaking changes; documentation only.
- README now covers all endpoints, including `/public-key` for PEM key retrieval.
- See README for full usage and endpoint documentation.
