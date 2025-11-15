# Azure OpenAI OAuth Implementation Spec

## Overview

This document provides a comprehensive specification of how OAuth authentication for Azure OpenAI could be implemented in the Command Code CLI using Microsoft Entra ID (formerly Azure Active Directory) authentication.

## Architecture Components

### 1. Core OAuth Module (`packages/command/src/auth/azure.ts`)

The central OAuth implementation using Microsoft Identity Platform with PKCE.

#### Constants

```typescript
const CLIENT_ID = '';
const TENANT_ID = 'common'; // or specific tenant ID
const AUTHORIZE_URL = `https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/authorize`;
const TOKEN_URL = `https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token`;
const REDIRECT_URI = 'http://localhost:8080/oauth/callback';
const SCOPES = ['https://cognitiveservices.azure.com/.default', 'offline_access'];
```

#### OAuth Flow

Azure uses Microsoft Identity Platform OAuth 2.0 with PKCE.

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
  - `response_mode`: 'query'
  - `prompt`: 'select_account'
- Returns: `{url, verifier, state}`

**2. `exchangeCodeForTokens(code, verifier, state)`**
- Makes POST request to TOKEN_URL with:
  - `grant_type`: 'authorization_code'
  - `client_id`: CLIENT_ID
  - `code`: Authorization code
  - `redirect_uri`: REDIRECT_URI
  - `code_verifier`: PKCE verifier
  - `scope`: SCOPES
- Receives token response:
  ```typescript
  {
    access_token: string;
    refresh_token: string;
    expires_in: number; // typically 3600
    token_type: string; // "Bearer"
    scope: string;
    id_token: string; // JWT
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
  - `scope`: SCOPES
- Updates stored tokens
- Removes auth data if refresh fails
- Returns new access token or undefined

**4. `getValidAccessToken()`**
- Retrieves stored auth data
- Calls refreshAccessToken() if needed
- Returns valid access token or undefined

**5. `getResourceName()`**
- Prompts user for Azure resource name
- Validates format (alphanumeric and hyphens)
- Stores resource name for API endpoint construction
- Returns resource name string

**6. `startLocalServer()`**
- Starts temporary HTTP server on port 8080
- Listens for OAuth callback
- Extracts authorization code
- Closes server after receiving code

### 2. Auth Storage Module

#### Storage Format
```json
{
  "azure": {
    "type": "oauth",
    "refresh": "refresh_token_here",
    "access": "access_token_here",
    "expires": 1234567890000,
    "resourceName": "my-openai-resource"
  }
}
```

### 3. CLI Commands (`packages/command/src/commands/azure-auth.ts`)

#### `auth login`
1. Prompt for Azure resource name
2. Generate authorization URL
3. Start local callback server
4. Open browser to authorization URL
5. Wait for callback with code
6. Exchange code for tokens
7. Store tokens and resource name
8. Display success message

#### `auth logout`
1. Check if authenticated
2. Optionally revoke tokens
3. Call `Auth.remove('azure')`
4. Display success message

#### `auth status`
1. Retrieve stored auth data
2. Display authentication status
3. Show resource name
4. Show token expiry

### 4. Interactive UI Component (`packages/command/src/components/azure-auth.tsx`)

#### Flow
1. Prompt for Azure resource name with validation
2. Start local callback server
3. Generate auth URL
4. Display authorization message
5. Open browser automatically
6. Show waiting spinner
7. Handle callback and exchange tokens
8. Update config with resource name
9. Display success

### 5. API Client Integration (`packages/command/src/clients/azure/azure.ts`)

```typescript
const oauthToken = await AzureOAuth.getValidAccessToken();
const resourceName = await AzureOAuth.getResourceName();

if (oauthToken && resourceName) {
  this.instance = new AzureOpenAI({
    apiKey: oauthToken,
    endpoint: `https://${resourceName}.openai.azure.com/`,
    apiVersion: '2024-08-01-preview',
  });
}
```

### 6. API Request Flow

```typescript
let token: string | undefined = undefined;
if (provider === PROVIDERS.AZURE) {
  token = await AzureOAuth.getValidAccessToken();
  const resourceName = await AzureOAuth.getResourceName();
  
  if (!resourceName) {
    throw new Error('Azure resource name not configured');
  }
  
  validateOAuthToken({token, provider});
}

if (token) {
  headers[HEADERS.OAUTH_TOKEN] = `Bearer ${token}`;
  headers['api-key'] = token; // Azure uses api-key header
}
```

## Security Features

### 1. Microsoft Identity Platform
- Enterprise-grade security
- Multi-factor authentication support
- Conditional access policies
- Compliance and auditing

### 2. PKCE
- Prevents code interception
- SHA256 challenge method
- Secure for public clients

### 3. Token Refresh
- Automatic refresh with buffer
- Refresh tokens typically valid for 90 days
- Can be revoked by admin

### 4. Resource Isolation
- Each Azure resource has its own endpoint
- Token scoped to Cognitive Services
- Resource name stored securely

### 5. Tenant Support
- Multi-tenant or single-tenant apps
- Tenant ID in authorization URL
- Supports both personal and work accounts

## Provider Configuration

```typescript
{
  [PROVIDER.AZURE]: {
    name: 'Azure OpenAI',
    description: 'Azure OpenAI Service',
    requiresAuth: true,
    hidden: false,
    authComponent: AzureAuth,
    checkAuth: async () => {
      const token = await AzureOAuth.getValidAccessToken();
      const resourceName = await AzureOAuth.getResourceName();
      return !!(token && resourceName);
    },
  },
}
```

## Implementation Checklist

- [ ] Create `packages/command/src/auth/azure.ts`
- [ ] Implement Microsoft Identity Platform OAuth
- [ ] Implement PKCE flow
- [ ] Implement local callback server
- [ ] Implement resource name configuration
- [ ] Create CLI commands
- [ ] Create UI component
- [ ] Integrate with Azure OpenAI client
- [ ] Handle tenant-specific URLs
- [ ] Add token refresh logic
- [ ] Add tests

## Error Handling

1. **Invalid Resource Name** - Validate format
2. **Tenant Not Found** - Check tenant ID
3. **Token Exchange Failed** - Display Microsoft error
4. **Refresh Failed** - Re-authenticate
5. **Missing Permissions** - Guide user to Azure portal

## API Endpoints

### Authorization
```
GET https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize
Parameters: client_id, response_type, redirect_uri, scope, code_challenge, code_challenge_method, state, response_mode, prompt
```

### Token Exchange
```
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=CLIENT_ID
&code=CODE
&redirect_uri=REDIRECT_URI
&code_verifier=VERIFIER
&scope=SCOPES
```

### Token Refresh
```
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&client_id=CLIENT_ID
&refresh_token=REFRESH_TOKEN
&scope=SCOPES
```

## Usage Patterns

```typescript
// Configure resource and authenticate
const resourceName = await promptResourceName();
const {url, verifier, state} = await createAuthorizationUrl();
openBrowser(url);
const code = await waitForCallback();
const tokens = await exchangeCodeForTokens(code, verifier, state);

// Use with Azure OpenAI
const token = await getValidAccessToken();
const client = new AzureOpenAI({
  apiKey: token,
  endpoint: `https://${resourceName}.openai.azure.com/`,
});
```

## Best Practices

1. **Store resource name** with tokens
2. **Validate resource name format** during input
3. **Support multiple tenants** via configuration
4. **Local callback server** for better UX
5. **Automatic token refresh**
6. **Clear error messages** from Microsoft
7. **Handle MFA** gracefully
8. **Support work and personal accounts**

## Key Features

- **Enterprise SSO**: Microsoft Entra ID integration
- **Local Callback**: Automatic code capture
- **Resource Configuration**: Store Azure resource name
- **Token Refresh**: 90-day refresh token validity
- **Multi-Tenant**: Support various tenant configurations

## Special Considerations

### Deployment Name Requirements
Azure OpenAI requires deployment names that match model names. Users need to:
1. Create deployments in Azure AI Foundry
2. Ensure deployment name matches model name
3. Configure resource name in Command Code

### Environment Variables
```bash
AZURE_RESOURCE_NAME=my-openai-resource
AZURE_TENANT_ID=common # or specific tenant
```

## References

- [Microsoft Identity Platform](https://docs.microsoft.com/en-us/azure/active-directory/develop/)
- [Azure OpenAI Documentation](https://docs.microsoft.com/en-us/azure/cognitive-services/openai/)
- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [PKCE RFC 7636](https://tools.ietf.org/html/rfc7636)

---

This specification provides the framework for implementing Azure OpenAI OAuth authentication using Microsoft Identity Platform with enterprise SSO support.
