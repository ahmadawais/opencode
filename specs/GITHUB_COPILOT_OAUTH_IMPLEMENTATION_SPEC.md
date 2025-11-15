# GitHub Copilot OAuth Implementation Specification

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [OAuth Device Flow](#oauth-device-flow)
4. [Plugin System](#plugin-system)
5. [Authentication Storage](#authentication-storage)
6. [Token Management](#token-management)
7. [API Integration](#api-integration)
8. [Enterprise Support](#enterprise-support)
9. [Provider Integration](#provider-integration)
10. [Command Line Interface](#command-line-interface)
11. [Implementation Details](#implementation-details)
12. [Security Considerations](#security-considerations)

---

## Overview

The GitHub Copilot OAuth implementation in opencode uses the **OAuth 2.0 Device Authorization Grant** flow (RFC 8628) to authenticate users with their GitHub accounts. This allows the CLI application to obtain access tokens without requiring a web browser redirect.

### Key Features
- Device code flow for terminal-based authentication
- Support for both GitHub.com and GitHub Enterprise deployments
- Automatic token refresh mechanism
- Secure credential storage
- Plugin-based authentication architecture
- Cost-free model pricing (all Copilot models marked as $0)

---

## Architecture

### High-Level Components

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode CLI                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Auth Command │──│ Plugin System│──│ Provider Manager │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ Auth Storage (JSON)   │
              │ ~/.opencode/auth.json │
              └──────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────┐
         │    GitHub OAuth Services      │
         ├───────────────────────────────┤
         │ • Device Authorization        │
         │ • Token Exchange              │
         │ • Copilot Token Generation    │
         └───────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │ GitHub Copilot API    │
              └──────────────────────┘
```

### Component Breakdown

1. **CLI Interface** - User-facing commands (`opencode auth login`)
2. **Plugin System** - Modular authentication providers
3. **Auth Storage** - Secure local credential storage
4. **Provider Manager** - Dynamic model loading and SDK integration
5. **Token Manager** - Automatic token refresh and validation

---

## OAuth Device Flow

### Flow Diagram

```
User                CLI                GitHub OAuth           Copilot API
 │                   │                      │                     │
 │ auth login ────>  │                      │                     │
 │                   │                      │                     │
 │                   │ POST device_code ──> │                     │
 │                   │ <─── device_code,    │                     │
 │                   │      user_code,      │                     │
 │                   │      verification_uri│                     │
 │                   │                      │                     │
 │ <──── Display ────│                      │                     │
 │    "Visit URL"    │                      │                     │
 │    "Enter code"   │                      │                     │
 │                   │                      │                     │
 │ Opens Browser     │                      │                     │
 │ ──────────────────────────────────────> │                     │
 │                   │                      │                     │
 │ Enters Code       │                      │                     │
 │ ──────────────────────────────────────> │                     │
 │                   │                      │                     │
 │ Authorizes        │                      │                     │
 │ ──────────────────────────────────────> │                     │
 │                   │                      │                     │
 │                   │ POST access_token ─> │                     │
 │                   │    (polling)         │                     │
 │                   │ <─── access_token ───│                     │
 │                   │                      │                     │
 │                   │ Store refresh token  │                     │
 │                   │                      │                     │
 │                   │ POST copilot_token ───────────────────────>│
 │                   │ <──────────────────────── copilot_token ───│
 │                   │                      │                     │
 │                   │ Store access token   │                     │
 │ <─── Success ─────│                      │                     │
```

### Step-by-Step Flow

#### 1. **Device Code Request**
```javascript
POST https://github.com/login/device/code
Content-Type: application/json

{
  "client_id": "Iv1.b507a08c87ecfe98",
  "scope": "read:user"
}
```

**Response:**
```javascript
{
  "device_code": "3584d83530557fdd1f46af8289938c8ef79f9dc5",
  "user_code": "8F43-6FCF",
  "verification_uri": "https://github.com/login/device",
  "expires_in": 900,
  "interval": 5
}
```

#### 2. **User Authorization**
- User navigates to `verification_uri` (https://github.com/login/device)
- Enters the `user_code` displayed by CLI
- Authenticates with GitHub credentials
- Authorizes the application

#### 3. **Token Polling**
CLI polls for authorization completion:

```javascript
POST https://github.com/login/oauth/access_token
Content-Type: application/json

{
  "client_id": "Iv1.b507a08c87ecfe98",
  "device_code": "3584d83530557fdd1f46af8289938c8ef79f9dc5",
  "grant_type": "urn:ietf:params:oauth:grant-type:device_code"
}
```

**Pending Response:**
```javascript
{
  "error": "authorization_pending"
}
```

**Success Response:**
```javascript
{
  "access_token": "gho_16C7e42F292c6912E7710c838347Ae178B4a",
  "token_type": "bearer",
  "scope": "read:user"
}
```

#### 4. **Copilot Token Exchange**
Exchange GitHub OAuth token for Copilot API token:

```javascript
POST https://api.github.com/copilot_internal/v2/token
Headers:
  Authorization: Bearer gho_16C7e42F292c6912E7710c838347Ae178B4a
  User-Agent: GitHubCopilotChat/0.32.4
  Editor-Version: vscode/1.105.1
  Editor-Plugin-Version: copilot-chat/0.32.4
  Copilot-Integration-Id: vscode-chat
```

**Response:**
```javascript
{
  "token": "copilot_token_here",
  "expires_at": 1640000000
}
```

---

## Plugin System

### Plugin Architecture

The authentication is implemented as a plugin following the opencode plugin architecture:

```typescript
export interface Plugin {
  auth?: {
    provider: string
    loader?: (auth: () => Promise<Auth>, provider: Provider) => Promise<Record<string, any>>
    methods: AuthMethod[]
  }
}
```

### GitHub Copilot Plugin Structure

The plugin is distributed as `opencode-copilot-auth@0.0.5` on npm and automatically loaded by opencode.

**Package Location:**
```
~/.bun/install/cache/opencode-copilot-auth@0.0.5/
```

**Plugin Entry Point (`index.mjs`):**

```javascript
export async function CopilotAuthPlugin({ client }) {
  return {
    auth: {
      provider: "github-copilot",
      loader: async (getAuth, provider) => {
        // Dynamic configuration loader
      },
      methods: [
        {
          type: "oauth",
          label: "Login with GitHub Copilot",
          prompts: [...],
          authorize: async (inputs) => {
            // OAuth flow implementation
          }
        }
      ]
    }
  }
}
```

### Plugin Loading

Plugins are loaded in `/packages/opencode/src/plugin/index.ts`:

```typescript
const plugins = [...(config.plugin ?? [])]
if (!Flag.OPENCODE_DISABLE_DEFAULT_PLUGINS) {
  plugins.push("opencode-copilot-auth@0.0.5")
  plugins.push("opencode-anthropic-auth@0.0.2")
}

for (let plugin of plugins) {
  if (!plugin.startsWith("file://")) {
    const lastAtIndex = plugin.lastIndexOf("@")
    const pkg = lastAtIndex > 0 ? plugin.substring(0, lastAtIndex) : plugin
    const version = lastAtIndex > 0 ? plugin.substring(lastAtIndex + 1) : "latest"
    plugin = await BunProc.install(pkg, version)
  }
  const mod = await import(plugin)
  for (const [_name, fn] of Object.entries(mod)) {
    const init = await fn(input)
    hooks.push(init)
  }
}
```

---

## Authentication Storage

### Storage Location

Credentials are stored in:
```
~/.opencode/data/auth.json
```

**File Permissions:** `0600` (read/write for owner only)

### Storage Schema

```typescript
type Auth = {
  type: "oauth"
  refresh: string      // GitHub OAuth access token (used as refresh token)
  access: string       // Copilot API token
  expires: number      // Expiration timestamp (milliseconds)
  enterpriseUrl?: string  // Optional: GitHub Enterprise URL
}
```

### Storage Format

```json
{
  "github-copilot": {
    "type": "oauth",
    "refresh": "gho_16C7e42F292c6912E7710c838347Ae178B4a",
    "access": "copilot_api_token_here",
    "expires": 1640000000000
  },
  "github-copilot-enterprise": {
    "type": "oauth",
    "refresh": "gho_EnterpriseTokenHere",
    "access": "enterprise_copilot_token",
    "expires": 1640000000000,
    "enterpriseUrl": "company.ghe.com"
  }
}
```

### Storage API

Located in `/packages/opencode/src/auth/index.ts`:

```typescript
export namespace Auth {
  // Get credentials for a provider
  export async function get(providerID: string): Promise<Info | undefined>
  
  // Get all credentials
  export async function all(): Promise<Record<string, Info>>
  
  // Set credentials for a provider
  export async function set(key: string, info: Info): Promise<void>
  
  // Remove credentials for a provider
  export async function remove(key: string): Promise<void>
}
```

**Key Implementation Details:**
1. Uses Bun.file() for efficient file operations
2. Automatically sets secure file permissions (0600)
3. Supports multiple authentication types (oauth, api, wellknown)
4. Thread-safe with atomic write operations

---

## Token Management

### Token Types

1. **Refresh Token** (GitHub OAuth Token)
   - Long-lived access token from GitHub OAuth
   - Stored as `refresh` in auth.json
   - Used to obtain Copilot API tokens
   - Persists across sessions

2. **Access Token** (Copilot API Token)
   - Short-lived token for Copilot API calls
   - Stored as `access` in auth.json
   - Expires after ~1 hour
   - Automatically refreshed

### Token Refresh Flow

```javascript
async fetch(input, init) {
  const info = await getAuth()
  
  // Check if token is expired or missing
  if (!info.access || info.expires < Date.now()) {
    // Exchange refresh token for new access token
    const response = await fetch(
      `https://api.${domain}/copilot_internal/v2/token`,
      {
        headers: {
          Accept: "application/json",
          Authorization: `Bearer ${info.refresh}`,
          "User-Agent": "GitHubCopilotChat/0.32.4",
          "Editor-Version": "vscode/1.105.1",
          "Editor-Plugin-Version": "copilot-chat/0.32.4",
          "Copilot-Integration-Id": "vscode-chat"
        }
      }
    )
    
    const tokenData = await response.json()
    
    // Update stored credentials
    await client.auth.set({
      path: { id: saveProviderID },
      body: {
        type: "oauth",
        refresh: info.refresh,
        access: tokenData.token,
        expires: tokenData.expires_at * 1000,
        ...(info.enterpriseUrl && { enterpriseUrl: info.enterpriseUrl })
      }
    })
    
    info.access = tokenData.token
  }
  
  // Use refreshed token for API call
  return fetch(input, {
    ...init,
    headers: {
      ...init.headers,
      Authorization: `Bearer ${info.access}`
    }
  })
}
```

### Automatic Refresh Triggers

Token refresh occurs automatically:
1. Before every API request
2. When `info.access` is empty
3. When `info.expires < Date.now()`

---

## API Integration

### Request Headers

All Copilot API requests include:

```javascript
const HEADERS = {
  "User-Agent": "GitHubCopilotChat/0.32.4",
  "Editor-Version": "vscode/1.105.1",
  "Editor-Plugin-Version": "copilot-chat/0.32.4",
  "Copilot-Integration-Id": "vscode-chat",
  "Authorization": `Bearer ${copilot_token}`,
  "Openai-Intent": "conversation-edits",
  "X-Initiator": isAgentCall ? "agent" : "user"
}
```

### Special Headers

#### Vision Requests
For messages containing images:
```javascript
headers["Copilot-Vision-Request"] = "true"
```

#### Agent vs User Initiator
Determined by message content:
```javascript
const isAgentCall = body.messages.some(
  (msg) => msg.role && ["tool", "assistant"].includes(msg.role)
)
headers["X-Initiator"] = isAgentCall ? "agent" : "user"
```

### Base URLs

**GitHub.com:**
```
https://api.githubcopilot.com
```

**GitHub Enterprise:**
```
https://copilot-api.{enterprise-domain}
```

Example: `https://copilot-api.company.ghe.com`

### API Endpoints

#### Chat Completions
```
POST https://api.githubcopilot.com/v1/chat/completions
```

Request format follows OpenAI API specification.

---

## Enterprise Support

### Deployment Types

The implementation supports two deployment types:

1. **GitHub.com (Public)**
   - Domain: `github.com`
   - API: `https://api.githubcopilot.com`
   - Provider ID: `github-copilot`

2. **GitHub Enterprise**
   - Domain: Custom (e.g., `company.ghe.com`)
   - API: `https://copilot-api.{domain}`
   - Provider ID: `github-copilot-enterprise`

### Enterprise Configuration Flow

#### User Prompts

```javascript
prompts: [
  {
    type: "select",
    key: "deploymentType",
    message: "Select GitHub deployment type",
    options: [
      {
        label: "GitHub.com",
        value: "github.com",
        hint: "Public"
      },
      {
        label: "GitHub Enterprise",
        value: "enterprise",
        hint: "Data residency or self-hosted"
      }
    ]
  },
  {
    type: "text",
    key: "enterpriseUrl",
    message: "Enter your GitHub Enterprise URL or domain",
    placeholder: "company.ghe.com or https://company.ghe.com",
    condition: (inputs) => inputs.deploymentType === "enterprise",
    validate: (value) => {
      if (!value) return "URL or domain is required"
      try {
        const url = value.includes("://")
          ? new URL(value)
          : new URL(`https://${value}`)
        if (!url.hostname) return "Please enter a valid URL or domain"
        return undefined
      } catch {
        return "Please enter a valid URL"
      }
    }
  }
]
```

### URL Normalization

```javascript
function normalizeDomain(url) {
  return url.replace(/^https?:\/\//, "").replace(/\/$/, "")
}

function getUrls(domain) {
  return {
    DEVICE_CODE_URL: `https://${domain}/login/device/code`,
    ACCESS_TOKEN_URL: `https://${domain}/login/oauth/access_token`,
    COPILOT_API_KEY_URL: `https://api.${domain}/copilot_internal/v2/token`
  }
}
```

### Provider Separation

Enterprise credentials are stored separately:

```javascript
// GitHub.com
"github-copilot": {
  "type": "oauth",
  "refresh": "...",
  "access": "...",
  "expires": 1640000000000
}

// GitHub Enterprise
"github-copilot-enterprise": {
  "type": "oauth",
  "refresh": "...",
  "access": "...",
  "expires": 1640000000000,
  "enterpriseUrl": "company.ghe.com"
}
```

---

## Provider Integration

### Provider Registration

Located in `/packages/opencode/src/provider/provider.ts`:

#### Dynamic Provider Creation

```javascript
// Add GitHub Copilot Enterprise provider that inherits from GitHub Copilot
if (database["github-copilot"]) {
  const githubCopilot = database["github-copilot"]
  database["github-copilot-enterprise"] = {
    ...githubCopilot,
    id: "github-copilot-enterprise",
    name: "GitHub Copilot Enterprise",
    // Enterprise uses different API endpoint - set dynamically based on auth
    api: undefined
  }
}
```

### Provider Loader

The plugin's loader function configures the provider dynamically:

```javascript
loader: async (getAuth, provider) => {
  let info = await getAuth()
  if (!info || info.type !== "oauth") return {}

  // Mark all models as free
  if (provider && provider.models) {
    for (const model of Object.values(provider.models)) {
      model.cost = {
        input: 0,
        output: 0
      }
    }
  }

  // Set baseURL based on deployment type
  const enterpriseUrl = info.enterpriseUrl
  const baseURL = enterpriseUrl
    ? `https://copilot-api.${normalizeDomain(enterpriseUrl)}`
    : "https://api.githubcopilot.com"

  return {
    baseURL,
    apiKey: "",
    async fetch(input, init) {
      // Custom fetch with token refresh
    }
  }
}
```

### Model Loading

Models are fetched from https://models.dev/api.json:

```javascript
export async function get() {
  refresh()
  const file = Bun.file(filepath)
  const result = await file.json().catch(() => {})
  if (result) return result as Record<string, Provider>
  const json = await data()
  return JSON.parse(json) as Record<string, Provider>
}
```

### Provider State Management

```javascript
const state = Instance.state(async () => {
  const config = await Config.get()
  const database = await ModelsDev.get()
  
  const providers = {}
  const models = new Map()
  const sdk = new Map()
  
  // Load from environment variables
  // Load from API keys storage
  // Load from custom loaders
  // Load from plugins
  
  return { models, providers, sdk }
})
```

---

## Command Line Interface

### Auth Commands

Located in `/packages/opencode/src/cli/cmd/auth.ts`:

#### Main Command
```bash
opencode auth
```

#### Subcommands

**1. Login**
```bash
opencode auth login
opencode auth login [url]  # For custom auth providers
```

**2. List Credentials**
```bash
opencode auth list
opencode auth ls
```

**3. Logout**
```bash
opencode auth logout
```

### Login Flow UI

#### Provider Selection
```
┌  Add credential
│
◇  Select provider
│  GitHub Copilot (recommended)
│  Anthropic (recommended)
│  OpenAI
│  Google
│  ...
```

#### Deployment Type Selection
```
◇  Select GitHub deployment type
│  ○ GitHub.com (Public)
│  ○ GitHub Enterprise (Data residency or self-hosted)
```

#### Enterprise URL Input
```
◇  Enter your GitHub Enterprise URL or domain
│  company.ghe.com or https://company.ghe.com
```

#### Device Code Display
```
◇   ──────────────────────────────────────────────╮
│                                                 │
│  Please visit: https://github.com/login/device  │
│  Enter code: 8F43-6FCF                          │
│                                                 │
├─────────────────────────────────────────────────╯
│
◓  Waiting for authorization...
```

#### Success
```
◆  Login successful
└  Done
```

### List Output

```bash
$ opencode auth list

Credentials ~/.opencode/data/auth.json

  ℹ GitHub Copilot oauth
  ℹ Anthropic api

2 credentials

Environment

  ℹ OpenAI OPENAI_API_KEY
  ℹ Google GOOGLE_GENERATIVE_AI_API_KEY

2 environment variables
```

---

## Implementation Details

### Constants

```javascript
// Client ID for GitHub OAuth App
const CLIENT_ID = "Iv1.b507a08c87ecfe98"

// Required headers for Copilot API
const HEADERS = {
  "User-Agent": "GitHubCopilotChat/0.32.4",
  "Editor-Version": "vscode/1.105.1",
  "Editor-Plugin-Version": "copilot-chat/0.32.4",
  "Copilot-Integration-Id": "vscode-chat"
}
```

### Error Handling

#### OAuth Errors
```javascript
if (data.error === "authorization_pending") {
  // Continue polling
  await new Promise(resolve => 
    setTimeout(resolve, deviceData.interval * 1000)
  )
  continue
}

if (data.error === "expired_token") {
  return { type: "failed" }
}

if (data.error === "access_denied") {
  return { type: "failed" }
}
```

#### Token Refresh Errors
```javascript
const response = await fetch(urls.COPILOT_API_KEY_URL, {
  headers: {
    Accept: "application/json",
    Authorization: `Bearer ${info.refresh}`,
    ...HEADERS
  }
})

if (!response.ok) {
  // Token refresh failed, user needs to re-authenticate
  return
}
```

### Polling Mechanism

```javascript
while (true) {
  const response = await fetch(urls.ACCESS_TOKEN_URL, {
    method: "POST",
    headers: {
      Accept: "application/json",
      "Content-Type": "application/json",
      "User-Agent": "GitHubCopilotChat/0.35.0"
    },
    body: JSON.stringify({
      client_id: CLIENT_ID,
      device_code: deviceData.device_code,
      grant_type: "urn:ietf:params:oauth:grant-type:device_code"
    })
  })

  if (!response.ok) return { type: "failed" }

  const data = await response.json()

  if (data.access_token) {
    // Success - store token
    return {
      type: "success",
      refresh: data.access_token,
      access: "",
      expires: 0,
      ...(enterpriseUrl && { 
        provider: "github-copilot-enterprise",
        enterpriseUrl: domain 
      })
    }
  }

  if (data.error === "authorization_pending") {
    // Wait and retry
    await new Promise(resolve => 
      setTimeout(resolve, deviceData.interval * 1000)
    )
    continue
  }

  if (data.error) return { type: "failed" }
}
```

### Model Cost Override

All GitHub Copilot models are marked as free:

```javascript
loader: async (getAuth, provider) => {
  if (provider && provider.models) {
    for (const model of Object.values(provider.models)) {
      model.cost = {
        input: 0,
        output: 0
      }
    }
  }
  // ... rest of loader
}
```

### Small Model Selection

For operations like title generation, premium models are excluded:

```javascript
export async function getSmallModel(providerID: string) {
  const provider = await state().then(state => state.providers[providerID])
  if (!provider) return
  
  let priority = [
    "claude-haiku-4-5",
    "claude-haiku-4.5",
    "3-5-haiku",
    "3.5-haiku",
    "gemini-2.5-flash",
    "gpt-5-nano"
  ]
  
  // claude-haiku-4.5 is premium in github copilot
  if (providerID === "github-copilot") {
    priority = priority.filter(m => m !== "claude-haiku-4.5")
  }
  
  for (const item of priority) {
    for (const model of Object.keys(provider.info.models)) {
      if (model.includes(item)) return getModel(providerID, model)
    }
  }
}
```

---

## Security Considerations

### 1. **Credential Storage**

**Security Measures:**
- File permissions set to `0600` (owner read/write only)
- Stored in user's home directory
- JSON format for easy inspection
- No plaintext passwords (only OAuth tokens)

**Location:**
```
~/.opencode/data/auth.json (chmod 600)
```

### 2. **Token Expiration**

- Access tokens expire after ~1 hour
- Automatic refresh before expiration
- Refresh tokens are long-lived
- No manual token rotation required

### 3. **Network Security**

- All API calls use HTTPS
- TLS certificate validation enabled
- No token transmission in URLs
- Tokens only in Authorization headers

### 4. **Client ID**

The OAuth client ID is public:
```javascript
const CLIENT_ID = "Iv1.b507a08c87ecfe98"
```

This is intentional and follows OAuth best practices for native applications. The security relies on:
- User authorization on GitHub
- Secure token storage
- HTTPS for all communications

### 5. **Enterprise Isolation**

- Separate provider IDs for public and enterprise
- Enterprise URLs validated before use
- No credential sharing between deployments
- Domain normalization for consistency

### 6. **Header Sanitization**

```javascript
delete headers["x-api-key"]  // Remove any conflicting auth headers
```

### 7. **Error Information**

- No sensitive data in error messages
- Generic failure responses
- Detailed errors only in debug mode
- No token leakage in logs

---

## Complete Implementation Checklist

Use this checklist when implementing GitHub Copilot OAuth in a new repository:

### Phase 1: Project Setup
- [ ] Set up plugin system architecture
- [ ] Create authentication storage module
- [ ] Implement secure file operations with proper permissions
- [ ] Set up provider management system

### Phase 2: OAuth Flow
- [ ] Implement device code request
- [ ] Create polling mechanism for authorization
- [ ] Handle authorization success/failure states
- [ ] Store OAuth tokens securely

### Phase 3: Token Management
- [ ] Implement token refresh logic
- [ ] Create automatic token expiration checking
- [ ] Exchange GitHub token for Copilot API token
- [ ] Handle token refresh errors gracefully

### Phase 4: Plugin System
- [ ] Create plugin interface definition
- [ ] Implement plugin loader
- [ ] Create GitHub Copilot auth plugin
- [ ] Add plugin registration in provider system

### Phase 5: Enterprise Support
- [ ] Add deployment type selection UI
- [ ] Implement URL normalization
- [ ] Create separate enterprise provider
- [ ] Handle enterprise-specific API endpoints

### Phase 6: CLI Interface
- [ ] Implement `auth login` command
- [ ] Create provider selection UI
- [ ] Add deployment type prompts
- [ ] Display device code and instructions
- [ ] Show authorization status

### Phase 7: API Integration
- [ ] Implement custom fetch with auth headers
- [ ] Add automatic token refresh on requests
- [ ] Handle vision request headers
- [ ] Detect agent vs user-initiated calls

### Phase 8: Testing
- [ ] Test public GitHub flow
- [ ] Test enterprise GitHub flow
- [ ] Test token refresh mechanism
- [ ] Test error handling
- [ ] Test concurrent requests

### Phase 9: Security
- [ ] Verify file permissions (0600)
- [ ] Validate HTTPS for all requests
- [ ] Sanitize headers
- [ ] Test credential isolation
- [ ] Audit error messages for leaks

### Phase 10: Documentation
- [ ] Document OAuth flow
- [ ] Create usage examples
- [ ] Document enterprise setup
- [ ] Add troubleshooting guide

---

## Code References

### Key Files

1. **Plugin Implementation**
   - Package: `opencode-copilot-auth@0.0.5`
   - Entry: `index.mjs`

2. **Auth Command**
   - File: `/packages/opencode/src/cli/cmd/auth.ts`
   - Lines: 68-325 (AuthLoginCommand)

3. **Auth Storage**
   - File: `/packages/opencode/src/auth/index.ts`
   - Lines: 1-65 (Full module)

4. **Provider Integration**
   - File: `/packages/opencode/src/provider/provider.ts`
   - Lines: 269-416 (Enterprise setup and plugin loading)

5. **Plugin System**
   - File: `/packages/opencode/src/plugin/index.ts`
   - Lines: 14-53 (Plugin loading)

6. **Plugin Interface**
   - File: `/packages/plugin/src/index.ts`
   - Lines: 35-142 (Auth interface definition)

---

## Example Usage Scenarios

### Scenario 1: First-Time User (GitHub.com)

```bash
$ opencode auth login

┌  Add credential
│
◇  Select provider
│  GitHub Copilot (recommended)
│
◇   ──────────────────────────────────────────────╮
│                                                 │
│  Please visit: https://github.com/login/device  │
│  Enter code: 8F43-6FCF                          │
│                                                 │
├─────────────────────────────────────────────────╯
│
◓  Waiting for authorization...
◆  Login successful
└  Done
```

### Scenario 2: Enterprise User

```bash
$ opencode auth login

┌  Add credential
│
◇  Select provider
│  GitHub Copilot
│
◇  Select GitHub deployment type
│  GitHub Enterprise (Data residency or self-hosted)
│
◇  Enter your GitHub Enterprise URL or domain
│  company.ghe.com
│
◇   ──────────────────────────────────────────────────╮
│                                                     │
│  Please visit: https://company.ghe.com/login/device │
│  Enter code: AB12-CD34                              │
│                                                     │
├───────────────────────────────────────────────────────╯
│
◓  Waiting for authorization...
◆  Login successful
└  Done
```

### Scenario 3: Using Copilot Models

```bash
$ opencode

# Model selection includes GitHub Copilot models
# All marked as $0 cost

$ opencode /models
┌ Available models
│ GitHub Copilot
│  ○ claude-sonnet-4-5 (free)
│  ○ gpt-5 (free)
│  ○ gemini-2.5-pro (free)
```

---

## Troubleshooting

### Common Issues

#### 1. Authorization Timeout
**Symptom:** Polling stops after device code expires

**Solution:**
- Device codes expire after 15 minutes
- Restart `opencode auth login`
- Complete authorization faster

#### 2. Token Refresh Failure
**Symptom:** API calls fail with 401 Unauthorized

**Possible Causes:**
- Refresh token expired or revoked
- GitHub OAuth app permissions changed
- Network connectivity issues

**Solution:**
- Run `opencode auth logout` for the provider
- Run `opencode auth login` again

#### 3. Enterprise URL Issues
**Symptom:** Cannot connect to enterprise GitHub

**Checks:**
- Verify URL format (with or without https://)
- Confirm GitHub Enterprise supports Copilot
- Check network access to enterprise domain
- Validate OAuth app is configured in enterprise

#### 4. Permission Denied (auth.json)
**Symptom:** Cannot read/write auth.json

**Solution:**
```bash
chmod 600 ~/.opencode/data/auth.json
chown $USER ~/.opencode/data/auth.json
```

#### 5. Missing Models
**Symptom:** No Copilot models available

**Possible Causes:**
- Models database not refreshed
- Provider not properly loaded
- Authentication incomplete

**Solution:**
```bash
# Refresh models database
opencode auth list  # Forces refresh

# Re-authenticate
opencode auth logout
opencode auth login
```

---

## API Endpoints Reference

### GitHub OAuth Endpoints

#### Device Authorization
```
POST https://github.com/login/device/code
POST https://{enterprise-domain}/login/device/code
```

#### Token Exchange
```
POST https://github.com/login/oauth/access_token
POST https://{enterprise-domain}/login/oauth/access_token
```

#### Copilot Token
```
POST https://api.github.com/copilot_internal/v2/token
POST https://api.{enterprise-domain}/copilot_internal/v2/token
```

### GitHub Copilot API Endpoints

#### Chat Completions
```
POST https://api.githubcopilot.com/v1/chat/completions
POST https://copilot-api.{enterprise-domain}/v1/chat/completions
```

---

## Version Information

**OpenCode Version:** Latest (as of implementation)
**Plugin Version:** opencode-copilot-auth@0.0.5
**OAuth Client ID:** Iv1.b507a08c87ecfe98
**Supported GitHub Versions:**
- GitHub.com (all features)
- GitHub Enterprise Server 3.10+
- GitHub Enterprise Cloud (all features)

---

## Future Enhancements

### Potential Improvements

1. **Multi-Account Support**
   - Support multiple GitHub accounts
   - Account switching UI
   - Per-project account selection

2. **Token Refresh Optimization**
   - Predictive refresh before expiration
   - Batch token refresh for multiple providers
   - Retry logic with exponential backoff

3. **Enhanced Security**
   - Keychain/Credential Manager integration
   - Encrypted credential storage
   - Token revocation on logout

4. **Better Enterprise Support**
   - Auto-discover enterprise URLs
   - Organization-wide configuration
   - SSO integration

5. **Improved Error Handling**
   - Detailed error messages
   - Recovery suggestions
   - Automatic retry mechanisms

---

## Conclusion

This specification document provides a comprehensive guide to implementing GitHub Copilot OAuth authentication in a new repository. The implementation uses industry-standard OAuth 2.0 Device Authorization Grant flow, supports both public and enterprise GitHub deployments, implements automatic token refresh, and provides a secure, user-friendly authentication experience.

Key takeaways:
1. Use OAuth Device Flow for CLI applications
2. Implement automatic token refresh
3. Support enterprise deployments
4. Store credentials securely with proper file permissions
5. Use a plugin architecture for extensibility
6. Provide clear user feedback during authentication
7. Handle errors gracefully with appropriate retry logic

By following this specification, you can create a robust, secure, and user-friendly GitHub Copilot integration that works seamlessly for both individual developers and enterprise teams.

---

**Document Version:** 1.0  
**Last Updated:** 2025-11-15  
**Author:** Generated from opencode codebase analysis  
**License:** Same as opencode project
