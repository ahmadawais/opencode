# GitHub Copilot OAuth Implementation Spec

## Overview

This document provides a comprehensive specification of how OAuth authentication for GitHub Copilot is implemented in the Command Code CLI. Use this as a reference to implement OAuth flows for GitHub Copilot provider.

## Architecture Components

### 1. Core OAuth Module (`packages/command/src/auth/github-copilot.ts`)

The central OAuth implementation using GitHub's Device Flow for OAuth authentication.

#### Constants

```typescript
const CLIENT_ID = 'Iv1.b507a08c87ecfe98';
const DEVICE_CODE_URL = 'https://github.com/login/device/code';
const TOKEN_URL = 'https://github.com/login/oauth/access_token';
const SCOPES = 'read:user';
```

#### OAuth Flow

GitHub Copilot uses **OAuth 2.0 Device Authorization Grant** flow (not PKCE), which is optimized for devices with limited input capabilities or no web browser.

#### Core Functions

**1. `createDeviceCode()`**
- Makes POST request to DEVICE_CODE_URL with:
  - `client_id`: CLIENT_ID
  - `scope`: SCOPES
- Receives device code response:
  ```typescript
  {
    device_code: string;
    user_code: string;
    verification_uri: string;
    expires_in: number;
    interval: number; // Polling interval in seconds
  }
  ```
- Returns: `{deviceCode, userCode, verificationUri, expiresIn, interval}`

**2. `pollForToken(deviceCode, interval)`**
- Polls TOKEN_URL at specified intervals with:
  - `client_id`: CLIENT_ID
  - `device_code`: Device code from step 1
  - `grant_type`: 'urn:ietf:params:oauth:grant-type:device_code'
- Handles polling responses:
  - `authorization_pending`: Continue polling
  - `slow_down`: Increase polling interval by 5 seconds
  - `expired_token`: Device code expired, restart flow
  - `access_denied`: User denied authorization
  - Success: Returns access token
- Receives token response:
  ```typescript
  {
    access_token: string;
    token_type: string;
    scope: string;
  }
  ```
- Stores tokens using Auth.set() with:
  ```typescript
  {
    type: 'oauth',
    access: access_token,
    expires: Date.now() + 8 * 60 * 60 * 1000, // 8 hours
    refresh: '', // GitHub device flow doesn't provide refresh tokens
  }
  ```

**3. `refreshAccessToken()`**
- GitHub Copilot access tokens from device flow typically don't expire for 8 hours
- When token is about to expire, user must re-authenticate through device flow
- Checks if token is expired (with 30-minute buffer)
- If expired, removes auth data and returns undefined (triggers re-authentication)
- Returns existing access token if still valid

**4. `getValidAccessToken()`**
- Retrieves stored auth data
- For OAuth, calls refreshAccessToken()
- Returns valid access token or undefined
- If undefined, triggers re-authentication flow

**5. `getCopilotToken(accessToken)`**
- Makes authenticated request to GitHub's Copilot token endpoint
- Exchanges GitHub access token for Copilot-specific token
- Endpoint: `https://api.github.com/copilot_internal/v2/token`
- Headers:
  - `Authorization`: `Bearer ${accessToken}`
  - `Accept`: `application/json`
- Returns Copilot token with expiry

### 2. Auth Storage Module (`packages/command/src/auth/index.ts`)

Manages persistent storage of authentication credentials. Same as Anthropic implementation.

#### Storage Location
- Directory: `~/.commandcode/` (INFO.DIRECTORY_NAME)
- File: `models.json` (production), `models.dev.json` (dev), `models.staging.json` (staging)
- Permissions: 0o600 (owner read/write only)

#### Auth Data Types

```typescript
// OAuth authentication
type OAuth = {
  type: 'oauth';
  refresh: string;  // Empty for GitHub device flow
  access: string;   // GitHub access token
  expires: number;  // Unix timestamp in milliseconds
};

// API Key authentication (alternative)
type ApiKey = {
  type: 'api';
  key: string;
};

type Info = OAuth | ApiKey;
```

#### Storage Format
```json
{
  "github-copilot": {
    "type": "oauth",
    "access": "gho_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "expires": 1234567890000,
    "refresh": ""
  }
}
```

### 3. CLI Commands (`packages/command/src/commands/github-copilot-auth.ts`)

Three authentication commands exposed via Commander.js.

#### `auth login`
1. Generate device code using `createDeviceCode()`
2. Display user code and verification URL to user
3. Open browser automatically to verification URL
4. Start polling for token using `pollForToken()`
5. Display progress/status during polling
6. On success, store token
7. Display success message

Example output:
```
Please visit: https://github.com/login/device
Enter code: XXXX-XXXX
Waiting for authorization...
```

#### `auth logout`
1. Check if authenticated
2. Call `Auth.remove('github-copilot')`
3. Display success message

#### `auth status`
1. Call `Auth.get('github-copilot')`
2. Display authentication status
3. Show token expiry information

### 4. Interactive UI Component (`packages/command/src/components/github-copilot-auth.tsx`)

React-based authentication flow using Ink.

#### Flow
1. Generate device code on component mount
2. Display user code prominently
3. Prompt user to open browser (Y/n)
4. Open browser on confirmation to verification URL
5. Start polling for authorization
6. Display polling status with spinner
7. Handle authorization completion
8. Update user config to set provider to GitHub Copilot
9. Call success callback

#### Features
- Automatic browser opening
- Clear user code display
- Polling status indicator
- Error handling for expired/denied authorization
- ESC key cancellation
- Timeout handling

### 5. API Client Integration (`packages/command/src/clients/github-copilot/github-copilot.ts`)

GitHub Copilot SDK client initialization with OAuth support.

```typescript
// Initialize with OAuth token
const oauthToken = await GitHubCopilotOAuth.getValidAccessToken();
if (oauthToken) {
  // Exchange for Copilot token
  const copilotToken = await GitHubCopilotOAuth.getCopilotToken(oauthToken);
  
  this.instance = new GitHubCopilotClient({
    authToken: copilotToken,
    baseURL: 'https://api.githubcopilot.com',
  });
}
```

### 6. API Request Flow

When making API requests to the Command Code backend:

#### CLI Side (`packages/command/src/chat/context-engine.ts`)

```typescript
let token: string | undefined = undefined;
if (provider === PROVIDERS.GITHUB_COPILOT) {
  token = await GitHubCopilotOAuth.getValidAccessToken();
  validateOAuthToken({token, provider});
}

const headers: Record<string, string> = {
  [HEADERS.INTERNAL_FLAG_HEADER]: oauthEnforced.toString(),
  // ... other headers
};

if (token) {
  headers[HEADERS.OAUTH_TOKEN] = `Bearer ${token}`;
}
```

#### API Side (`packages/api/src/utils/auth/validate-oauth-access.ts`)

Server validates OAuth access (same pattern as Anthropic).

## Security Features

### 1. Device Authorization Flow
- Optimized for CLI applications
- No client secret required
- User authenticates via browser on trusted device
- Device code separate from user code

### 2. Polling Security
- Respects rate limiting with configurable intervals
- Handles slow_down responses
- Timeout after device code expiration

### 3. Secure Storage
- File permissions: 0o600 (owner read/write only)
- Tokens stored locally in user's home directory
- Environment-specific files (dev/staging/prod)

### 4. Token Expiration
- Tokens expire after 8 hours typically
- User must re-authenticate when expired
- 30-minute buffer for token refresh checks
- Graceful fallback to re-authentication

### 5. Access Control
- Internal flag (`--co`) required for OAuth usage
- Admin or pre-approved user validation on API side
- Header-based token transmission

## Provider Configuration

Located in `packages/command/src/utils/provider-config.ts`:

```typescript
{
  [PROVIDER.GITHUB_COPILOT]: {
    name: 'GitHub Copilot',
    description: 'GitHub Copilot',
    requiresAuth: true,
    hidden: true, // Only visible with --co flag (admin only)
    authComponent: GitHubCopilotAuth,
    checkAuth: async () => {
      const token = await GitHubCopilotOAuth.getValidAccessToken();
      return !!token;
    },
  },
}
```

## Constants & Configuration

### Shared Constants (`packages/shared/src/constants.ts`)

```typescript
// OAuth control flags
export const INTERNAL_FLAG = '--co';
export const DISABLE_OAUTH_FLAG = '--xco';

// HTTP Headers
export const HEADERS = {
  INTERNAL_FLAG_HEADER: 'x-co-flag',
  OAUTH_TOKEN: 'x-oauth-token',
  PROJECT_SLUG: 'x-project-slug',
  TASTE_LEARNING: 'x-taste-learning',
  TASTE_USAGE: 'x-taste-usage',
};

// Provider constants
export const PROVIDERS = {
  GITHUB_COPILOT: 'github-copilot',
} as const;
```

## Implementation Checklist for GitHub Copilot

### 1. Create OAuth Module
- [x] Create `packages/command/src/auth/github-copilot.ts`
- [x] Implement Device Authorization Flow
- [x] Define OAuth constants (CLIENT_ID, URLs)
- [x] Implement `createDeviceCode()`
- [x] Implement `pollForToken()`
- [x] Implement `refreshAccessToken()` (re-auth pattern)
- [x] Implement `getValidAccessToken()`
- [x] Implement `getCopilotToken()` for token exchange
- [x] Use Auth.set/get/remove for storage

### 2. Create CLI Command
- [x] Create `packages/command/src/commands/github-copilot-auth.ts`
- [x] Implement `login`, `logout`, `status` subcommands
- [x] Add browser auto-open to device verification URL
- [x] Implement polling UI with status updates
- [x] Handle polling responses and errors
- [x] Display clear status messages

### 3. Create UI Component
- [x] Create `packages/command/src/components/github-copilot-auth.tsx`
- [x] Implement interactive auth flow
- [x] Display device code and verification URL
- [x] Add browser confirmation prompt
- [x] Implement polling with visual feedback
- [x] Add error handling and cancellation
- [x] Update user config on success

### 4. Integrate with Client
- [x] Update provider client initialization
- [x] Call `getValidAccessToken()` before creating client
- [x] Exchange access token for Copilot token
- [x] Pass Copilot token to SDK constructor
- [x] Configure appropriate base URL

### 5. Update API Request Flow
- [x] Get token before API calls
- [x] Validate token presence when required
- [x] Add token to request headers with Bearer prefix
- [x] Handle OAuth enforcement flags

### 6. Add Provider Configuration
- [x] Add entry to `provider-config.ts`
- [x] Set `requiresAuth: true`
- [x] Set `hidden: true` (admin-only)
- [x] Implement `checkAuth` function
- [x] Reference auth component

### 7. Update Constants
- [x] Add provider to PROVIDERS constant
- [x] Update validation schemas

### 8. Add Tests
- [ ] Test OAuth device flow end-to-end
- [ ] Test polling logic and error handling
- [ ] Test token expiration and re-auth
- [ ] Test OAuth enforcement
- [ ] Test storage operations

## Error Handling

### Common Error Scenarios

1. **Device Code Expired**
   - Device code valid for limited time (typically 15 minutes)
   - Display error message and prompt to restart flow
   - Exit with error code

2. **Authorization Denied**
   - User denied authorization in browser
   - Display error message
   - Exit with error code

3. **Polling Timeout**
   - User didn't authorize within time limit
   - Stop polling and display timeout message
   - Prompt to restart flow

4. **Network Errors**
   - Handle connection failures during polling
   - Implement retry logic with backoff
   - Display clear error messages

5. **Token Expired**
   - Remove expired tokens from storage
   - Trigger re-authentication flow
   - Let application handle auth prompt

6. **OAuth Not Authorized**
   - API validates user is admin or pre-approved
   - Returns 403 with clear error message
   - CLI displays error to user

7. **Slow Down Response**
   - GitHub requests slower polling
   - Increase interval by 5 seconds
   - Continue polling with new interval

## Testing

### Manual Testing Flow
1. Run `cmd auth login`
2. Note the user code displayed
3. Verify browser opens to github.com/login/device
4. Enter user code in browser
5. Authorize application
6. Verify CLI detects authorization
7. Verify success message
8. Run `cmd auth status` to confirm
9. Test API request with OAuth
10. Wait for token expiry (8 hours) and test re-auth
11. Run `cmd auth logout` to clean up

## API Endpoints

### Device Code Request
```
POST https://github.com/login/device/code
Content-Type: application/json

{
  "client_id": "CLIENT_ID",
  "scope": "read:user"
}

Response:
{
  "device_code": "...",
  "user_code": "XXXX-XXXX",
  "verification_uri": "https://github.com/login/device",
  "expires_in": 900,
  "interval": 5
}
```

### Token Polling
```
POST https://github.com/login/oauth/access_token
Content-Type: application/json

{
  "client_id": "CLIENT_ID",
  "device_code": "DEVICE_CODE",
  "grant_type": "urn:ietf:params:oauth:grant-type:device_code"
}

Response (Success):
{
  "access_token": "...",
  "token_type": "bearer",
  "scope": "read:user"
}

Response (Pending):
{
  "error": "authorization_pending"
}

Response (Slow Down):
{
  "error": "slow_down"
}
```

### Copilot Token Exchange
```
GET https://api.github.com/copilot_internal/v2/token
Authorization: Bearer GITHUB_ACCESS_TOKEN
Accept: application/json

Response:
{
  "token": "...",
  "expires_at": 1234567890
}
```

## Usage Patterns

### Check Authentication Status
```typescript
const token = await GitHubCopilotOAuth.getValidAccessToken();
const isAuthenticated = !!token;
```

### Start Device Flow
```typescript
const {deviceCode, userCode, verificationUri, interval} = 
  await createDeviceCode();
console.log(`Go to ${verificationUri} and enter code: ${userCode}`);
const token = await pollForToken(deviceCode, interval);
```

### Exchange for Copilot Token
```typescript
const accessToken = await getValidAccessToken();
const copilotToken = await getCopilotToken(accessToken);
```

## Best Practices

1. **Use Device Flow** for CLI applications (no client secret needed)
2. **Respect polling intervals** to avoid rate limiting
3. **Handle all polling responses** (pending, slow_down, expired, denied)
4. **Store tokens securely** with restricted file permissions
5. **Clear error messages** for users during polling
6. **Timeout handling** with user-friendly messages
7. **Admin-only features** behind feature flags
8. **Server-side validation** of OAuth access
9. **Cross-platform support** for browser opening
10. **Graceful re-authentication** when tokens expire

## Key Differences from PKCE Flow

1. **Device Flow vs PKCE**: Optimized for devices without web browsers
2. **No Code Verifier**: Device flow doesn't use PKCE challenge
3. **Polling Required**: Client polls for authorization completion
4. **User Code Entry**: User manually enters code in browser
5. **No Refresh Tokens**: Typically requires full re-authentication
6. **Rate Limiting**: Must respect polling interval adjustments
7. **Timeout Handling**: Device codes expire (typically 15 minutes)

## References

- [OAuth 2.0 Device Authorization Grant (RFC 8628)](https://tools.ietf.org/html/rfc8628)
- [GitHub Device Flow Documentation](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#device-flow)
- [GitHub Copilot API Documentation](https://docs.github.com/en/copilot)

---

This specification provides all the necessary information to implement GitHub Copilot OAuth flow using the Device Authorization Grant. Follow the implementation checklist and use this as a reference alongside the Anthropic implementation.
