# Hugging Face OAuth Implementation Spec

## Overview

This document provides a comprehensive specification of how OAuth authentication for Hugging Face could be implemented in the Command Code CLI. Hugging Face supports both user access tokens and OAuth for application access.

## Architecture Components

### 1. Core OAuth Module (`packages/command/src/auth/huggingface.ts`)

The central OAuth implementation using OAuth 2.0 with PKCE.

#### Constants

```typescript
const CLIENT_ID = '';
const AUTHORIZE_URL = 'https://huggingface.co/oauth/authorize';
const TOKEN_URL = 'https://huggingface.co/oauth/token';
const REDIRECT_URI = 'http://localhost:8080/oauth/callback';
const API_BASE_URL = 'https://api-inference.huggingface.co';
const SCOPES = ['inference-api', 'profile'];
```

#### OAuth Flow

Hugging Face uses OAuth 2.0 Authorization Code flow with PKCE.

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
  - `state`: Random state
- Returns: `{url, verifier, state}`

**2. `exchangeCodeForTokens(code, verifier, state)`**
- Makes POST request to TOKEN_URL with:
  - `grant_type`: 'authorization_code'
  - `client_id`: CLIENT_ID
  - `code`: Authorization code
  - `redirect_uri`: REDIRECT_URI
  - `code_verifier`: PKCE verifier
- Receives token response:
  ```typescript
  {
    access_token: string;
    refresh_token: string;
    expires_in: number;
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
- If expired, makes POST to TOKEN_URL with:
  - `grant_type`: 'refresh_token'
  - `client_id`: CLIENT_ID
  - `refresh_token`: Stored refresh token
- Updates stored tokens
- Returns new access token or undefined

**4. `getValidAccessToken()`**
- Retrieves stored auth data
- Handles both OAuth and user access tokens
- For OAuth, calls refreshAccessToken()
- Returns valid access token or undefined

**5. `getUserAccessToken()`**
- Helper to get user access token (simpler alternative to OAuth)
- User creates token at https://huggingface.co/settings/tokens
- Returns stored user access token or undefined

**6. `startLocalServer()`**
- Starts temporary HTTP server on port 8080
- Listens for OAuth callback
- Extracts authorization code
- Closes server after receiving code

### 2. Auth Storage Module

#### Storage Format
```json
{
  "huggingface": {
    "type": "oauth",
    "refresh": "refresh_token_here",
    "access": "access_token_here",
    "expires": 1234567890000
  }
}
```

Or for user access token:
```json
{
  "huggingface": {
    "type": "api",
    "key": "hf_xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  }
}
```

### 3. CLI Commands (`packages/command/src/commands/huggingface-auth.ts`)

#### `auth login`
Provides two authentication options:

**Option 1: OAuth Flow**
1. Start local callback server
2. Generate authorization URL
3. Open browser to authorization URL
4. Wait for callback with code
5. Exchange code for tokens
6. Display success message

**Option 2: User Access Token**
1. Display instructions to visit https://huggingface.co/settings/tokens
2. Prompt user to create token with 'inference-api' permission
3. Prompt user to paste token
4. Validate token format (hf_*)
5. Store token
6. Display success message

#### `auth logout`
1. Check if authenticated
2. Optionally revoke OAuth token
3. Call `Auth.remove('huggingface')`
4. Display success message

#### `auth status`
1. Retrieve stored auth data
2. Display authentication status
3. For OAuth, show token expiry
4. For user token, show masked token

### 4. Interactive UI Component (`packages/command/src/components/huggingface-auth.tsx`)

#### Flow
1. Display auth method selection (OAuth or User Token)
2. **For OAuth:**
   - Start local callback server
   - Generate and display auth URL
   - Open browser automatically
   - Wait for callback
   - Exchange code for tokens
3. **For User Token:**
   - Display token creation instructions
   - Prompt for token with validation
   - Store token
4. Update config
5. Call success callback

### 5. API Client Integration (`packages/command/src/clients/huggingface/huggingface.ts`)

```typescript
// Try OAuth first, fallback to user token
let authToken: string | undefined;

const oauthToken = await HuggingFaceOAuth.getValidAccessToken();
if (oauthToken) {
  authToken = oauthToken;
} else {
  const userToken = await HuggingFaceOAuth.getUserAccessToken();
  if (userToken) {
    authToken = userToken;
  }
}

if (authToken) {
  this.instance = new HfInference(authToken, {
    baseURL: API_BASE_URL,
  });
}
```

## Security Features

### 1. PKCE
- Prevents code interception
- SHA256 challenge method
- Secure for public clients

### 2. Token Scopes
- `inference-api`: Access to Inference API
- `profile`: User profile information
- Fine-grained permissions

### 3. Dual Authentication
- OAuth for applications
- User access tokens for simplicity
- Both stored securely

### 4. Token Management
- Automatic refresh with buffer
- User tokens don't expire
- Revocation support

### 5. Local Callback Server
- Automatic code capture
- No manual copying
- Timeout handling

## Provider Configuration

```typescript
{
  [PROVIDER.HUGGINGFACE]: {
    name: 'Hugging Face',
    description: 'Hugging Face Inference API',
    requiresAuth: true,
    hidden: false,
    authComponent: HuggingFaceAuth,
    checkAuth: async () => {
      const token = await HuggingFaceOAuth.getValidAccessToken();
      if (token) return true;
      const userToken = await HuggingFaceOAuth.getUserAccessToken();
      return !!userToken;
    },
  },
}
```

## Implementation Checklist

- [ ] Create `packages/command/src/auth/huggingface.ts`
- [ ] Implement PKCE flow
- [ ] Implement local callback server
- [ ] Implement token exchange
- [ ] Implement token refresh
- [ ] Support user access tokens
- [ ] Create CLI commands
- [ ] Create UI component
- [ ] Integrate with Hugging Face client
- [ ] Add token validation
- [ ] Add tests

## Error Handling

1. **Invalid Token Format** - Validate hf_* prefix
2. **Token Exchange Failed** - Display Hugging Face error
3. **Insufficient Permissions** - Guide to token settings
4. **Network Errors** - Clear error messages
5. **Server Start Failed** - Suggest manual token entry

## API Endpoints

### Authorization
```
GET https://huggingface.co/oauth/authorize
Parameters: client_id, response_type, redirect_uri, scope, code_challenge, code_challenge_method, state
```

### Token Exchange
```
POST https://huggingface.co/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=CLIENT_ID
&code=CODE
&redirect_uri=REDIRECT_URI
&code_verifier=VERIFIER
```

### Token Refresh
```
POST https://huggingface.co/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&client_id=CLIENT_ID
&refresh_token=REFRESH_TOKEN
```

## Usage Patterns

```typescript
// OAuth flow with local server
const server = await startLocalServer();
const {url, verifier, state} = await createAuthorizationUrl();
openBrowser(url);
const code = await server.waitForCallback();
await exchangeCodeForTokens(code, verifier, state);
server.close();

// Use with Hugging Face
const token = await getValidAccessToken() || await getUserAccessToken();
const client = new HfInference(token);
```

## Best Practices

1. **Offer both auth methods** for flexibility
2. **OAuth for applications** requiring refresh
3. **User tokens for simplicity** and testing
4. **Local callback server** for better UX
5. **Validate token format** (hf_* prefix)
6. **Clear permission instructions**
7. **Automatic token refresh**
8. **Secure token storage**

## Key Features

- **Dual Authentication**: OAuth and user access tokens
- **Local Callback Server**: No manual code copying
- **Fine-Grained Scopes**: Request only needed permissions
- **User Token Option**: Simpler alternative for personal use
- **Inference API Access**: Full API access with authentication

## Special Considerations

### User Access Tokens vs OAuth

- **User Access Tokens**: 
  - Simpler to create
  - Don't expire
  - Good for personal use and testing
  - Created at https://huggingface.co/settings/tokens

- **OAuth**:
  - For application access
  - Supports refresh tokens
  - Better for production
  - More secure with PKCE

### Token Permissions

When creating user access tokens, ensure 'inference-api' permission is selected.

## References

- [Hugging Face OAuth Documentation](https://huggingface.co/docs/hub/oauth)
- [Hugging Face Inference API](https://huggingface.co/docs/api-inference/)
- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [PKCE RFC 7636](https://tools.ietf.org/html/rfc7636)

---

This specification provides the framework for implementing Hugging Face OAuth authentication with both OAuth and user access token support for maximum flexibility.
