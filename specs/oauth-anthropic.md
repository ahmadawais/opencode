# Anthropic Claude OAuth Implementation Spec

## Overview

This document provides a comprehensive specification of how OAuth authentication for Anthropic Claude Code is implemented in the Command Code CLI. Use this as a reference to implement similar OAuth flows for other providers.

## Architecture Components

### 1. Core OAuth Module (`packages/command/src/auth/anthropic.ts`)

The central OAuth implementation using PKCE (Proof Key for Code Exchange) flow.

#### Constants

```typescript
const CLIENT_ID = '';
const AUTHORIZE_URL = 'https://claude.ai/oauth/authorize';
const TOKEN_URL = 'https://console.anthropic.com/v1/oauth/token';
const REDIRECT_URI = 'https://console.anthropic.com/oauth/code/callback';
```

#### OAuth Scopes
```
org:create_api_key user:profile user:inference
```

#### Core Functions

**1. `createAuthorizationUrl()`**
- Generates PKCE code verifier (32 random bytes, base64url encoded)
- Creates SHA256 code challenge from verifier
- Generates random state (32 bytes, base64url)
- Constructs authorization URL with parameters:
  - `code`: 'true'
  - `client_id`: CLIENT_ID
  - `response_type`: 'code'
  - `redirect_uri`: REDIRECT_URI
  - `scope`: OAuth scopes
  - `code_challenge`: SHA256 hash of verifier
  - `code_challenge_method`: 'S256'
  - `state`: Random state for CSRF protection
- Returns: `{url, verifier, state}`

**2. `exchangeCodeForTokens(code, verifier, state)`**
- Makes POST request to TOKEN_URL with:
  - `grant_type`: 'authorization_code'
  - `client_id`: CLIENT_ID
  - `code`: Authorization code from user
  - `redirect_uri`: REDIRECT_URI
  - `code_verifier`: PKCE verifier
  - `state`: State for validation
- Receives token response:
  ```typescript
  {
    access_token: string;
    refresh_token: string;
    expires_in: number; // seconds
    token_type: string;
  }
  ```
- Stores tokens using Auth.set() with:
  ```typescript
  {
    type: 'oauth',
    refresh: refresh_token,
    access: access_token,
    expires: Date.now() + expires_in * 1000
  }
  ```

**3. `refreshAccessToken()`**
- Checks if token is expired (with 5-minute buffer)
- If still valid, returns existing access token
- If expired, makes POST to TOKEN_URL with:
  - `grant_type`: 'refresh_token'
  - `client_id`: CLIENT_ID
  - `refresh_token`: Stored refresh token
- Updates stored tokens with new values
- Removes auth data if refresh fails
- Returns new access token or undefined

**4. `getValidAccessToken()`**
- Retrieves stored auth data
- Handles both OAuth and API key types
- For OAuth, calls refreshAccessToken()
- Returns valid access token or undefined

### 2. Auth Storage Module (`packages/command/src/auth/index.ts`)

Manages persistent storage of authentication credentials.

#### Storage Location
- Directory: `~/.commandcode/` (INFO.DIRECTORY_NAME)
- File: `models.json` (production), `models.dev.json` (dev), `models.staging.json` (staging)
- Permissions: 0o600 (owner read/write only)

#### Auth Data Types

```typescript
// OAuth authentication
type OAuth = {
  type: 'oauth';
  refresh: string;  // Refresh token
  access: string;   // Access token
  expires: number;  // Unix timestamp in milliseconds
};

// API Key authentication
type ApiKey = {
  type: 'api';
  key: string;
};

type Info = OAuth | ApiKey;
```

#### Storage Format
```json
{
  "anthropic": {
    "type": "oauth",
    "refresh": "refresh_token_here",
    "access": "access_token_here",
    "expires": 1234567890000
  }
}
```

#### Core Functions

- **`get(provider)`**: Retrieve auth data for provider
- **`set(provider, info)`**: Store auth data with secure permissions
- **`remove(provider)`**: Delete auth data, remove file if empty
- **`list()`**: Get all stored auth data

### 3. CLI Commands (`packages/command/src/commands/anthropic-auth.ts`)

Three authentication commands exposed via Commander.js.

#### `auth login`
1. Generate authorization URL using `createAuthorizationUrl()`
2. Display URL to user
3. Attempt to open browser automatically:
   - macOS: `open`
   - Windows: `start`
   - Linux: `xdg-open`
4. Prompt user to paste authorization code
5. Extract code (strip any `#` fragments)
6. Call `exchangeCodeForTokens(code, verifier, state)`
7. Display success message

#### `auth logout`
1. Check if authenticated
2. Call `Auth.remove('anthropic')`
3. Display success message

#### `auth status`
1. Call `Auth.get('anthropic')`
2. Display authentication status
3. For OAuth, show token expiry and refresh status

### 4. Interactive UI Component (`packages/command/src/components/anthropic-auth.tsx`)

React-based authentication flow using Ink.

#### Flow
1. Generate auth URL on component mount
2. Prompt user to open browser (Y/n)
3. Open browser on confirmation
4. Display masked input for authorization code
5. Handle paste detection (bracketed paste and clipboard)
6. Submit code and exchange for tokens
7. Update user config to set provider to Anthropic
8. Call success callback

#### Features
- Automatic browser opening
- Masked input display (password-like)
- Clipboard paste detection
- Error handling and display
- ESC key cancellation

### 5. API Client Integration (`packages/command/src/clients/anthropic/anthropic.ts`)

Anthropic SDK client initialization with OAuth support.

```typescript
// Initialize with OAuth token
const oauthToken = await AnthropicOAuth.getValidAccessToken();
if (oauthToken) {
  this.instance = new Anthropic({
    authToken: oauthToken,
    defaultHeaders: {
      'anthropic-beta': 'oauth-2025-04-20,web-fetch-2025-09-10',
    },
  });
}
```

### 6. API Request Flow

When making API requests to the Command Code backend:

#### CLI Side (`packages/command/src/chat/context-engine.ts`)

```typescript
let token: string | undefined = undefined;
if (provider === PROVIDERS.ANTHROPIC) {
  token = await AnthropicOAuth.getValidAccessToken();
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

Server validates OAuth access:

```typescript
async function validateOAuthAccess({
  c,
  ownerUserId,
  ownerOrgId,
  oAuthToken,
  internalFlagEnabled,
}) {
  // No OAuth token, no validation needed
  if (!oAuthToken) return;
  
  // Check if internal flag is enabled
  if (!internalFlagEnabled) {
    throw new ApiError({
      code: 'BAD_REQUEST',
      status: 400,
      message: 'Provider not available. Use Command Code provider.',
    });
  }
  
  // Check if user is admin or pre-approved
  const admin = await isUserAdmin({userId: ownerUserId});
  const preApproved = await isUserPreApproved({userId: ownerUserId});
  
  if (!admin && !preApproved) {
    throw new ApiError({
      code: 'FORBIDDEN',
      status: 403,
      message: 'Invalid flag provided',
    });
  }
}
```

## Security Features

### 1. PKCE (Proof Key for Code Exchange)
- Prevents authorization code interception attacks
- Uses SHA256 challenge method
- Code verifier never sent to authorization server

### 2. State Parameter
- Random 32-byte state for CSRF protection
- Validated during token exchange

### 3. Secure Storage
- File permissions: 0o600 (owner read/write only)
- Tokens stored locally in user's home directory
- Environment-specific files (dev/staging/prod)

### 4. Token Refresh
- Automatic token refresh with 5-minute buffer
- Failed refresh removes invalid tokens
- Graceful fallback to API key authentication

### 5. Access Control
- Internal flag (`--co`) required for OAuth usage
- Admin or pre-approved user validation on API side
- Header-based token transmission

## Provider Configuration

Located in `packages/command/src/utils/provider-config.ts`:

```typescript
{
  [PROVIDER.ANTHROPIC]: {
    name: 'Claude',
    description: 'Claude Pro/Max',
    requiresAuth: true,
    hidden: true, // Only visible with --co flag (admin only)
    authComponent: AnthropicAuth,
    checkAuth: async () => {
      const token = await AnthropicOAuth.getValidAccessToken();
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
  ANTHROPIC: 'anthropic',
} as const;
```

## Implementation Checklist for New Providers

### 1. Create OAuth Module
- [x] Create `packages/command/src/auth/<provider>.ts`
- [x] Implement PKCE flow with code verifier/challenge
- [x] Define OAuth constants (CLIENT_ID, URLs, scopes)
- [x] Implement `createAuthorizationUrl()`
- [x] Implement `exchangeCodeForTokens()`
- [x] Implement `refreshAccessToken()`
- [x] Implement `getValidAccessToken()`
- [x] Use Auth.set/get/remove for storage

### 2. Create CLI Command
- [x] Create `packages/command/src/commands/<provider>-auth.ts`
- [x] Implement `login`, `logout`, `status` subcommands
- [x] Add browser auto-open with cross-platform support
- [x] Handle authorization code input
- [x] Display clear status messages

### 3. Create UI Component
- [x] Create `packages/command/src/components/<provider>-auth.tsx`
- [x] Implement interactive auth flow
- [x] Add browser confirmation prompt
- [x] Handle authorization code input with paste detection
- [x] Add error handling and cancellation
- [x] Update user config on success

### 4. Integrate with Client
- [x] Update provider client initialization
- [x] Call `getValidAccessToken()` before creating client
- [x] Pass token to SDK constructor
- [x] Add appropriate headers for OAuth beta features

### 5. Update API Request Flow
- [x] Get token before API calls
- [x] Validate token presence when required
- [x] Add token to request headers with Bearer prefix
- [x] Handle OAuth enforcement flags

### 6. Add Provider Configuration
- [x] Add entry to `provider-config.ts`
- [x] Set `requiresAuth: true`
- [x] Set `hidden: true` if admin-only
- [x] Implement `checkAuth` function
- [x] Reference auth component

### 7. Update Constants
- [x] Add provider to PROVIDERS constant
- [x] Add any provider-specific headers
- [x] Update validation schemas

### 8. Add Tests
- [ ] Test OAuth flow end-to-end
- [ ] Test token refresh logic
- [ ] Test OAuth enforcement
- [ ] Test storage operations
- [ ] Test error handling

## Error Handling

### Common Error Scenarios

1. **No Authorization Code**
   - Handled in CLI command
   - Display error message and exit

2. **Token Exchange Failed**
   - HTTP error from token endpoint
   - Parse error response and display
   - Exit with error code

3. **Refresh Token Failed**
   - Remove invalid tokens from storage
   - Return undefined to trigger re-authentication
   - Let application handle auth prompt

4. **OAuth Not Authorized**
   - API validates user is admin or pre-approved
   - Returns 403 with clear error message
   - CLI displays error to user

5. **Missing Internal Flag**
   - API validates internal flag header
   - Returns 400 if flag not enabled
   - Prevents OAuth usage without authorization

## Testing

### Unit Tests
Located in `packages/command/src/chat/__tests__/oauth-enforcement.test.ts`

Key test scenarios:
- OAuth enforcement with missing token
- OAuth enforcement with valid token
- OAuth enforcement bypass with disable flag
- Non-OAuth provider (no enforcement)

### Manual Testing Flow
1. Run `cmd auth login`
2. Verify browser opens
3. Authorize in browser
4. Copy authorization code
5. Paste code in CLI
6. Verify success message
7. Run `cmd auth status` to confirm
8. Test API request with OAuth
9. Wait for token expiry and test refresh
10. Run `cmd auth logout` to clean up

## API Endpoints

### Token Exchange
```
POST https://console.anthropic.com/v1/oauth/token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "client_id": "CLIENT_ID",
  "code": "AUTH_CODE",
  "redirect_uri": "REDIRECT_URI",
  "code_verifier": "VERIFIER",
  "state": "STATE"
}

Response:
{
  "access_token": "...",
  "refresh_token": "...",
  "expires_in": 3600,
  "token_type": "Bearer"
}
```

### Token Refresh
```
POST https://console.anthropic.com/v1/oauth/token
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "client_id": "CLIENT_ID",
  "refresh_token": "REFRESH_TOKEN"
}

Response: (same as token exchange)
```

## Usage Patterns

### Check Authentication Status
```typescript
const token = await AnthropicOAuth.getValidAccessToken();
const isAuthenticated = !!token;
```

### Validate OAuth for Request
```typescript
let token: string | undefined = undefined;
if (provider === PROVIDERS.ANTHROPIC) {
  token = await AnthropicOAuth.getValidAccessToken();
  validateOAuthToken({token, provider});
}
```

### Add Token to Request Headers
```typescript
if (token) {
  headers[HEADERS.OAUTH_TOKEN] = `Bearer ${token}`;
}
```

## Best Practices

1. **Always use PKCE** for public clients (CLI)
2. **Store tokens securely** with restricted file permissions
3. **Implement automatic token refresh** with expiry buffer
4. **Graceful fallback** to alternative authentication
5. **Clear error messages** for users
6. **Admin-only features** behind feature flags
7. **Server-side validation** of OAuth access
8. **Cross-platform support** for browser opening
9. **Paste detection** for better UX
10. **Comprehensive error handling** at all levels

## Key Differences from Other OAuth Flows

1. **PKCE Required**: Public client without client secret
2. **Manual Code Entry**: User copies code from browser
3. **Local Storage**: Tokens stored in user's home directory
4. **Dual Auth Support**: OAuth and API key coexist
5. **Admin Restrictions**: OAuth gated behind internal flag
6. **Server-Side Enforcement**: API validates OAuth access

## References

- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [PKCE RFC 7636](https://tools.ietf.org/html/rfc7636)
- [Anthropic OAuth Documentation](https://docs.anthropic.com/en/api/oauth)

---

This specification provides all the necessary information to implement a similar OAuth flow for other providers. Follow the implementation checklist and use the Anthropic implementation as a reference.
