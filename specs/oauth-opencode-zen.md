# OpenCode Zen OAuth Implementation Spec

## Overview

This document provides a comprehensive specification of how OAuth authentication for OpenCode Zen is implemented in the Command Code CLI. OpenCode Zen is the managed service provided by the OpenCode team with tested and verified models. Use this as a reference to implement the OAuth flow for the OpenCode Zen provider.

## Architecture Components

### 1. Core OAuth Module (`packages/command/src/auth/opencode.ts`)

The central OAuth implementation using PKCE (Proof Key for Code Exchange) flow.

#### Constants

```typescript
const CLIENT_ID = '';
const AUTHORIZE_URL = 'https://opencode.ai/oauth/authorize';
const TOKEN_URL = 'https://opencode.ai/api/oauth/token';
const REDIRECT_URI = 'https://opencode.ai/oauth/callback';
const API_BASE_URL = 'https://opencode.ai/api';
```

#### OAuth Scopes
```
api:access user:profile
```

#### Core Functions

**1. `createAuthorizationUrl()`**
- Generates PKCE code verifier (32 random bytes, base64url encoded)
- Creates SHA256 code challenge from verifier
- Generates random state (32 bytes, base64url)
- Constructs authorization URL with parameters:
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

**5. `getApiKey()`**
- Helper function to get API key if user prefers API key auth
- Retrieves stored API key from auth storage
- Returns API key string or undefined

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
  "opencode": {
    "type": "oauth",
    "refresh": "refresh_token_here",
    "access": "access_token_here",
    "expires": 1234567890000
  }
}
```

Or for API key authentication:

```json
{
  "opencode": {
    "type": "api",
    "key": "oc_xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  }
}
```

#### Core Functions

- **`get(provider)`**: Retrieve auth data for provider
- **`set(provider, info)`**: Store auth data with secure permissions
- **`remove(provider)`**: Delete auth data, remove file if empty
- **`list()`**: Get all stored auth data

### 3. CLI Commands (`packages/command/src/commands/opencode-auth.ts`)

Three authentication commands exposed via Commander.js, with support for both OAuth and API key methods.

#### `auth login`
Provides two authentication options:

**Option 1: OAuth Flow**
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

**Option 2: API Key**
1. Display instructions to visit https://opencode.ai/auth
2. Prompt user to create an API key
3. Prompt user to paste API key
4. Store API key using Auth.set()
5. Display success message

#### `auth logout`
1. Check if authenticated
2. Call `Auth.remove('opencode')`
3. Display success message

#### `auth status`
1. Call `Auth.get('opencode')`
2. Display authentication status
3. For OAuth, show token expiry and refresh status
4. For API key, show masked key (first 10 chars)

### 4. Interactive UI Component (`packages/command/src/components/opencode-auth.tsx`)

React-based authentication flow using Ink with support for both auth methods.

#### Flow
1. Display auth method selection (OAuth or API Key)
2. **For OAuth:**
   - Generate auth URL on method selection
   - Prompt user to open browser (Y/n)
   - Open browser on confirmation
   - Display masked input for authorization code
   - Handle paste detection (bracketed paste and clipboard)
   - Submit code and exchange for tokens
3. **For API Key:**
   - Display instructions to visit https://opencode.ai/auth
   - Prompt for API key with masked input
   - Validate API key format
   - Store API key
4. Update user config to set provider to OpenCode Zen
5. Call success callback

#### Features
- Auth method selection (OAuth vs API Key)
- Automatic browser opening for OAuth
- Masked input display for both auth code and API key
- Clipboard paste detection
- API key format validation
- Error handling and display
- ESC key cancellation

### 5. API Client Integration (`packages/command/src/clients/opencode/opencode.ts`)

OpenCode SDK client initialization with OAuth support.

```typescript
// Try OAuth first, fallback to API key
let authToken: string | undefined;

const oauthToken = await OpencodeOAuth.getValidAccessToken();
if (oauthToken) {
  authToken = oauthToken;
} else {
  const apiKey = await OpencodeOAuth.getApiKey();
  if (apiKey) {
    authToken = apiKey;
  }
}

if (authToken) {
  this.instance = new OpencodeClient({
    authToken,
    baseURL: API_BASE_URL,
    defaultHeaders: {
      'X-Client-Version': packageVersion,
    },
  });
}
```

### 6. API Request Flow

When making API requests to the Command Code backend:

#### CLI Side (`packages/command/src/chat/context-engine.ts`)

```typescript
let token: string | undefined = undefined;
if (provider === PROVIDERS.OPENCODE) {
  token = await OpencodeOAuth.getValidAccessToken();
  // If no OAuth token, try API key
  if (!token) {
    token = await OpencodeOAuth.getApiKey();
  }
  validateOAuthToken({token, provider});
}

const headers: Record<string, string> = {
  [HEADERS.INTERNAL_FLAG_HEADER]: oauthEnforced.toString(),
  // ... other headers
};

if (token) {
  // Token could be OAuth access token or API key
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
  
  // Validate token with OpenCode API
  const tokenValid = await validateOpencodeToken(oAuthToken);
  if (!tokenValid) {
    throw new ApiError({
      code: 'UNAUTHORIZED',
      status: 401,
      message: 'Invalid OpenCode token',
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

### 5. Dual Authentication Support
- OAuth for enhanced security and user experience
- API key as fallback for simpler setup
- Both methods stored securely

### 6. Access Control
- Internal flag (`--co`) required for OAuth usage
- Token validation on OpenCode API side
- Header-based token transmission

## Provider Configuration

Located in `packages/command/src/utils/provider-config.ts`:

```typescript
{
  [PROVIDER.OPENCODE]: {
    name: 'OpenCode Zen',
    description: 'Managed models by OpenCode team',
    requiresAuth: true,
    hidden: false, // Publicly available
    authComponent: OpencodeAuth,
    checkAuth: async () => {
      const token = await OpencodeOAuth.getValidAccessToken();
      if (token) return true;
      const apiKey = await OpencodeOAuth.getApiKey();
      return !!apiKey;
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
  OPENCODE: 'opencode',
} as const;
```

## Implementation Checklist for OpenCode Zen

### 1. Create OAuth Module
- [x] Create `packages/command/src/auth/opencode.ts`
- [x] Implement PKCE flow with code verifier/challenge
- [x] Define OAuth constants (CLIENT_ID, URLs, scopes)
- [x] Implement `createAuthorizationUrl()`
- [x] Implement `exchangeCodeForTokens()`
- [x] Implement `refreshAccessToken()`
- [x] Implement `getValidAccessToken()`
- [x] Implement `getApiKey()` helper
- [x] Use Auth.set/get/remove for storage

### 2. Create CLI Command
- [x] Create `packages/command/src/commands/opencode-auth.ts`
- [x] Implement `login`, `logout`, `status` subcommands
- [x] Support both OAuth and API key methods in login
- [x] Add browser auto-open with cross-platform support
- [x] Handle authorization code input
- [x] Handle API key input with validation
- [x] Display clear status messages for both auth types

### 3. Create UI Component
- [x] Create `packages/command/src/components/opencode-auth.tsx`
- [x] Implement auth method selection
- [x] Implement OAuth flow
- [x] Implement API key flow
- [x] Add browser confirmation prompt
- [x] Handle authorization code input with paste detection
- [x] Handle API key input with validation
- [x] Add error handling and cancellation
- [x] Update user config on success

### 4. Integrate with Client
- [x] Update provider client initialization
- [x] Try OAuth first, fallback to API key
- [x] Call `getValidAccessToken()` before creating client
- [x] Call `getApiKey()` if OAuth not available
- [x] Pass token to SDK constructor
- [x] Add appropriate headers

### 5. Update API Request Flow
- [x] Get OAuth token before API calls
- [x] Fallback to API key if OAuth not available
- [x] Validate token presence when required
- [x] Add token to request headers with Bearer prefix
- [x] Handle OAuth enforcement flags

### 6. Add Provider Configuration
- [x] Add entry to `provider-config.ts`
- [x] Set `requiresAuth: true`
- [x] Set `hidden: false` (public provider)
- [x] Implement `checkAuth` function for both auth types
- [x] Reference auth component

### 7. Update Constants
- [x] Add provider to PROVIDERS constant
- [x] Update validation schemas

### 8. Add Tests
- [ ] Test OAuth flow end-to-end
- [ ] Test API key flow
- [ ] Test token refresh logic
- [ ] Test fallback from OAuth to API key
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
   - Try API key if available
   - Let application handle auth prompt

4. **Invalid API Key**
   - Validate API key format (oc_*)
   - Display error message
   - Prompt to re-enter

5. **OAuth Not Authorized**
   - API validates token with OpenCode service
   - Returns 401/403 with clear error message
   - CLI displays error to user
   - Suggests trying API key method

6. **Network Errors**
   - Handle connection failures
   - Display clear error messages
   - Suggest checking internet connection

## Testing

### Unit Tests
Located in `packages/command/src/chat/__tests__/oauth-enforcement.test.ts`

Key test scenarios:
- OAuth enforcement with missing token
- OAuth enforcement with valid token
- OAuth enforcement bypass with disable flag
- API key fallback when OAuth fails
- Token refresh logic
- Storage operations

### Manual Testing Flow

**OAuth Flow:**
1. Run `cmd auth login`
2. Select OpenCode Zen provider
3. Choose OAuth method
4. Verify browser opens to opencode.ai
5. Authorize in browser
6. Copy authorization code
7. Paste code in CLI
8. Verify success message
9. Run `cmd auth status` to confirm
10. Test API request with OAuth
11. Wait for token expiry and test refresh
12. Run `cmd auth logout` to clean up

**API Key Flow:**
1. Run `cmd auth login`
2. Select OpenCode Zen provider
3. Choose API Key method
4. Visit https://opencode.ai/auth
5. Create API key
6. Paste API key in CLI
7. Verify success message
8. Run `cmd auth status` to confirm
9. Test API request with API key
10. Run `cmd auth logout` to clean up

## API Endpoints

### Authorization URL
```
GET https://opencode.ai/oauth/authorize
Parameters:
  - client_id: CLIENT_ID
  - response_type: code
  - redirect_uri: REDIRECT_URI
  - scope: api:access user:profile
  - code_challenge: SHA256(verifier)
  - code_challenge_method: S256
  - state: RANDOM_STATE
```

### Token Exchange
```
POST https://opencode.ai/api/oauth/token
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
POST https://opencode.ai/api/oauth/token
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "client_id": "CLIENT_ID",
  "refresh_token": "REFRESH_TOKEN"
}

Response: (same as token exchange)
```

### API Key Creation
```
Web UI: https://opencode.ai/auth
User creates API key with format: oc_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Usage Patterns

### Check Authentication Status
```typescript
const oauthToken = await OpencodeOAuth.getValidAccessToken();
const apiKey = await OpencodeOAuth.getApiKey();
const isAuthenticated = !!(oauthToken || apiKey);
```

### Get Auth Token (OAuth or API Key)
```typescript
let token = await OpencodeOAuth.getValidAccessToken();
if (!token) {
  token = await OpencodeOAuth.getApiKey();
}
```

### Add Token to Request Headers
```typescript
if (token) {
  headers[HEADERS.OAUTH_TOKEN] = `Bearer ${token}`;
}
```

## Best Practices

1. **Support both auth methods** for flexibility
2. **Try OAuth first** for better UX and security
3. **Fallback to API key** for simpler setup
4. **Always use PKCE** for OAuth flows
5. **Store tokens securely** with restricted file permissions
6. **Implement automatic token refresh** with expiry buffer
7. **Graceful fallback** to alternative authentication
8. **Clear error messages** for users
9. **Validate API keys** before storing
10. **Server-side validation** of both OAuth and API keys

## Key Differences from Anthropic Implementation

1. **Dual Authentication**: Supports both OAuth and API key
2. **Public Provider**: Not hidden behind admin flags
3. **OpenCode-Specific**: Managed by OpenCode team
4. **API Key Format**: Uses `oc_*` prefix
5. **Simpler Setup**: API key option for quick start
6. **Fallback Logic**: OAuth preferred, API key as fallback

## References

- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [PKCE RFC 7636](https://tools.ietf.org/html/rfc7636)
- [OpenCode Documentation](https://opencode.ai/docs)
- [OpenCode API Documentation](https://opencode.ai/docs/api)

---

This specification provides all the necessary information to implement OAuth flow for OpenCode Zen with API key fallback. Follow the implementation checklist and use the Anthropic implementation as an additional reference for the OAuth flow.
