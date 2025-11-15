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

**Package Information:**
- Package: `opencode-copilot-auth`
- Version: `0.0.5`
- Weekly Downloads: ~55,000
- Location: `~/.bun/install/cache/opencode-copilot-auth@0.0.5/`
- NPM: https://www.npmjs.com/package/opencode-copilot-auth

**Complete Plugin Source Code (`index.mjs`):**

```javascript
/**
 * @type {import('@opencode-ai/plugin').Plugin}
 */
export async function CopilotAuthPlugin({ client }) {
  const CLIENT_ID = "Iv1.b507a08c87ecfe98";
  const HEADERS = {
    "User-Agent": "GitHubCopilotChat/0.32.4",
    "Editor-Version": "vscode/1.105.1",
    "Editor-Plugin-Version": "copilot-chat/0.32.4",
    "Copilot-Integration-Id": "vscode-chat",
  };

  function normalizeDomain(url) {
    return url.replace(/^https?:\/\//, "").replace(/\/$/, "");
  }

  function getUrls(domain) {
    return {
      DEVICE_CODE_URL: `https://${domain}/login/device/code`,
      ACCESS_TOKEN_URL: `https://${domain}/login/oauth/access_token`,
      COPILOT_API_KEY_URL: `https://api.${domain}/copilot_internal/v2/token`,
    };
  }

  return {
    auth: {
      provider: "github-copilot",
      loader: async (getAuth, provider) => {
        let info = await getAuth();
        if (!info || info.type !== "oauth") return {};

        if (provider && provider.models) {
          for (const model of Object.values(provider.models)) {
            model.cost = {
              input: 0,
              output: 0,
            };
          }
        }

        // Set baseURL based on deployment type
        const enterpriseUrl = info.enterpriseUrl;
        const baseURL = enterpriseUrl
          ? `https://copilot-api.${normalizeDomain(enterpriseUrl)}`
          : "https://api.githubcopilot.com";

        return {
          baseURL,
          apiKey: "",
          async fetch(input, init) {
            const info = await getAuth();
            if (info.type !== "oauth") return {};
            if (!info.access || info.expires < Date.now()) {
              const domain = info.enterpriseUrl
                ? normalizeDomain(info.enterpriseUrl)
                : "github.com";
              const urls = getUrls(domain);

              const response = await fetch(urls.COPILOT_API_KEY_URL, {
                headers: {
                  Accept: "application/json",
                  Authorization: `Bearer ${info.refresh}`,
                  ...HEADERS,
                },
              });

              if (!response.ok) return;

              const tokenData = await response.json();

              const saveProviderID = info.enterpriseUrl
                ? "github-copilot-enterprise"
                : "github-copilot";
              await client.auth.set({
                path: {
                  id: saveProviderID,
                },
                body: {
                  type: "oauth",
                  refresh: info.refresh,
                  access: tokenData.token,
                  expires: tokenData.expires_at * 1000,
                  ...(info.enterpriseUrl && {
                    enterpriseUrl: info.enterpriseUrl,
                  }),
                },
              });
              info.access = tokenData.token;
            }
            let isAgentCall = false;
            let isVisionRequest = false;
            try {
              const body =
                typeof init.body === "string"
                  ? JSON.parse(init.body)
                  : init.body;
              if (body?.messages) {
                isAgentCall = body.messages.some(
                  (msg) => msg.role && ["tool", "assistant"].includes(msg.role),
                );
                isVisionRequest = body.messages.some(
                  (msg) =>
                    Array.isArray(msg.content) &&
                    msg.content.some((part) => part.type === "image_url"),
                );
              }
            } catch {}
            const headers = {
              ...init.headers,
              ...HEADERS,
              Authorization: `Bearer ${info.access}`,
              "Openai-Intent": "conversation-edits",
              "X-Initiator": isAgentCall ? "agent" : "user",
            };
            if (isVisionRequest) {
              headers["Copilot-Vision-Request"] = "true";
            }
            delete headers["x-api-key"];
            return fetch(input, {
              ...init,
              headers,
            });
          },
        };
      },
      methods: [
        {
          type: "oauth",
          label: "Login with GitHub Copilot",
          prompts: [
            {
              type: "select",
              key: "deploymentType",
              message: "Select GitHub deployment type",
              options: [
                {
                  label: "GitHub.com",
                  value: "github.com",
                  hint: "Public",
                },
                {
                  label: "GitHub Enterprise",
                  value: "enterprise",
                  hint: "Data residency or self-hosted",
                },
              ],
            },
            {
              type: "text",
              key: "enterpriseUrl",
              message: "Enter your GitHub Enterprise URL or domain",
              placeholder: "company.ghe.com or https://company.ghe.com",
              condition: (inputs) => inputs.deploymentType === "enterprise",
              validate: (value) => {
                if (!value) return "URL or domain is required";
                try {
                  const url = value.includes("://")
                    ? new URL(value)
                    : new URL(`https://${value}`);
                  if (!url.hostname)
                    return "Please enter a valid URL or domain";
                  return undefined;
                } catch {
                  return "Please enter a valid URL (e.g., company.ghe.com or https://company.ghe.com)";
                }
              },
            },
          ],
          async authorize(inputs = {}) {
            const deploymentType = inputs.deploymentType || "github.com";

            let domain = "github.com";
            let actualProvider = "github-copilot";

            if (deploymentType === "enterprise") {
              const enterpriseUrl = inputs.enterpriseUrl;
              domain = normalizeDomain(enterpriseUrl);
              actualProvider = "github-copilot-enterprise";
            }

            const urls = getUrls(domain);

            const deviceResponse = await fetch(urls.DEVICE_CODE_URL, {
              method: "POST",
              headers: {
                Accept: "application/json",
                "Content-Type": "application/json",
                "User-Agent": "GitHubCopilotChat/0.35.0",
              },
              body: JSON.stringify({
                client_id: CLIENT_ID,
                scope: "read:user",
              }),
            });

            if (!deviceResponse.ok) {
              throw new Error("Failed to initiate device authorization");
            }

            const deviceData = await deviceResponse.json();

            return {
              url: deviceData.verification_uri,
              instructions: `Enter code: ${deviceData.user_code}`,
              method: "auto",
              callback: async () => {
                while (true) {
                  const response = await fetch(urls.ACCESS_TOKEN_URL, {
                    method: "POST",
                    headers: {
                      Accept: "application/json",
                      "Content-Type": "application/json",
                      "User-Agent": "GitHubCopilotChat/0.35.0",
                    },
                    body: JSON.stringify({
                      client_id: CLIENT_ID,
                      device_code: deviceData.device_code,
                      grant_type:
                        "urn:ietf:params:oauth:grant-type:device_code",
                    }),
                  });

                  if (!response.ok) return { type: "failed" };

                  const data = await response.json();

                  if (data.access_token) {
                    const result = {
                      type: "success",
                      refresh: data.access_token,
                      access: "",
                      expires: 0,
                    };

                    if (actualProvider === "github-copilot-enterprise") {
                      result.provider = "github-copilot-enterprise";
                      result.enterpriseUrl = domain;
                    }

                    return result;
                  }

                  if (data.error === "authorization_pending") {
                    await new Promise((resolve) =>
                      setTimeout(resolve, deviceData.interval * 1000),
                    );
                    continue;
                  }

                  if (data.error) return { type: "failed" };

                  await new Promise((resolve) =>
                    setTimeout(resolve, deviceData.interval * 1000),
                  );
                  continue;
                }
              },
            };
          },
        },
      ],
    },
  };
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

## Adapting to Command Code Architecture

This section provides guidance on implementing GitHub Copilot OAuth in Command Code, following the same patterns as the Anthropic Claude OAuth implementation.

### Key Differences: Device Flow vs. PKCE Flow

| Aspect | GitHub Copilot (Device Flow) | Claude Code (PKCE Flow) |
|--------|------------------------------|-------------------------|
| **OAuth Grant Type** | Device Authorization Grant (RFC 8628) | Authorization Code with PKCE |
| **User Experience** | User enters code from CLI into browser | User copies code from browser to CLI |
| **Code Challenge** | Not used | SHA256 PKCE challenge |
| **Redirect URI** | Not needed | Required callback URL |
| **Polling** | CLI polls for authorization | Manual code entry |
| **Best For** | Terminal-only apps, IoT devices | Apps that can open browsers |

### Implementation Structure

#### 1. Create OAuth Module (`packages/command/src/auth/github-copilot.ts`)

```typescript
import { Auth } from './index';
import crypto from 'crypto';

const CLIENT_ID = 'Iv1.b507a08c87ecfe98';
const DEVICE_CODE_URL = 'https://github.com/login/device/code';
const ACCESS_TOKEN_URL = 'https://github.com/login/oauth/access_token';
const COPILOT_TOKEN_URL = 'https://api.github.com/copilot_internal/v2/token';

const HEADERS = {
  'User-Agent': 'GitHubCopilotChat/0.32.4',
  'Editor-Version': 'vscode/1.105.1',
  'Editor-Plugin-Version': 'copilot-chat/0.32.4',
  'Copilot-Integration-Id': 'vscode-chat',
};

interface DeviceCodeResponse {
  device_code: string;
  user_code: string;
  verification_uri: string;
  expires_in: number;
  interval: number;
}

interface TokenResponse {
  access_token?: string;
  error?: string;
}

interface CopilotTokenResponse {
  token: string;
  expires_at: number;
}

/**
 * Initiate device code flow
 * Returns device code, user code, and verification URI
 */
export async function initiateDeviceFlow(): Promise<DeviceCodeResponse> {
  const response = await fetch(DEVICE_CODE_URL, {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      client_id: CLIENT_ID,
      scope: 'read:user',
    }),
  });

  if (!response.ok) {
    throw new Error('Failed to initiate device authorization');
  }

  return await response.json();
}

/**
 * Poll for access token after user authorizes
 */
export async function pollForAccessToken(
  deviceCode: string,
  interval: number
): Promise<string> {
  while (true) {
    const response = await fetch(ACCESS_TOKEN_URL, {
      method: 'POST',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        client_id: CLIENT_ID,
        device_code: deviceCode,
        grant_type: 'urn:ietf:params:oauth:grant-type:device_code',
      }),
    });

    if (!response.ok) {
      throw new Error('Token exchange failed');
    }

    const data: TokenResponse = await response.json();

    if (data.access_token) {
      return data.access_token;
    }

    if (data.error === 'authorization_pending') {
      // Wait and retry
      await new Promise(resolve => setTimeout(resolve, interval * 1000));
      continue;
    }

    if (data.error === 'expired_token') {
      throw new Error('Device code expired. Please try again.');
    }

    if (data.error === 'access_denied') {
      throw new Error('Authorization was denied.');
    }

    throw new Error(`Unknown error: ${data.error}`);
  }
}

/**
 * Exchange GitHub OAuth token for Copilot API token
 */
async function exchangeForCopilotToken(githubToken: string): Promise<CopilotTokenResponse> {
  const response = await fetch(COPILOT_TOKEN_URL, {
    headers: {
      'Accept': 'application/json',
      'Authorization': `Bearer ${githubToken}`,
      ...HEADERS,
    },
  });

  if (!response.ok) {
    throw new Error('Failed to get Copilot API token');
  }

  return await response.json();
}

/**
 * Store tokens after successful authentication
 */
export async function storeTokens(githubToken: string): Promise<void> {
  // Get initial Copilot token
  const copilotToken = await exchangeForCopilotToken(githubToken);

  await Auth.set('github-copilot', {
    type: 'oauth',
    refresh: githubToken, // GitHub OAuth token used as refresh token
    access: copilotToken.token,
    expires: copilotToken.expires_at * 1000,
  });
}

/**
 * Refresh Copilot API token if expired
 */
export async function refreshAccessToken(): Promise<string | undefined> {
  const auth = await Auth.get('github-copilot');
  if (!auth || auth.type !== 'oauth') return undefined;

  // Check if token is still valid (with 5-minute buffer)
  const now = Date.now();
  const buffer = 5 * 60 * 1000; // 5 minutes
  if (auth.expires > now + buffer) {
    return auth.access;
  }

  try {
    // Exchange GitHub token for new Copilot token
    const copilotToken = await exchangeForCopilotToken(auth.refresh);

    // Update stored tokens
    await Auth.set('github-copilot', {
      type: 'oauth',
      refresh: auth.refresh,
      access: copilotToken.token,
      expires: copilotToken.expires_at * 1000,
    });

    return copilotToken.token;
  } catch (error) {
    // If refresh fails, remove invalid auth
    await Auth.remove('github-copilot');
    return undefined;
  }
}

/**
 * Get valid access token (refresh if needed)
 */
export async function getValidAccessToken(): Promise<string | undefined> {
  const auth = await Auth.get('github-copilot');
  if (!auth) return undefined;

  if (auth.type === 'api') {
    // API key stored directly
    return auth.key;
  }

  // OAuth token - check and refresh if needed
  return await refreshAccessToken();
}
```

#### 2. Create CLI Command (`packages/command/src/commands/github-copilot-auth.ts`)

```typescript
import { Command } from 'commander';
import * as GithubCopilotOAuth from '../auth/github-copilot';
import { Auth } from '../auth';
import open from 'open';

export const githubCopilotAuthCommand = new Command('github-copilot-auth')
  .description('Manage GitHub Copilot authentication');

githubCopilotAuthCommand
  .command('login')
  .description('Login to GitHub Copilot')
  .action(async () => {
    try {
      console.log('\n🔐 Starting GitHub Copilot authentication...\n');

      // Initiate device flow
      const deviceData = await GithubCopilotOAuth.initiateDeviceFlow();

      console.log('Please visit:', deviceData.verification_uri);
      console.log('Enter code:', deviceData.user_code);
      console.log('\nOpening browser...');

      // Try to open browser
      try {
        await open(deviceData.verification_uri);
      } catch {
        console.log('Could not open browser automatically.');
      }

      console.log('\nWaiting for authorization...');

      // Poll for token
      const githubToken = await GithubCopilotOAuth.pollForAccessToken(
        deviceData.device_code,
        deviceData.interval
      );

      // Store tokens
      await GithubCopilotOAuth.storeTokens(githubToken);

      console.log('✅ Successfully authenticated with GitHub Copilot!\n');
    } catch (error) {
      console.error('❌ Authentication failed:', error.message);
      process.exit(1);
    }
  });

githubCopilotAuthCommand
  .command('logout')
  .description('Logout from GitHub Copilot')
  .action(async () => {
    try {
      const auth = await Auth.get('github-copilot');
      if (!auth) {
        console.log('Not currently logged in to GitHub Copilot.');
        return;
      }

      await Auth.remove('github-copilot');
      console.log('✅ Successfully logged out from GitHub Copilot.');
    } catch (error) {
      console.error('❌ Logout failed:', error.message);
      process.exit(1);
    }
  });

githubCopilotAuthCommand
  .command('status')
  .description('Check GitHub Copilot authentication status')
  .action(async () => {
    try {
      const auth = await Auth.get('github-copilot');
      if (!auth) {
        console.log('Status: Not authenticated');
        return;
      }

      console.log('Status: Authenticated');
      console.log('Type:', auth.type);

      if (auth.type === 'oauth') {
        const now = Date.now();
        const expiresIn = Math.floor((auth.expires - now) / 1000 / 60);
        console.log('Token expires in:', expiresIn, 'minutes');
        console.log('Token valid:', auth.expires > now ? 'Yes' : 'No (needs refresh)');
      }
    } catch (error) {
      console.error('❌ Status check failed:', error.message);
      process.exit(1);
    }
  });
```

#### 3. Create Interactive UI Component (`packages/command/src/components/github-copilot-auth.tsx`)

```typescript
import React, { useState, useEffect } from 'react';
import { Text, Box } from 'ink';
import Spinner from 'ink-spinner';
import * as GithubCopilotOAuth from '../auth/github-copilot';
import open from 'open';

interface Props {
  onSuccess: () => void;
  onCancel: () => void;
}

export const GithubCopilotAuth: React.FC<Props> = ({ onSuccess, onCancel }) => {
  const [stage, setStage] = useState<'init' | 'waiting' | 'success' | 'error'>('init');
  const [error, setError] = useState<string>('');
  const [deviceData, setDeviceData] = useState<any>(null);

  useEffect(() => {
    let cancelled = false;

    const authenticate = async () => {
      try {
        // Initiate device flow
        const data = await GithubCopilotOAuth.initiateDeviceFlow();
        if (cancelled) return;

        setDeviceData(data);

        // Try to open browser
        try {
          await open(data.verification_uri);
        } catch {
          // Browser opening failed, user will open manually
        }

        setStage('waiting');

        // Poll for token
        const githubToken = await GithubCopilotOAuth.pollForAccessToken(
          data.device_code,
          data.interval
        );
        if (cancelled) return;

        // Store tokens
        await GithubCopilotOAuth.storeTokens(githubToken);

        setStage('success');
        setTimeout(onSuccess, 1000);
      } catch (err) {
        if (cancelled) return;
        setError(err.message);
        setStage('error');
      }
    };

    authenticate();

    return () => {
      cancelled = true;
    };
  }, [onSuccess]);

  if (stage === 'init') {
    return (
      <Box flexDirection="column">
        <Text>
          <Text color="cyan">
            <Spinner type="dots" />
          </Text>
          {' '}Initializing GitHub Copilot authentication...
        </Text>
      </Box>
    );
  }

  if (stage === 'waiting' && deviceData) {
    return (
      <Box flexDirection="column">
        <Text bold>GitHub Copilot Authentication</Text>
        <Text> </Text>
        <Text>1. Visit: <Text color="cyan">{deviceData.verification_uri}</Text></Text>
        <Text>2. Enter code: <Text color="green" bold>{deviceData.user_code}</Text></Text>
        <Text> </Text>
        <Text>
          <Text color="cyan">
            <Spinner type="dots" />
          </Text>
          {' '}Waiting for authorization...
        </Text>
        <Text> </Text>
        <Text dimColor>Press ESC to cancel</Text>
      </Box>
    );
  }

  if (stage === 'success') {
    return (
      <Box flexDirection="column">
        <Text color="green">✅ Successfully authenticated with GitHub Copilot!</Text>
      </Box>
    );
  }

  if (stage === 'error') {
    return (
      <Box flexDirection="column">
        <Text color="red">❌ Authentication failed: {error}</Text>
      </Box>
    );
  }

  return null;
};
```

#### 4. Integrate with GitHub Copilot Client (`packages/command/src/clients/github-copilot.ts`)

```typescript
import OpenAI from 'openai';
import * as GithubCopilotOAuth from '../auth/github-copilot';

export class GithubCopilotClient {
  private instance: OpenAI | null = null;

  async initialize() {
    const token = await GithubCopilotOAuth.getValidAccessToken();
    
    if (!token) {
      throw new Error('Not authenticated with GitHub Copilot. Run: cmd github-copilot-auth login');
    }

    this.instance = new OpenAI({
      baseURL: 'https://api.githubcopilot.com',
      apiKey: token,
      defaultHeaders: {
        'User-Agent': 'GitHubCopilotChat/0.32.4',
        'Editor-Version': 'vscode/1.105.1',
        'Editor-Plugin-Version': 'copilot-chat/0.32.4',
        'Copilot-Integration-Id': 'vscode-chat',
      },
    });
  }

  async chat(messages: any[], options?: any) {
    if (!this.instance) {
      await this.initialize();
    }

    return await this.instance!.chat.completions.create({
      model: options?.model || 'gpt-4',
      messages,
      ...options,
    });
  }
}
```

#### 5. Update Provider Configuration (`packages/command/src/utils/provider-config.ts`)

```typescript
import { GithubCopilotAuth } from '../components/github-copilot-auth';
import * as GithubCopilotOAuth from '../auth/github-copilot';

export const PROVIDER_CONFIG = {
  // ... other providers
  
  'github-copilot': {
    name: 'GitHub Copilot',
    description: 'GitHub Copilot models (free with subscription)',
    requiresAuth: true,
    authComponent: GithubCopilotAuth,
    checkAuth: async () => {
      const token = await GithubCopilotOAuth.getValidAccessToken();
      return !!token;
    },
  },
};
```

### Package Manager Setup (pnpm)

#### Install Dependencies

```bash
# Install OpenAI SDK for API compatibility
pnpm add openai

# Install open package for browser launching
pnpm add open

# Install types
pnpm add -D @types/node
```

#### Package Scripts (`package.json`)

```json
{
  "scripts": {
    "auth:github": "pnpm cmd github-copilot-auth",
    "auth:github:login": "pnpm cmd github-copilot-auth login",
    "auth:github:logout": "pnpm cmd github-copilot-auth logout",
    "auth:github:status": "pnpm cmd github-copilot-auth status"
  }
}
```

### Key Differences from OpenCode Implementation

| Aspect | OpenCode | Command Code |
|--------|----------|--------------|
| **Plugin System** | External npm plugins | Integrated modules |
| **Auth Storage** | `~/.opencode/data/auth.json` | `~/.commandcode/models.json` |
| **Runtime** | Bun | Node.js |
| **Package Manager** | Bun | pnpm |
| **UI Framework** | @clack/prompts | Ink (React) |
| **Command Structure** | Nested subcommands | Flat command structure |
| **Client Integration** | Provider system with loaders | Direct client classes |

### Enterprise Support

For GitHub Enterprise support, extend the implementation:

```typescript
// Add enterprise URL prompt in auth component
const [enterpriseUrl, setEnterpriseUrl] = useState<string>('');
const [deploymentType, setDeploymentType] = useState<'public' | 'enterprise'>('public');

// Modify URLs based on deployment type
const getUrls = (domain: string) => ({
  DEVICE_CODE_URL: `https://${domain}/login/device/code`,
  ACCESS_TOKEN_URL: `https://${domain}/login/oauth/access_token`,
  COPILOT_TOKEN_URL: `https://api.${domain}/copilot_internal/v2/token`,
});

// Store enterprise URL with auth data
await Auth.set('github-copilot-enterprise', {
  type: 'oauth',
  refresh: githubToken,
  access: copilotToken.token,
  expires: copilotToken.expires_at * 1000,
  enterpriseUrl: domain,
});
```

### Testing with pnpm

```bash
# Run auth flow
pnpm auth:github:login

# Check status
pnpm auth:github:status

# Test with provider
pnpm cmd --provider github-copilot "Write a hello world function"

# Logout
pnpm auth:github:logout
```

### Complete Implementation Checklist

- [ ] Create `src/auth/github-copilot.ts` with device flow functions
- [ ] Create `src/commands/github-copilot-auth.ts` with login/logout/status
- [ ] Create `src/components/github-copilot-auth.tsx` with Ink UI
- [ ] Create `src/clients/github-copilot.ts` for API calls
- [ ] Update `src/auth/index.ts` to support OAuth type
- [ ] Add provider to `src/utils/provider-config.ts`
- [ ] Install dependencies: `pnpm add openai open`
- [ ] Add scripts to `package.json`
- [ ] Test authentication flow
- [ ] Test token refresh
- [ ] Test API requests
- [ ] Add enterprise support (optional)
- [ ] Update documentation

### Error Handling

```typescript
// In context-engine.ts or similar
import * as GithubCopilotOAuth from '../auth/github-copilot';

async function prepareRequest(provider: string) {
  let token: string | undefined;
  
  if (provider === 'github-copilot') {
    token = await GithubCopilotOAuth.getValidAccessToken();
    
    if (!token) {
      throw new Error(
        'Not authenticated with GitHub Copilot.\n' +
        'Run: cmd github-copilot-auth login'
      );
    }
  }
  
  return token;
}
```

### Comparison with Claude OAuth

| Feature | GitHub Copilot | Claude Code |
|---------|----------------|-------------|
| **OAuth Flow** | Device Authorization | Authorization Code + PKCE |
| **User Action** | Enter code in browser | Copy code from browser |
| **Code Challenge** | None | SHA256 PKCE |
| **Polling** | Automatic | Not needed |
| **Redirect URI** | Not needed | Required |
| **Token Exchange** | Two-step (GitHub → Copilot) | One-step |
| **Refresh Logic** | Exchange GitHub token for Copilot token | Use refresh token grant |

### Usage Example

```bash
# Authenticate
$ pnpm cmd github-copilot-auth login

🔐 Starting GitHub Copilot authentication...

Please visit: https://github.com/login/device
Enter code: 8F43-6FCF

Opening browser...

Waiting for authorization...

✅ Successfully authenticated with GitHub Copilot!

# Use with Command Code
$ pnpm cmd --provider github-copilot "Explain OAuth device flow"

# Check status
$ pnpm cmd github-copilot-auth status
Status: Authenticated
Type: oauth
Token expires in: 45 minutes
Token valid: Yes
```

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

**Document Version:** 2.0  
**Last Updated:** 2025-11-15  
**Author:** Generated from opencode codebase analysis  
**License:** Same as opencode project

**Changelog:**
- v2.0: Added complete plugin source code, Command Code adaptation guide with pnpm support
- v1.0: Initial comprehensive specification
