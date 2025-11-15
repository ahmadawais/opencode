# OpenAI OAuth Implementation Spec

## Overview

This document provides a comprehensive specification of how OAuth authentication for OpenAI could be implemented in the Command Code CLI. While OpenAI currently primarily uses API keys, this spec outlines how OAuth could be integrated for enhanced security and enterprise SSO scenarios.

## Architecture Components

### 1. Core OAuth Module (`packages/command/src/auth/openai.ts`)

The central OAuth implementation using PKCE (Proof Key for Code Exchange) flow.

#### Constants

```typescript
const CLIENT_ID = '';
const AUTHORIZE_URL = 'https://auth.openai.com/oauth/authorize';
const TOKEN_URL = 'https://auth.openai.com/oauth/token';
const REDIRECT_URI = 'https://platform.openai.com/oauth/callback';
const API_BASE_URL = 'https://api.openai.com/v1';
```

#### OAuth Scopes
```
api.read api.write user.profile
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
    scope: string;
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
  refresh: string;
  access: string;
  expires: number;
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
  "openai": {
    "type": "oauth",
    "refresh": "refresh_token_here",
    "access": "access_token_here",
    "expires": 1234567890000
  }
}
```

### 3. CLI Commands (`packages/command/src/commands/openai-auth.ts`)

Three authentication commands with support for both OAuth and API key methods.

#### `auth login`
Provides two authentication options:

**Option 1: OAuth Flow (Enterprise/SSO)**
1. Generate authorization URL using `createAuthorizationUrl()`
2. Display URL to user
3. Open browser automatically
4. Prompt user to paste authorization code
5. Call `exchangeCodeForTokens(code, verifier, state)`
6. Display success message

**Option 2: API Key (Standard)**
1. Display instructions to visit https://platform.openai.com/api-keys
2. Prompt user to create an API key
3. Prompt user to paste API key
4. Store API key using Auth.set()
5. Display success message

#### `auth logout`
1. Check if authenticated
2. Call `Auth.remove('openai')`
3. Display success message

#### `auth status`
1. Call `Auth.get('openai')`
2. Display authentication status
3. For OAuth, show token expiry and refresh status
4. For API key, show masked key

### 4. Interactive UI Component (`packages/command/src/components/openai-auth.tsx`)

React-based authentication flow using Ink with support for both auth methods.

#### Flow
1. Display auth method selection (OAuth/SSO or API Key)
2. **For OAuth:**
   - Generate auth URL
   - Open browser
   - Prompt for authorization code
   - Exchange for tokens
3. **For API Key:**
   - Display platform.openai.com instructions
   - Prompt for API key with validation
   - Store API key
4. Update user config
5. Call success callback

### 5. API Client Integration (`packages/command/src/clients/openai/openai.ts`)

OpenAI SDK client initialization with OAuth support.

```typescript
// Try OAuth first, fallback to API key
let authToken: string | undefined;

const oauthToken = await OpenAIOAuth.getValidAccessToken();
if (oauthToken) {
  authToken = oauthToken;
} else {
  const apiKey = await OpenAIOAuth.getApiKey();
  if (apiKey) {
    authToken = apiKey;
  }
}

if (authToken) {
  this.instance = new OpenAI({
    apiKey: authToken,
    baseURL: API_BASE_URL,
  });
}
```

### 6. API Request Flow

#### CLI Side
```typescript
let token: string | undefined = undefined;
if (provider === PROVIDERS.OPENAI) {
  token = await OpenAIOAuth.getValidAccessToken();
  if (!token) {
    token = await OpenAIOAuth.getApiKey();
  }
  validateOAuthToken({token, provider});
}

if (token) {
  headers[HEADERS.OAUTH_TOKEN] = `Bearer ${token}`;
}
```

## Security Features

### 1. PKCE (Proof Key for Code Exchange)
- Prevents authorization code interception
- SHA256 challenge method
- Code verifier never sent to authorization server

### 2. State Parameter
- 32-byte random state for CSRF protection
- Validated during token exchange

### 3. Secure Storage
- File permissions: 0o600
- Local storage in user home directory
- Environment-specific files

### 4. Token Refresh
- Automatic refresh with 5-minute buffer
- Failed refresh removes invalid tokens
- Graceful fallback to API key

### 5. Dual Authentication Support
- OAuth for enterprise/SSO scenarios
- API key for standard usage
- Both methods stored securely

## Provider Configuration

```typescript
{
  [PROVIDER.OPENAI]: {
    name: 'OpenAI',
    description: 'OpenAI Platform',
    requiresAuth: true,
    hidden: false,
    authComponent: OpenAIAuth,
    checkAuth: async () => {
      const token = await OpenAIOAuth.getValidAccessToken();
      if (token) return true;
      const apiKey = await OpenAIOAuth.getApiKey();
      return !!apiKey;
    },
  },
}
```

## Constants & Configuration

```typescript
export const PROVIDERS = {
  OPENAI: 'openai',
} as const;
```

## Implementation Checklist

- [ ] Create `packages/command/src/auth/openai.ts`
- [ ] Implement PKCE flow
- [ ] Implement `createAuthorizationUrl()`
- [ ] Implement `exchangeCodeForTokens()`
- [ ] Implement `refreshAccessToken()`
- [ ] Implement `getValidAccessToken()`
- [ ] Implement `getApiKey()` helper
- [ ] Create `packages/command/src/commands/openai-auth.ts`
- [ ] Implement `login`, `logout`, `status` commands
- [ ] Support both OAuth and API key in login
- [ ] Create `packages/command/src/components/openai-auth.tsx`
- [ ] Implement auth method selection
- [ ] Implement OAuth flow
- [ ] Implement API key flow
- [ ] Integrate with OpenAI client
- [ ] Update API request flow
- [ ] Add provider configuration
- [ ] Add tests

## Error Handling

1. **No Authorization Code** - Display error and exit
2. **Token Exchange Failed** - Parse error response and display
3. **Refresh Token Failed** - Remove tokens, trigger re-auth
4. **Invalid API Key** - Validate format and display error
5. **Network Errors** - Clear error messages

## API Endpoints

### Authorization
```
GET https://auth.openai.com/oauth/authorize
Parameters: client_id, response_type, redirect_uri, scope, code_challenge, code_challenge_method, state
```

### Token Exchange
```
POST https://auth.openai.com/oauth/token
{
  "grant_type": "authorization_code",
  "client_id": "CLIENT_ID",
  "code": "CODE",
  "redirect_uri": "REDIRECT_URI",
  "code_verifier": "VERIFIER",
  "state": "STATE"
}
```

### Token Refresh
```
POST https://auth.openai.com/oauth/token
{
  "grant_type": "refresh_token",
  "client_id": "CLIENT_ID",
  "refresh_token": "REFRESH_TOKEN"
}
```

## Usage Patterns

```typescript
// Check authentication
const oauthToken = await OpenAIOAuth.getValidAccessToken();
const apiKey = await OpenAIOAuth.getApiKey();
const isAuthenticated = !!(oauthToken || apiKey);

// Get auth token (OAuth or API key)
let token = await OpenAIOAuth.getValidAccessToken();
if (!token) {
  token = await OpenAIOAuth.getApiKey();
}
```

## Best Practices

1. Support both OAuth and API key methods
2. Try OAuth first for enterprise scenarios
3. Fallback to API key for simplicity
4. Use PKCE for all OAuth flows
5. Store tokens with 0o600 permissions
6. Automatic token refresh
7. Clear error messages
8. Validate API keys before storing

## Key Features

- **Dual Authentication**: OAuth for enterprise, API key for standard
- **PKCE Flow**: Secure OAuth implementation
- **Token Refresh**: Automatic with 5-minute buffer
- **API Key Fallback**: Simpler alternative auth method

## References

- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [PKCE RFC 7636](https://tools.ietf.org/html/rfc7636)
- [OpenAI API Documentation](https://platform.openai.com/docs)

---

This specification provides the framework for implementing OAuth authentication for OpenAI. While OpenAI currently uses API keys primarily, this spec enables OAuth integration for enterprise SSO scenarios.
