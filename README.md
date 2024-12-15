# Registration and Login Flow with Auth0: Customization and Extensibility

This document highlights areas of customization available out-of-the-box (OOTB) in Auth0, along with links to relevant documentation for extensibility features.


## Step by step setup for testing  within auth0

This section provides a step-by-step guide for setting up your Auth0 environment to support the described flows.

### 1. Create an Auth0 Tenant
- Sign up for Auth0 at [Auth0 Sign Up](https://auth0.com/signup) if you don’t already have an account.
- Create a tenant from the Auth0 Dashboard.

### 2. Set Up Applications
- Go to the Applications tab in the Auth0 Dashboard.
- Click **Create Application** and choose the application type (e.g., Single Page App, Regular Web App).
- Configure application settings, such as callback URLs and allowed origins.
  - Example Callback URL: `https://your-app/callback`
  - Example Allowed Origins: `https://your-app`

### 3. Configure Database Connections
- Navigate to **Authentication > Database** in the Dashboard.
- Create a new database connection or use the default one.
- Enable this connection for your application.
- Enable passkeys and enable email as an attribute and within the email attribute configuration enable the use OTP for email verification


### 5. Customize Branding
- Go to **Branding > Universal Login** to customize the login and signup pages.
- Add your logo, colors, and any custom CSS.
- Use the Preview feature to test the look and feel.


### 6. Create Actions
- Go to **Actions > Library** and create a new action for the Post-Login trigger.
- Copy the code for the post login action and use within this action
- Add the provided `onExecutePostLogin` and `onContinuePostLogin` code.
- Deploy the action and link it to the Post-Login trigger.

### 7. Configure Secrets in Actions
- While editing the action, navigate to the **Secrets** tab.
- In this example we addded secrets such as `ONFIDO_ID_VERIFICATION_URL`, `ONFIDO_REGION`, `ONFIDO_API_TOKEN`, and `SESSION_TOKEN_SECRET`.

### 8. Test the Flow
- Use a test user to simulate the signup and login flows.
- Ensure that OTP emails are sent, ID verification is triggered, and the user can complete all steps.

### 9. Monitor Logs and Metrics
- Navigate to **Monitoring > Logs** to debug issues.
- Use real-time logs to verify actions and API calls.



## Registration Flow (Redirect)

### Flow Overview

1. **User Initiates Signup**: The user clicks on "Sign Up" in your app and is redirected to the Auth0 signup page.
2. **OTP Verification**:
   - The user enters their email and clicks "Continue".
   - Auth0 sends an OTP to the user's email using a customizable email template.
   - The user enters the OTP on the Auth0-hosted page.
3. **Passkey Option**:
   - After OTP verification, the user is presented with the option to create a passkey or continue without it.
4. **Password Creation**:
   - The user chooses to continue without a passkey and creates a password.
5. **ID Verification**:
   - A Post-Login Action redirects the user to an ID verification flow.
   - The user completes ID verification via Onfido.
   - Upon successful ID verification, the user is logged in.

### Customizable Areas

- **Signup Page Branding**: Use [Auth0 Universal Login](https://auth0.com/docs/authenticate/login/auth0-universal-login) to customize the signup page’s look and feel.
- **Email Template for OTP**: Customize the [verification email template](https://auth0.com/docs/customize/email/email-templates) for OTP delivery.
- **Actions for Post-Login**: Leverage [Auth0 Actions](https://auth0.com/docs/customize/actions) to extend the flow and implement custom logic, such as redirecting to ID verification.
- **Redirects in Actions**: Use [Action Redirects](https://auth0.com/docs/customize/actions/explore-triggers/signup-and-login-triggers/login-trigger/redirect-with-actions) to handle the external ID verification process and return control to Auth0.

### Post-Login Action Code

The following code handles the redirection to Onfido for ID verification and updates the user’s metadata upon completion:

```javascript
const { Onfido, Region } = require('@onfido/api');

exports.onExecutePostLogin = async (event, api) => {
  if (event.user.app_metadata && event.user.app_metadata.idvVerified === true) return;

  const ONFIDO_ID_VERIFICATION_URL = event.secrets.ONFIDO_ID_VERIFICATION_URL;
  const ONFIDO_REGION = event.secrets.ONFIDO_REGION;
  const ONFIDO_API_TOKEN = event.secrets.ONFIDO_API_TOKEN;
  const SESSION_TOKEN_SECRET = event.secrets.SESSION_TOKEN_SECRET;

  const onfidoClient = new Onfido({
    apiToken: ONFIDO_API_TOKEN,
    region: Region[ONFIDO_REGION] || Region.EU,
  });

  try {
    const applicant = (event.user.app_metadata && event.user.app_metadata.applicantId)
      ? await onfidoClient.applicant.find(event.user.app_metadata.applicantId)
      : await onfidoClient.applicant.create({
          firstName: event.user.given_name || 'anon',
          lastName: event.user.family_name || 'anon',
          email: event.user.email || 'anon@example.com',
          location: { ip_address: event.request.ip },
          consents: [{ name: "privacy_notices_read", granted: true }],
        });

    api.user.setAppMetadata("applicantId", applicant.id);

    const sessionToken = api.redirect.encodeToken({
      secret: SESSION_TOKEN_SECRET,
      expiresInSeconds: 300,
      payload: { iss: `https://${event.request.hostname}/`, aud: "onfido", applicant: applicant.id },
    });

    const url = `${ONFIDO_ID_VERIFICATION_URL}${sessionToken}`;
    api.redirect.sendUserTo(url);
  } catch (e) {
    api.access.deny(`IDV could not be started: ${e.message || "Something went wrong!"}`);
  }
};

exports.onContinuePostLogin = async (event, api) => {
  const SESSION_TOKEN_SECRET = event.secrets.SESSION_TOKEN_SECRET;

  try {
    const payload = api.redirect.validateToken({
      secret: SESSION_TOKEN_SECRET,
      tokenParameterName: "session_token",
    });
    api.user.setAppMetadata("idvVerified", true);
  } catch (e) {
    api.access.deny("Your ID verification failed. Please contact support.");
  }
};
```

## Authentication Flow

### Flow Overview

1. **User Logs In**: The user enters their credentials (email and password).
2. **Passkey Prompt**:
   - Since the ID verification was completed during registration, the user is prompted to create a passkey.
   - The user creates a passkey for future logins.

### Customizable Areas

- **Login Page Branding**: Customize the login page through [Universal Login branding](https://auth0.com/docs/customize/universal-login-pages).
- **Passkey Setup**: Enable passkeys and customize  [passkey experience](https://auth0.com/docs/authenticate/database-connections/passkeys).

## Extensibility Features and Links

### Redirects in Actions: Detailed Example

Redirects in Auth0 Actions allow you to pause the login process, redirect the user to an external site for additional processing, and then return to Auth0 to resume the flow. This is particularly useful for workflows like identity verification.

#### Example: Onfido Integration
1. **Redirect to Onfido**:
   - The `onExecutePostLogin` function in your Action initiates a redirect to the hosted application's Onfido’s verification page.
   - The user completes the required steps (e.g., uploading a document).
2. **Return to Auth0**:
   - Onfido redirects the user back to Auth0 with a `session_token`.
   - Auth0’s `onContinuePostLogin` function validates this token to confirm the user’s verification status.

#### External Site Flow
For example, if the external site is for Onfido:
- Auth0 encodes the necessary information (e.g., user ID, applicant ID) in a token and appends it to the Onfido URL.
- Onfido uses this information to identify the user and guide them through the verification process.
- Once completed, Onfido redirects the user back to Auth0 with the encoded `session_token`.
- The code for the external site providing the ID verification is available at [Onfido ID Verification sample code](https://github.com/pushpabrol/onfido-idv-verification-express-app)
### Branding
- [Universal Login Branding](https://auth0.com/docs/customize/universal-login-pages)
- [Email Template Customization](https://auth0.com/docs/customize/email/email-templates)

### Actions
- [Actions Overview](https://auth0.com/docs/customize/actions)
- [Redirect with Actions](https://auth0.com/docs/customize/actions/explore-triggers/signup-and-login-triggers/login-trigger/redirect-with-actions)
- [Secrets in Actions](https://auth0.com/docs/customize/actions/write-your-first-action#add-a-secret)

### State Management in Actions
Auth0 acts as a state machine during Actions execution. When using redirects, Actions pause the login process, allowing external interactions before resuming. Learn more: [Actions State Management](https://auth0.com/docs/customize/actions/explore-triggers/signup-and-login-triggers/login-trigger/redirect-with-actions#start-a-redirect).

### ID Verification Sample
- [Onfido Integration](https://marketplace.auth0.com/integrations/onfido-identity-verification)


### Additional Resources

- [Auth0 Management API](https://auth0.com/docs/api/management/v2/introduction)
- [Database Connections](https://auth0.com/docs/authenticate/database-connections)


