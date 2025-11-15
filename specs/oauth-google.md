# Google OAuth Implementation Spec

## Overview

This document provides a comprehensive specification of how OAuth authentication for Google (Vertex AI, Gemini) is implemented in the Command Code CLI. Google services use OAuth 2.0 for user authentication and service account credentials for programmatic access.

## Architecture Components

### 1. Core OAuth Module (`packages/command/src/auth/google.ts`)

The central OAuth implementation using OAuth 2.0 Authorization Code flow with PKCE.

#### Constants

```typescript
const CLIENT_ID = '';
const CLIENT_SECRET = ''; // Optional for public clients
const AUTHORIZE_URL = 'https://accounts.google.com/o/oauth2/v2/auth';
const TOKEN_URL = 'https://oauth2.googleapis.com/token';
const REDIRECT_URI = 'http://localhost:8080/oauth/callback';
const SCOPES = [
  'https://www.googleapis.com/auth/cloud-platform',
  'https://www.googleapis.com/auth/userinfo.email',
  'https://www.googleapis.com/auth/userinfo.profile',
];
```

#### OAuth Flow

Google uses OAuth 2.0 Authorization Code flow with PKCE for client applications.

#### Core Functions

**1. `createAuthorizationUrl()`**
- Generates PKCE code verifier (32 random bytes, base64url encoded)
- Creates SHA256 code challenge from verifier
- Generates random state (32 bytes, base64url)
- Constructs authorization URL with parameters:
  - `client_id`: CLIENT_ID
  - `response_type`: 'code'
  - `redirect_uri`: REDIRECT_URI
  - `scope`: Space-separated scopes
  - `code_challenge`: SHA256 hash of verifier
  - `code_challenge_method`: 'S256'
  - `state`: Random state for CSRF protection
  - `access_type`: 'offline' (to get refresh token)
  - `prompt`: 'consent' (to force consent screen)
- Returns: `{url, verifier, state}`

**2. `exchangeCodeForTokens(code, verifier, state)`**
- Makes POST request to TOKEN_URL with:
  - `grant_type`: 'authorization_code'
  - `client_id`: CLIENT_ID
  - `code`: Authorization code from user
  - `redirect_uri`: REDIRECT_URI
  - `code_verifier`: PKCE verifier
- Receives token response:
  ```typescript
  {
    access_token: string;
    refresh_token: string;
    expires_in: number; // seconds (typically 3600)
    token_type: string; // "Bearer"
    scope: string;
    id_token?: string; // JWT with user info
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
- Handles OAuth tokens
- Calls refreshAccessToken() if needed
- Returns valid access token or undefined

**5. `startLocalServer()`**
- Starts a temporary local HTTP server on port 8080
- Listens for OAuth callback with authorization code
- Extracts code from query parameters
- Returns authorization code
- Closes server after receiving code

### 2. Auth Storage Module (`packages/command/src/auth/index.ts`)

Manages persistent storage of authentication credentials.

#### Storage Location
- Directory: `~/.commandcode/`
- File: `models.json` (production), `models.dev.json` (dev)
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

type Info = OAuth;
```

#### Storage Format
```json
{
  "google": {
    "type": "oauth",
    "refresh": "1//refresh_token_here",
    "access": "ya29.access_token_here",
    "expires": 1234567890000
  }
}
```

### 3. CLI Commands (`packages/command/src/commands/google-auth.ts`)

Three authentication commands for Google OAuth.

#### `auth login`
1. Generate authorization URL using `createAuthorizationUrl()`
2. Start local callback server on port 8080
3. Display URL to user
4. Open browser automatically to authorization URL
5. Wait for OAuth callback with authorization code
6. Extract code from callback
7. Call `exchangeCodeForTokens(code, verifier, state)`
8. Stop local server
9. Display success message

#### `auth logout`
1. Check if authenticated
2. Optionally revoke token with Google
3. Call `Auth.remove('google')`
4. Display success message

#### `auth status`
1. Call `Auth.get('google')`
2. Display authentication status
3. Show token expiry time
4. Show refresh token availability

### 4. Interactive UI Component (`packages/command/src/components/google-auth.tsx`)

React-based authentication flow using Ink.

#### Flow
1. Start local callback server
2. Generate auth URL on component mount
3. Display message about opening browser
4. Open browser automatically
5. Show "Waiting for authorization..." spinner
6. Wait for callback with code
7. Exchange code for tokens
8. Update user config to set provider to Google
9. Call success callback

#### Features
- Automatic local server management
- Browser auto-open
- Waiting indicator during authorization
- Error handling for server issues
- Timeout handling (5 minutes)
- ESC key cancellation

### 5. API Client Integration (`packages/command/src/clients/google/google.ts`)

Google Vertex AI SDK client initialization with OAuth support.

```typescript
// Initialize with OAuth token
const oauthToken = await GoogleOAuth.getValidAccessToken();
if (oauthToken) {
  const project = process.env.GOOGLE_CLOUD_PROJECT || 'default-project';
  const location = process.env.GOOGLE_CLOUD_LOCATION || 'us-central1';
  
  this.instance = new VertexAI({
    project,
    location,
    authToken: oauthToken,
  });
}
```

### 6. API Request Flow

#### CLI Side

```typescript
let token: string | undefined = undefined;
if (provider === PROVIDERS.GOOGLE) {
  token = await GoogleOAuth.getValidAccessToken();
  validateOAuthToken({token, provider});
}

const headers: Record<string, string> = {
  [HEADERS.INTERNAL_FLAG_HEADER]: oauthEnforced.toString(),
};

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
- Random 32-byte state for CSRF protection
- Validated during token exchange

### 3. Secure Storage
- File permissions: 0o600
- Tokens stored in user home directory
- Environment-specific files

### 4. Token Refresh
- Automatic refresh with 5-minute buffer
- Google refresh tokens don't expire by default
- Failed refresh removes invalid tokens

### 5. Local Callback Server
- Temporary server for OAuth callback
- Automatically closed after receiving code
- Timeout after 5 minutes
- Only accepts requests from localhost

### 6. Scope Restrictions
- Request only necessary scopes
- User explicitly grants permissions
- Scope displayed during consent

## Provider Configuration

```typescript
{
  [PROVIDER.GOOGLE]: {
    name: 'Google',
    description: 'Google Vertex AI / Gemini',
    requiresAuth: true,
    hidden: false,
    authComponent: GoogleAuth,
    checkAuth: async () => {
      const token = await GoogleOAuth.getValidAccessToken();
      return !!token;
    },
  },
}
```

## Constants & Configuration

```typescript
export const PROVIDERS = {
  GOOGLE: 'google',
  GOOGLE_VERTEX: 'google-vertex',
} as const;
```

## Implementation Checklist

- [ ] Create `packages/command/src/auth/google.ts`
- [ ] Implement PKCE flow
- [ ] Implement `createAuthorizationUrl()`
- [ ] Implement `startLocalServer()` for callback
- [ ] Implement `exchangeCodeForTokens()`
- [ ] Implement `refreshAccessToken()`
- [ ] Implement `getValidAccessToken()`
- [ ] Create `packages/command/src/commands/google-auth.ts`
- [ ] Implement `login`, `logout`, `status` commands
- [ ] Handle local server lifecycle
- [ ] Create `packages/command/src/components/google-auth.tsx`
- [ ] Implement OAuth flow with local server
- [ ] Add timeout handling
- [ ] Integrate with Google Vertex AI client
- [ ] Update API request flow
- [ ] Add provider configuration
- [ ] Add tests

## Error Handling

1. **Local Server Failed to Start**
   - Port 8080 already in use
   - Display error with alternative port suggestion
   - Suggest manual code entry fallback

2. **Authorization Timeout**
   - User didn't authorize within 5 minutes
   - Stop local server
   - Display timeout message

3. **Token Exchange Failed**
   - Invalid code or expired
   - Display error from Google
   - Prompt to retry

4. **Refresh Token Failed**
   - Token revoked or expired
   - Remove stored tokens
   - Trigger re-authentication

5. **Network Errors**
   - Connection failures
   - Display clear error messages
   - Suggest checking internet connection

## API Endpoints

### Authorization URL
```
GET https://accounts.google.com/o/oauth2/v2/auth
Parameters:
  - client_id: CLIENT_ID
  - response_type: code
  - redirect_uri: http://localhost:8080/oauth/callback
  - scope: space-separated scopes
  - code_challenge: SHA256(verifier)
  - code_challenge_method: S256
  - state: RANDOM_STATE
  - access_type: offline
  - prompt: consent
```

### Token Exchange
```
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=CLIENT_ID
&code=AUTH_CODE
&redirect_uri=REDIRECT_URI
&code_verifier=VERIFIER

Response:
{
  "access_token": "ya29...",
  "refresh_token": "1//...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "scope": "...",
  "id_token": "eyJ..."
}
```

### Token Refresh
```
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&client_id=CLIENT_ID
&refresh_token=REFRESH_TOKEN

Response: (same format as token exchange)
```

### Token Revocation (Optional)
```
POST https://oauth2.googleapis.com/revoke
Content-Type: application/x-www-form-urlencoded

token=ACCESS_TOKEN_OR_REFRESH_TOKEN
```

## Usage Patterns

### Check Authentication Status
```typescript
const token = await GoogleOAuth.getValidAccessToken();
const isAuthenticated = !!token;
```

### Start OAuth Flow with Local Server
```typescript
const server = await startLocalServer();
const {url, verifier, state} = await createAuthorizationUrl();
console.log(`Visit: ${url}`);
openBrowser(url);

const code = await server.waitForCallback();
const tokens = await exchangeCodeForTokens(code, verifier, state);
server.close();
```

## Best Practices

1. **Use PKCE** for all OAuth flows
2. **Local callback server** for better UX (no manual code copy)
3. **Automatic browser opening** when possible
4. **Timeout handling** for authorization flow
5. **Secure token storage** with 0o600 permissions
6. **Automatic token refresh** with buffer
7. **Scope minimization** - request only needed scopes
8. **Clear error messages** for users
9. **Graceful server shutdown** after callback
10. **Cross-platform support** for browser opening

## Key Differences from Other Flows

1. **Local Callback Server**: Uses localhost server instead of manual code entry
2. **Offline Access**: Requests `access_type=offline` for refresh token
3. **Consent Prompt**: Forces consent screen to ensure refresh token
4. **Longer Refresh Tokens**: Google refresh tokens typically don't expire
5. **Scope URLs**: Google uses full URL scopes (not simple strings)
6. **ID Token**: Optionally returns JWT with user information

## Special Considerations

### Service Accounts vs User OAuth

Google supports two authentication methods:
- **OAuth** (this spec): For user authentication, requires browser
- **Service Accounts**: For server/CI, uses JSON key files

This spec covers user OAuth. For service accounts:
```typescript
// Service account alternative
const auth = new GoogleAuth({
  keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
  scopes: SCOPES,
});
```

### Project and Location Configuration

Google Vertex AI requires:
```typescript
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1
```

These should be set as environment variables or prompted during setup.

## References

- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [PKCE RFC 7636](https://tools.ietf.org/html/rfc7636)
- [Google OAuth 2.0 Documentation](https://developers.google.com/identity/protocols/oauth2)
- [Google Cloud Authentication](https://cloud.google.com/docs/authentication)
- [Vertex AI Documentation](https://cloud.google.com/vertex-ai/docs)

---

This specification provides the framework for implementing Google OAuth authentication with local callback server for improved user experience. The local server approach eliminates manual code copying while maintaining security through PKCE.
