# OAuth Implementation Specifications

This directory contains comprehensive OAuth implementation specifications for various LLM providers in Command Code CLI. Each specification follows a consistent structure and can be used as a reference to implement OAuth authentication flows.

## Available Specifications

### 1. [Anthropic Claude OAuth](./oauth-anthropic.md)
- **Provider**: Anthropic (Claude Pro/Max)
- **OAuth Flow**: PKCE (Proof Key for Code Exchange)
- **Key Features**:
  - SHA256 code challenge
  - Manual authorization code entry
  - Token refresh with 5-minute buffer
  - Admin-only access with internal flag
  - Dual auth support (OAuth + API key)

### 2. [GitHub Copilot OAuth](./oauth-github-copilot.md)
- **Provider**: GitHub Copilot
- **OAuth Flow**: Device Authorization Grant (RFC 8628)
- **Key Features**:
  - Device flow optimized for CLI
  - User code entry in browser
  - Polling for authorization completion
  - 8-hour token validity
  - Copilot-specific token exchange
  - No refresh token (re-auth required)

### 3. [OpenCode Zen OAuth](./oauth-opencode-zen.md)
- **Provider**: OpenCode Zen (Managed by OpenCode team)
- **OAuth Flow**: PKCE (Proof Key for Code Exchange)
- **Key Features**:
  - Dual authentication methods (OAuth + API key)
  - SHA256 code challenge
  - Token refresh with 5-minute buffer
  - Public provider (not admin-only)
  - API key fallback for simpler setup
  - Both auth methods in single flow

## Specification Structure

Each OAuth specification document includes:

### 1. Architecture Components
- Core OAuth module with flow implementation
- Auth storage module for credential management
- CLI commands for user interaction
- Interactive UI components
- API client integration
- API request flow (CLI and server side)

### 2. Security Features
- OAuth flow security (PKCE, Device Flow, State)
- Token storage and permissions
- Token refresh mechanisms
- Access control and validation

### 3. Implementation Details
- Constants and configuration
- Provider-specific endpoints
- Error handling scenarios
- Testing approaches

### 4. Implementation Checklist
Complete step-by-step checklist for implementing each provider:
- Create OAuth module
- Create CLI commands
- Create UI components
- Integrate with client
- Update API request flow
- Add provider configuration
- Update constants
- Add tests

### 5. API Documentation
- Authorization endpoints
- Token exchange endpoints
- Token refresh endpoints
- Request/response formats

### 6. Best Practices
- Security recommendations
- Error handling patterns
- User experience guidelines
- Cross-platform considerations

## OAuth Flow Comparison

| Feature | Anthropic | GitHub Copilot | OpenCode Zen |
|---------|-----------|----------------|--------------|
| **Flow Type** | PKCE | Device Authorization | PKCE |
| **Code Entry** | Manual paste | User code in browser | Manual paste |
| **Token Refresh** | Yes (5 min buffer) | No (re-auth) | Yes (5 min buffer) |
| **Refresh Token** | Yes | No | Yes |
| **Admin Only** | Yes | Yes | No |
| **API Key Fallback** | Yes | Yes | Yes (primary option) |
| **Token Validity** | ~1 hour | ~8 hours | ~1 hour |
| **Polling Required** | No | Yes | No |
| **Browser Opening** | Automatic | Automatic | Automatic |

## Common Patterns

All OAuth implementations share these common patterns:

### 1. Auth Storage
```typescript
{
  "provider-id": {
    "type": "oauth",
    "access": "access_token_here",
    "refresh": "refresh_token_here",
    "expires": 1234567890000
  }
}
```

### 2. Token Validation
```typescript
const token = await ProviderOAuth.getValidAccessToken();
if (token) {
  // Use token for API requests
}
```

### 3. Request Headers
```typescript
headers[HEADERS.OAUTH_TOKEN] = `Bearer ${token}`;
```

### 4. Error Handling
- Token expiration → Refresh or re-authenticate
- Invalid token → Remove and prompt re-auth
- Network errors → Clear error messages
- Access denied → Suggest alternatives

## Security Considerations

### For All Providers

1. **Secure Storage**
   - File permissions: 0o600
   - Local storage only
   - Environment-specific files

2. **Token Transmission**
   - Always use HTTPS
   - Bearer token in headers
   - Never log tokens

3. **Token Refresh**
   - Automatic refresh before expiry
   - Graceful handling of refresh failures
   - Removal of invalid tokens

4. **Access Control**
   - Server-side validation
   - Admin/pre-approval checks where required
   - Internal flag enforcement

## Implementation Priority

Recommended order for implementing OAuth providers:

1. **Start with Anthropic** - Standard PKCE flow, good reference implementation
2. **Then OpenCode Zen** - Adds API key fallback pattern
3. **Finally GitHub Copilot** - More complex device flow with polling

## Adding New Provider Specifications

When adding a new OAuth provider specification:

1. **Copy template** from existing spec (Anthropic recommended)
2. **Update provider-specific details**:
   - Constants (CLIENT_ID, URLs, scopes)
   - OAuth flow type (PKCE, Device Flow, etc.)
   - Token refresh logic
   - Special requirements
3. **Document unique features**
4. **Update this README** with new provider entry
5. **Add to comparison table**

## Testing OAuth Implementations

Each implementation should be tested for:

- [ ] Authorization flow completion
- [ ] Token storage and retrieval
- [ ] Token refresh logic
- [ ] Token expiration handling
- [ ] Error scenarios (network, denied, expired)
- [ ] Cross-platform browser opening
- [ ] Re-authentication flow
- [ ] Logout functionality
- [ ] Status checking

## Resources

### OAuth Standards
- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [PKCE RFC 7636](https://tools.ietf.org/html/rfc7636)
- [Device Authorization Grant RFC 8628](https://tools.ietf.org/html/rfc8628)

### Provider Documentation
- [Anthropic OAuth](https://docs.anthropic.com/en/api/oauth)
- [GitHub Device Flow](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#device-flow)
- [OpenCode Documentation](https://opencode.ai/docs)

## Contributing

When contributing OAuth specifications:

1. Follow the established structure
2. Include complete implementation checklist
3. Document all security features
4. Provide API endpoint examples
5. Include error handling scenarios
6. Add testing guidelines
7. Update this README

## Support

For questions or issues with OAuth implementations:
- Review existing specifications for patterns
- Check the comparison table for differences
- Refer to OAuth RFCs for standard behavior
- Consult provider-specific documentation

---

*These specifications are designed for implementing OAuth authentication in Command Code CLI. Adapt as needed for your specific implementation.*
