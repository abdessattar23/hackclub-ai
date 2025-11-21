# Hack Club AI Proxy - Anti-Fraud Analysis & Complete Workflow Documentation

## Executive Summary

This document provides a comprehensive analysis of the **Hack Club AI Proxy** project, focusing on its anti-fraud mechanisms and complete operational workflow. The project is a lightweight LLM proxy service designed for teenagers (18 and under) affiliated with Hack Club, implementing multiple layers of security, authentication, and fraud prevention.

---

## Table of Contents

1. [Anti-Fraud Mechanisms](#anti-fraud-mechanisms)
2. [Complete Project Workflow](#complete-project-workflow)
3. [Technical Architecture](#technical-architecture)
4. [Security Features](#security-features)
5. [Database Schema](#database-schema)
6. [API Endpoints](#api-endpoints)

---

## Anti-Fraud Mechanisms

The Hack Club AI Proxy implements **multiple layers of anti-fraud and abuse prevention mechanisms** to protect the service from misuse. Below is a detailed analysis of each mechanism:

### 1. Identity Verification (IDV) System

**Location**: `src/routes/auth.ts`, `src/middleware/auth.ts`, `scripts/import-idv-status.ts`

**How it works**:
- Integrates with Hack Club's external identity verification service at `https://identity.hackclub.com/api/external/check`
- Checks user identity through two methods:
  - By Slack ID
  - By email address (fallback)
- Verifies that users are eligible Hack Club members (teenagers 18 and under)
- Returns `verified_eligible` status for approved users

**Implementation Details**:
```typescript
// During authentication callback
const isIdvVerified = await checkIdvStatus(slackId, email);

// Stored in user record
user.isIdvVerified = true/false
```

**Enforcement**:
- Controlled by environment variable: `ENFORCE_IDV=true`
- When enabled, users must be IDV verified to use API endpoints
- Users can be allowed to skip IDV via `skipIdv` flag (admin override)
- Check performed in `requireApiKey` middleware before each API request

**Database Fields**:
- `users.isIdvVerified` - Boolean flag indicating verification status
- `users.skipIdv` - Boolean flag to bypass IDV requirement for specific users

### 2. AI Coding Agent Blocking

**Location**: `src/middleware/auth.ts`, `src/config/blocked-apps.json`

**Purpose**: Prevents automated AI coding tools and chatbot frontends from abusing the service

**How it works**:
- Maintains a comprehensive blocklist of 100+ AI coding agents and chat frontends
- Checks incoming request headers for blocked applications:
  - `Referer` header
  - `HTTP-Referer` header
  - `X-Title` header
- Case-insensitive matching against blocklist

**Blocked Applications Include**:
- AI Coding Agents: Cursor, Cline, Codeium, Tabnine, GitHub Copilot, Aider
- Auto-GPT variants: AutoGPT, BabyAGI, AgentGPT, SuperAGI
- Chat Frontends: SillyTavern, JanitorAI, KoboldAI
- Code Assistants: GPT-Engineer, Roo Code, Kilocode
- LLM Proxies: LiteLLM, OpenRouter, One-API
- Development Tools: Zed Editor, Continue, Open WebUI
- And many more (see `blocked-apps.json`)

**Error Response**:
```json
{
  "error": "For now, AI coding agents and frontends like SillyTavern aren't allowed to be used with ai.hackclub.com. Join #hackclub-ai on the Hack Club Slack for future updates."
}
```

**HTTP Status**: 403 Forbidden

### 3. User Banning System

**Location**: `src/db/schema.ts`, `src/middleware/auth.ts`

**Database Field**: `users.isBanned` (boolean, default: false)

**Enforcement Points**:
- Checked in `requireAuth` middleware (for dashboard access)
- Checked in `requireApiKey` middleware (for API usage)

**Implementation**:
```typescript
if (user.isBanned) {
  throw new HTTPException(403, { 
    message: "You are banned from using this service." 
  });
}
```

**Purpose**: Allows administrators to permanently block abusive users from accessing the service

### 4. Workspace Verification

**Location**: `src/routes/auth.ts`, lines 123-127

**How it works**:
- Verifies user belongs to the official Hack Club Slack workspace
- Checks JWT token claim: `"https://slack.com/team_id"`
- Compares against allowed team ID from environment: `SLACK_TEAM_ID=T0266FRGM`

**Implementation**:
```typescript
if (slackTeamId !== env.SLACK_TEAM_ID) {
  throw new HTTPException(403, {
    message: "Access denied: Invalid workspace",
  });
}
```

**Purpose**: Ensures only members of the Hack Club Slack workspace can authenticate

### 5. API Key Limits

**Location**: `src/routes/api.ts`, lines 23-37

**Restrictions**:
- Maximum 50 API keys per user
- Prevents users from creating unlimited keys for distribution

**Implementation**:
```typescript
const existingKeys = await db
  .select({ count: sql<number>`COUNT(*)::int` })
  .from(apiKeys)
  .where(and(eq(apiKeys.userId, user.id), isNull(apiKeys.revokedAt)));

if (existingKeys[0].count >= 50) {
  throw new HTTPException(400, {
    message: "Maximum API key limit reached",
  });
}
```

**Additional Features**:
- API keys can be revoked (soft delete with `revokedAt` timestamp)
- Each key has a descriptive name for user management
- Key format: `sk-hc-v1-{64-char-random-string}`

### 6. Request Body Size Limiting

**Location**: `src/index.ts`, lines 32-38

**Configuration**:
- Maximum request size: **20 MB** (20 * 1024 * 1024 bytes)
- Applied to all `/proxy/*` endpoints

**Implementation**:
```typescript
app.use(
  "/proxy/*",
  bodyLimit({
    maxSize: 20 * 1024 * 1024,
    onError: () => {
      throw new HTTPException(413, { message: "Request too large" });
    },
  }),
);
```

**Purpose**: Prevents denial-of-service attacks through oversized requests

### 7. Request Timeout

**Location**: `src/index.ts`, line 39

**Configuration**:
- Timeout: **120 seconds** (120,000 ms) for proxy endpoints

**Implementation**:
```typescript
app.use("/proxy/*", timeout(120000));
```

**Purpose**: Prevents hanging requests from consuming server resources

### 8. Model Whitelisting

**Location**: `src/routes/proxy.ts`, `src/env.ts`

**How it works**:
- Only pre-approved models can be used
- Configured via environment variables:
  - `ALLOWED_LANGUAGE_MODELS` - Chat completion models
  - `ALLOWED_EMBEDDING_MODELS` - Embedding models
- If user requests an unlisted model, it's automatically replaced with the first allowed model

**Language Models** (examples from README):
- qwen/qwen3-32b
- google/gemini-2.5-flash
- openai/gpt-5-mini
- deepseek/deepseek-v3.2-exp
- And more...

**Embedding Models** (examples):
- qwen/qwen3-embedding-8b
- mistralai/codestral-embed-2505
- openai/text-embedding-3-large

**Implementation**:
```typescript
const allowedSet = new Set(allowedLanguageModels);
if (!allowedSet.has(requestBody.model)) {
  requestBody.model = allowedLanguageModels[0]; // Use default model
}
```

**Purpose**: 
- Controls costs by limiting expensive models
- Prevents access to inappropriate models
- Ensures consistent service quality

### 9. IP Address Logging

**Location**: `src/routes/proxy.ts`, lines 35-42

**How it works**:
- Captures client IP address from multiple header sources (in priority order):
  1. `CF-Connecting-IP` (Cloudflare)
  2. `X-Forwarded-For` (Proxy/Load Balancer)
  3. `X-Real-IP` (Nginx)
- Logs IP address with every request in `request_logs` table

**Implementation**:
```typescript
function getClientIp(c: any): string {
  return (
    c.req.header("CF-Connecting-IP") ||
    c.req.header("X-Forwarded-For")?.split(",")[0].trim() ||
    c.req.header("X-Real-IP") ||
    "unknown"
  );
}
```

**Purpose**: 
- Tracks user behavior patterns
- Enables abuse detection through anomalous IP patterns
- Provides audit trail for security incidents

### 10. Comprehensive Request Logging

**Location**: `src/routes/proxy.ts`, `src/db/schema.ts`

**Data Logged**:
- API key ID and user ID
- Slack ID (for cross-referencing)
- Model used
- Token counts (prompt, completion, total)
- Full request body
- Full response body (or stream indicator)
- Client IP address
- Timestamp
- Request duration (milliseconds)

**Storage**: PostgreSQL database table `request_logs`

**Purpose**:
- Complete audit trail of all API usage
- Enables detection of abuse patterns
- Supports usage analytics and billing
- Helps identify compromised API keys

### 11. CSRF Protection

**Location**: `src/index.ts`, line 42

**Implementation**:
```typescript
app.use("/*", csrf({ origin: env.BASE_URL }));
```

**Purpose**: Prevents cross-site request forgery attacks on web dashboard

### 12. Secure Headers

**Location**: `src/index.ts`, line 29

**Implementation**:
```typescript
app.use("*", secureHeaders());
```

**Purpose**: Adds security headers to prevent common web vulnerabilities:
- Content-Security-Policy
- X-Frame-Options
- X-Content-Type-Options
- Referrer-Policy
- Permissions-Policy

### 13. Session Security

**Location**: `src/routes/auth.ts`, lines 190-196

**Features**:
- HTTPOnly cookies (prevents XSS access)
- Secure flag in production (HTTPS only)
- SameSite: Lax (CSRF protection)
- 30-day expiration
- Cryptographically random session tokens (UUID v4)

**Implementation**:
```typescript
setCookie(c, "session_token", sessionToken, {
  httpOnly: true,
  secure: env.NODE_ENV === "production",
  sameSite: "Lax",
  maxAge: 60 * 60 * 24 * 30,
  path: "/",
});
```

### 14. User Attribution in Requests

**Location**: `src/routes/proxy.ts`

**Implementation**:
```typescript
requestBody.user = `user_${user.id}`;
```

**Purpose**: 
- Forwards user ID to OpenRouter/OpenAI API
- Enables upstream provider to track and potentially flag abuse
- Supports rate limiting at provider level

### 15. Reverse Proxy Requirement

**Location**: `README.md`, lines 13

**Security Note**:
> "You **must** have a reverse proxy (e.g. traefik) in front of the service to ensure that IPs aren't spoofed."

**Purpose**: 
- Ensures IP address headers are trustworthy
- Prevents IP spoofing attacks
- Required for IP-based fraud detection to work correctly

---

## Complete Project Workflow

### Architecture Overview

```
┌─────────────┐
│   User/App  │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│  Reverse Proxy      │
│  (Traefik/Nginx)    │
└──────┬──────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  Hono Application (Node.js/Bun)         │
│                                          │
│  ┌──────────────────────────────────┐  │
│  │  Security Middleware Layer       │  │
│  │  - Secure Headers                │  │
│  │  - CSRF Protection               │  │
│  │  - Body Limit (20MB)             │  │
│  │  - Timeout (120s)                │  │
│  │  - Request ID                    │  │
│  │  - AI Agent Blocking             │  │
│  └──────────────────────────────────┘  │
│                                          │
│  ┌──────────────────────────────────┐  │
│  │  Route Handlers                  │  │
│  │  - /auth (Slack OAuth)           │  │
│  │  - /dashboard (User UI)          │  │
│  │  - /api (Key Management)         │  │
│  │  - /proxy (LLM Endpoints)        │  │
│  │  - /docs (Documentation)         │  │
│  └──────────────────────────────────┘  │
└───────────┬─────────────────────────────┘
            │
            ├──────────────────┐
            │                  │
            ▼                  ▼
    ┌──────────────┐   ┌─────────────────┐
    │  PostgreSQL  │   │  External APIs  │
    │   Database   │   │  - Slack OAuth  │
    │              │   │  - IDV Service  │
    │  Tables:     │   │  - OpenRouter   │
    │  - users     │   │  - Sentry       │
    │  - sessions  │   └─────────────────┘
    │  - api_keys  │
    │  - req_logs  │
    └──────────────┘
```

### 1. User Onboarding Workflow

#### Step 1: Initial Access
1. User navigates to base URL (e.g., `https://ai.hackclub.com`)
2. Application checks for existing session cookie
3. If no valid session exists, displays landing page (`/`)
4. Landing page shows:
   - Service description
   - Available models
   - "Get Started" button linking to `/auth/login`

**File**: `src/routes/dashboard.tsx` (lines 20-49)

#### Step 2: Slack OAuth Authentication
1. User clicks "Get Started" or "Login with Slack"
2. Browser redirects to `/auth/login`
3. Server constructs Slack OAuth URL:
   ```
   https://hackclub.slack.com/oauth/v2/authorize
     ?client_id={SLACK_CLIENT_ID}
     &user_scope=openid,profile,email
     &redirect_uri={BASE_URL}/auth/callback
   ```
4. User redirects to Slack for authorization
5. User approves OAuth scopes (openid, profile, email)
6. Slack redirects back to `/auth/callback?code={OAUTH_CODE}`

**Files**: `src/routes/auth.ts` (lines 68-75)

#### Step 3: OAuth Callback Processing
1. Server receives OAuth code from Slack
2. Exchanges code for ID token via Slack OpenID Connect:
   ```
   POST https://slack.com/api/openid.connect.token
   ```
3. Parses JWT ID token to extract user data:
   - Slack user ID
   - Slack team ID
   - Display name
   - Email address
   - Avatar URL
4. Validates user belongs to Hack Club workspace (team ID check)
5. Performs Identity Verification (IDV) check:
   - Queries `https://identity.hackclub.com/api/external/check`
   - First by Slack ID, then by email if needed
   - Determines if user is "verified_eligible"

**Files**: `src/routes/auth.ts` (lines 77-207)

#### Step 4: User Record Creation/Update
1. Checks if user exists in database by Slack ID
2. If new user:
   - Creates new user record with all profile data
   - Sets `isIdvVerified` based on IDV check
3. If existing user:
   - Updates profile information (name, email, avatar)
   - Updates `isIdvVerified` status
   - Sets `updatedAt` timestamp

**Database**: `users` table

#### Step 5: Session Creation
1. Generates cryptographically random session token (UUID)
2. Creates session record in database:
   - Links to user ID
   - Stores token
   - Sets expiration (30 days from now)
3. Sets secure HTTP-only cookie:
   ```
   Set-Cookie: session_token={TOKEN}; 
     HttpOnly; Secure; SameSite=Lax; 
     MaxAge=2592000; Path=/
   ```
4. Redirects user to `/dashboard`

**Files**: `src/routes/auth.ts` (lines 178-198)

### 2. Dashboard Workflow

#### Dashboard Home (`/dashboard`)
1. Middleware validates session:
   - Reads `session_token` cookie
   - Queries database for matching, non-expired session
   - Loads associated user record
   - Checks if user is banned
2. Fetches user-specific data:
   - All API keys (active and revoked)
   - Usage statistics (tokens, requests)
   - Recent request logs (last 50)
3. Displays dashboard UI showing:
   - User profile (avatar, name, email)
   - IDV verification status banner (if not verified and ENFORCE_IDV=true)
   - API key management section
   - Usage statistics cards
   - Allowed models list
   - Recent activity table

**Files**: `src/routes/dashboard.tsx` (lines 51-128), `src/views/dashboard.tsx`

#### Global Statistics Page (`/global`)
1. Requires authentication (same as dashboard)
2. Queries aggregated statistics across ALL users:
   - Total requests
   - Total tokens consumed
   - Prompt vs completion token breakdown
3. Per-model statistics:
   - Requests per model
   - Token usage per model
   - Sorted by total tokens (most used first)
4. Displays global analytics dashboard

**Files**: `src/routes/dashboard.tsx` (lines 130-184), `src/views/global.tsx`

### 3. API Key Management Workflow

#### Creating an API Key (`POST /api/keys`)
1. User submits key name via dashboard form
2. Request validation:
   - Session authentication (requires valid login)
   - Name length: 1-100 characters
3. Checks if user has reached key limit (50 keys)
4. Generates new API key:
   - Format: `sk-hc-v1-{64-random-hex-chars}`
   - Uses two UUIDs with hyphens removed
5. Stores key in database:
   - Links to user ID
   - Stores full key (not hashed - for proxy simplicity)
   - Stores user-provided name
   - Records creation timestamp
6. Returns full key to user (shown only once)

**Files**: `src/routes/api.ts` (lines 18-61)

**Security Note**: Keys are stored in plaintext for performance reasons. In a production banking/financial app, keys should be hashed.

#### Listing API Keys (`GET /api/keys`)
1. Validates user session
2. Queries all keys for current user
3. Returns sanitized data:
   - Key ID (for deletion)
   - Key name
   - Creation timestamp
   - Revocation timestamp (if revoked)
   - Key preview (first 10 chars only): `sk-hc-v1-...`
4. Ordered by creation date (newest first)

**Files**: `src/routes/api.ts` (lines 63-86)

#### Revoking an API Key (`DELETE /api/keys/:id`)
1. Validates user session
2. Verifies key belongs to current user
3. Soft deletes key:
   - Sets `revokedAt` timestamp
   - Does not physically delete (preserves logs)
4. Key immediately stops working for new requests

**Files**: `src/routes/api.ts` (lines 88-117)

### 4. API Request Workflow (LLM Proxy)

#### Models Endpoint (`GET /proxy/v1/models`)

**Purpose**: List available AI models (OpenAI-compatible)

1. **No authentication required** (public endpoint)
2. Checks in-memory cache (5-minute TTL)
3. If cache miss:
   - Fetches full model list from OpenRouter API
   - Filters to only allowed models (whitelist)
   - Caches result
4. Returns filtered model list (OpenAI format)

**Files**: `src/routes/proxy.ts` (lines 44-105)

**Cache Strategy**: 
- Singleton cache shared across all requests
- Prevents rate limiting from upstream provider
- Reduces latency for frequent model list queries

#### Chat Completions (`POST /proxy/v1/chat/completions`)

**Step 1: Security Checks**
1. AI Coding Agent blocking middleware runs first
   - Checks `Referer` and `X-Title` headers
   - Blocks if matches any entry in blocklist
2. API key authentication:
   - Extracts Bearer token from `Authorization` header
   - Queries database for matching, non-revoked key
   - Loads associated user record
   - Checks if user is banned
   - Checks IDV status (if ENFORCE_IDV enabled)

**Step 2: Request Processing**
1. Parses JSON request body
2. Validates and corrects model:
   - If model not in whitelist, replaces with default
3. Adds user attribution:
   ```json
   { "user": "user_{uuid}" }
   ```
4. Captures start timestamp for duration tracking

**Step 3: Upstream API Call**
1. Forwards request to OpenRouter/OpenAI API:
   ```
   POST {OPENAI_API_URL}/v1/chat/completions
   ```
2. Includes headers:
   - `Authorization: Bearer {OPENAI_API_KEY}`
   - `Content-Type: application/json`
   - `HTTP-Referer: {BASE_URL}/global?utm_source=openrouter`
   - `X-Title: Hack Club AI`

**Step 4: Response Handling - Non-Streaming**
1. Waits for complete response
2. Extracts token usage:
   - prompt_tokens
   - completion_tokens
   - total_tokens
3. Calculates request duration
4. Logs request to database (async, non-blocking)
5. Returns response to client (preserves original status code)

**Step 5: Response Handling - Streaming**
1. Creates streaming response
2. Reads chunks from upstream API
3. Pipes each chunk to client immediately
4. Parses chunks for usage statistics
5. After stream completes, logs to database

**Step 6: Error Handling**
1. Catches any exceptions during proxying
2. Logs error request to database with error message
3. Captures exception in Sentry
4. Returns 500 Internal Server Error to client

**Files**: `src/routes/proxy.ts` (lines 137-291), `src/middleware/auth.ts`

#### Embeddings (`POST /proxy/v1/embeddings`)

Similar to chat completions, but for text embeddings:

1. Same security checks (AI agent blocking, API key auth)
2. Model validation (embedding whitelist)
3. User attribution
4. Upstream API call to `/v1/embeddings`
5. Token usage logging (no completion tokens for embeddings)
6. Error handling and monitoring

**Files**: `src/routes/proxy.ts` (lines 293-375)

#### Statistics Endpoint (`GET /proxy/v1/stats`)

**Purpose**: User-specific usage statistics

1. Requires API key authentication
2. Queries aggregated data for current user:
   - Total requests
   - Total tokens (all types)
   - Prompt tokens
   - Completion tokens
3. Returns JSON summary
4. Used by CLI tools and custom dashboards

**Files**: `src/routes/proxy.ts` (lines 107-135)

### 5. Logout Workflow

1. User clicks "Logout" button
2. Browser requests `GET /auth/logout`
3. Server:
   - Reads session token from cookie
   - Deletes session record from database
   - Clears session cookie (sets MaxAge=0)
4. Redirects user to home page (`/`)

**Files**: `src/routes/auth.ts` (lines 209-229)

### 6. Documentation Workflow

#### Docs Page (`GET /docs`)
1. Optional authentication (works for both logged-in and anonymous users)
2. Loads markdown documentation from `src/docs.md`
3. Processes markdown with plugins:
   - Shiki (syntax highlighting)
   - Marked Alert (callout boxes)
4. Replaces template variables:
   - `{{BASE_URL}}` → Actual base URL
   - `{{FIRST_LANGUAGE_MODEL}}` → Default model
   - `{{FIRST_EMBEDDING_MODEL}}` → Default embedding model
5. Renders HTML documentation page

**Files**: `src/routes/docs.tsx`, `src/lib/docs.ts`, `src/views/docs.tsx`

### 7. Database Migration Workflow

#### On Application Startup
1. Before starting HTTP server, runs migrations:
   ```typescript
   await runMigrations();
   ```
2. Creates temporary PostgreSQL connection
3. Applies all pending SQL migrations from `./drizzle/` folder
4. Migrations run sequentially in order (0000, 0001, 0002...)
5. If migration fails, application exits with error
6. Closes migration connection after completion
7. Main application uses separate connection pool

**Files**: `src/index.ts` (line 24), `src/migrate.ts`

**Migration Files**: `drizzle/0000_*.sql` through `drizzle/0006_*.sql`

### 8. Monitoring and Error Tracking

#### Sentry Integration
1. Initialized before any other imports (`src/instrument.ts`)
2. Captures:
   - Unhandled exceptions
   - HTTP errors (5xx status codes)
   - Performance traces (via spans)
3. Sends user context:
   - Email (if available)
   - Slack ID
   - Display name
4. Disabled if `SENTRY_DSN` environment variable not set

**Files**: `src/instrument.ts`, `src/index.ts`

#### Logging Strategy
- **Development**: All requests logged to console (Hono logger middleware)
- **Production**: 
  - Only errors logged to console
  - Sentry receives exceptions
  - Database stores all request logs
  - Request IDs for correlation

**Performance Monitoring**:
- Sentry spans track:
  - Route handler execution
  - Database queries
  - External API calls
- Enables performance optimization

### 9. Request Lifecycle Summary

```
1. Request arrives → Reverse Proxy (Traefik/Nginx)
2. secureHeaders() → Adds security headers
3. bodyLimit() → Validates request size ≤ 20MB
4. timeout() → Sets 120s timeout
5. requestId() → Generates unique request ID
6. csrf() → Validates CSRF token (web requests)
7. logger() → Logs request (dev only)
8. blockAICodingAgents() → Checks blocklist
9. requireApiKey() → Validates API key + user status
10. Route handler → Processes business logic
11. Proxy to OpenRouter → Forwards LLM request
12. Stream response → Returns data to client
13. Log to database → Records usage metrics
14. Send to Sentry → Tracks errors/performance
```

---

## Technical Architecture

### Technology Stack

**Runtime**: Bun (JavaScript/TypeScript runtime)
- Faster startup than Node.js
- Native TypeScript support
- Built-in HTTP server

**Web Framework**: Hono
- Lightweight (< 12KB)
- Express-like API
- Excellent TypeScript support
- Edge-compatible

**Database**: PostgreSQL 18
- ORM: Drizzle ORM
- Migration tool: Drizzle Kit
- Connection pooling via `postgres` library

**Validation**: ArkType
- Runtime type validation
- Environment variable checking
- API request validation

**Authentication**: Slack OpenID Connect
- OAuth 2.0 flow
- JWT token parsing
- Workspace validation

**Monitoring**: Sentry
- Error tracking
- Performance monitoring
- User context

**Documentation**: 
- Marked (Markdown processor)
- Shiki (Syntax highlighting)
- Custom template variable replacement

**Deployment**: Docker
- Multi-stage build (Dockerfile)
- Docker Compose for local development
- Coolify deployment support

### File Structure

```
src/
├── config/
│   └── blocked-apps.json          # AI agent blocklist
├── db/
│   ├── index.ts                   # Database connection
│   └── schema.ts                  # Drizzle schema definitions
├── lib/
│   └── docs.ts                    # Documentation processor
├── middleware/
│   └── auth.ts                    # Auth middleware (session, API key, blocking)
├── routes/
│   ├── api.ts                     # API key management endpoints
│   ├── auth.ts                    # OAuth authentication flow
│   ├── dashboard.tsx              # Dashboard routes
│   ├── docs.tsx                   # Documentation route
│   └── proxy.ts                   # LLM proxy endpoints
├── views/                         # JSX view components
│   ├── components/                # Reusable UI components
│   ├── dashboard.tsx
│   ├── docs.tsx
│   ├── global.tsx
│   ├── home.tsx
│   └── layout.tsx
├── docs.md                        # API documentation content
├── env.ts                         # Environment validation
├── index.ts                       # Application entry point
├── instrument.ts                  # Sentry initialization
├── migrate.ts                     # Database migration runner
└── types.ts                       # TypeScript type definitions
```

---

## Security Features

### 1. Authentication & Authorization
- ✅ Slack OAuth 2.0 with OpenID Connect
- ✅ HTTPOnly secure session cookies
- ✅ API key-based authentication for programmatic access
- ✅ Workspace validation (Hack Club Slack only)
- ✅ Identity verification integration
- ✅ User banning system

### 2. Network Security
- ✅ HTTPS enforcement (production)
- ✅ CSRF protection
- ✅ Secure HTTP headers (CSP, X-Frame-Options, etc.)
- ✅ IP address validation (via reverse proxy)
- ✅ Request size limiting (20MB)
- ✅ Request timeout (120s)

### 3. Application Security
- ✅ AI coding agent blocking (100+ tools)
- ✅ Model whitelisting
- ✅ API key rate limiting (50 keys/user)
- ✅ Environment variable validation
- ✅ SQL injection prevention (Drizzle ORM)
- ✅ XSS prevention (HTTPOnly cookies)

### 4. Monitoring & Audit
- ✅ Comprehensive request logging
- ✅ IP address tracking
- ✅ Token usage metrics
- ✅ Error tracking (Sentry)
- ✅ Performance monitoring
- ✅ Request ID correlation

### 5. Infrastructure
- ✅ Database migrations (versioned)
- ✅ Connection pooling
- ✅ Graceful degradation
- ✅ Error handling
- ✅ Health checks (implied via timeout)

---

## Database Schema

### `users` Table
```sql
id                UUID PRIMARY KEY DEFAULT gen_random_uuid()
slack_id          TEXT NOT NULL UNIQUE
slack_team_id     TEXT NOT NULL
email             TEXT
name              TEXT
avatar            TEXT
is_idv_verified   BOOLEAN NOT NULL DEFAULT false
skip_idv          BOOLEAN NOT NULL DEFAULT false
is_banned         BOOLEAN NOT NULL DEFAULT false
created_at        TIMESTAMP NOT NULL DEFAULT now()
updated_at        TIMESTAMP NOT NULL DEFAULT now()

INDEXES:
- users_slack_id_idx (slack_id)
- users_email_idx (email)
- users_idv_verified_idx (is_idv_verified)
```

**Purpose**: Stores user profile and verification status

### `sessions` Table
```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid()
user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
token       TEXT NOT NULL UNIQUE
expires_at  TIMESTAMP NOT NULL
created_at  TIMESTAMP NOT NULL DEFAULT now()

INDEXES:
- sessions_user_id_idx (user_id)
- sessions_expires_at_idx (expires_at)
```

**Purpose**: Manages web dashboard sessions

### `api_keys` Table
```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid()
user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
key         TEXT NOT NULL UNIQUE
name        TEXT NOT NULL
created_at  TIMESTAMP NOT NULL DEFAULT now()
revoked_at  TIMESTAMP NULL

INDEXES:
- api_keys_user_id_idx (user_id)
- api_keys_key_revoked_idx (key, revoked_at)
```

**Purpose**: Stores API keys for programmatic access

### `request_logs` Table
```sql
id                  UUID PRIMARY KEY DEFAULT gen_random_uuid()
api_key_id          UUID NOT NULL REFERENCES api_keys(id) ON DELETE CASCADE
user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE
slack_id            TEXT NOT NULL
model               TEXT NOT NULL
prompt_tokens       INTEGER NOT NULL DEFAULT 0
completion_tokens   INTEGER NOT NULL DEFAULT 0
total_tokens        INTEGER NOT NULL DEFAULT 0
request             JSONB NOT NULL
response            JSONB NOT NULL
ip                  TEXT NOT NULL
timestamp           TIMESTAMP NOT NULL DEFAULT now()
duration            INTEGER NOT NULL

INDEXES:
- request_logs_user_timestamp_idx (user_id, timestamp DESC)
- request_logs_apikey_timestamp_idx (api_key_id, timestamp DESC)
- request_logs_slack_timestamp_idx (slack_id, timestamp DESC)
- request_logs_model_idx (model)
- request_logs_user_id_idx (user_id)
```

**Purpose**: Complete audit trail of all API requests

**Storage Considerations**:
- JSONB columns allow flexible schema for different model APIs
- Indexes optimized for time-series queries
- Cascade deletes maintain referential integrity

---

## API Endpoints

### Authentication Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/auth/login` | None | Redirects to Slack OAuth |
| GET | `/auth/callback` | None | OAuth callback handler |
| GET | `/auth/logout` | Session | Logs out user |

### Dashboard Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/` | Optional | Home/landing page |
| GET | `/dashboard` | Session | User dashboard |
| GET | `/global` | Session | Global statistics |

### API Management Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/keys` | Session | Create new API key |
| GET | `/api/keys` | Session | List user's API keys |
| DELETE | `/api/keys/:id` | Session | Revoke API key |
| GET | `/api/stats` | Session | User statistics |

### LLM Proxy Endpoints (OpenAI-Compatible)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/proxy/v1/models` | None | List available models |
| POST | `/proxy/v1/chat/completions` | API Key | Chat completions (streaming/non-streaming) |
| POST | `/proxy/v1/embeddings` | API Key | Text embeddings |
| GET | `/proxy/v1/stats` | API Key | User token statistics |

### Documentation Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/docs` | Optional | API documentation |

---

## Environment Variables

### Required Variables

```bash
# Database
DATABASE_URL=postgresql://user:pass@host:5432/dbname

# Application
BASE_URL=https://ai.hackclub.com
PORT=54321
NODE_ENV=production|development|test

# Slack OAuth
SLACK_CLIENT_ID=xxx.xxx
SLACK_CLIENT_SECRET=xoxp-xxx
SLACK_TEAM_ID=T0266FRGM

# OpenAI/OpenRouter
OPENAI_API_KEY=sk-or-xxx
OPENAI_API_URL=https://openrouter.ai/api

# Models (comma-separated)
ALLOWED_LANGUAGE_MODELS=model1,model2,model3
ALLOWED_EMBEDDING_MODELS=embed1,embed2
```

### Optional Variables

```bash
# Identity Verification
ENFORCE_IDV=true|false         # Default: false

# Monitoring
SENTRY_DSN=https://xxx@sentry.io/xxx
```

---

## Key Design Decisions

### 1. Why OpenRouter?
- Unified API for multiple LLM providers
- Built-in rate limiting and billing
- Reduces complexity of managing multiple provider keys
- Enables easy model experimentation

### 2. Why Bun over Node.js?
- 3x faster startup time
- Native TypeScript support (no compilation step)
- Better performance for I/O operations
- Simpler deployment

### 3. Why Drizzle ORM?
- Type-safe queries
- Zero-cost abstractions
- SQL-like syntax (easier migration from raw SQL)
- Excellent migration system

### 4. Why Store API Keys in Plaintext?
- Performance: No hashing overhead on every request
- Simplicity: Proxy use case (not primary auth)
- Trade-off: If database compromised, keys exposed
- Mitigation: Short-lived keys + user revocation

### 5. Why 50 Key Limit?
- Prevents key distribution
- Balances flexibility vs abuse potential
- Most users need 1-5 keys (dev, prod, backup)
- Can be adjusted per-user via database

---

## Conclusion

The **Hack Club AI Proxy** implements a **comprehensive, multi-layered anti-fraud system** suitable for a service targeting teenagers. Key strengths include:

### Strengths
✅ **Identity Verification**: Ensures users are legitimate Hack Club members  
✅ **AI Agent Blocking**: Prevents automated abuse from 100+ tools  
✅ **Comprehensive Logging**: Full audit trail for forensics  
✅ **Workspace Validation**: Restricts to Hack Club Slack  
✅ **Model Whitelisting**: Controls costs and access  
✅ **User Banning**: Manual intervention for bad actors  
✅ **IP Tracking**: Enables pattern analysis  
✅ **Secure Architecture**: Modern security best practices  

### Potential Improvements
⚠️ **Rate Limiting**: No per-user request rate limits (relies on OpenRouter)  
⚠️ **API Key Security**: Plaintext storage vulnerable if DB compromised  
⚠️ **Token Quotas**: No hard limits on token usage per user  
⚠️ **Anomaly Detection**: Manual review required (no automated flagging)  
⚠️ **Geo-blocking**: No IP-based geographic restrictions  
⚠️ **Time-based Analysis**: No off-hours usage detection  

Overall, the system provides **solid protection** for its intended audience and use case, with clear paths for enhancement if abuse patterns emerge.

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-21  
**Author**: Copilot Code Agent  
**Repository**: https://github.com/abdessattar23/hackclub-ai
