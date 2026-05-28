# Social Media Management — Phased Development Plan

> Project: 123-social-media-management · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript 5.x (strict mode) | Full-stack type safety across API, frontend, and worker processes; largest ecosystem for social API client libraries (tweepy alternatives exist but TS is stronger for the full-stack dashboard); type sharing between API and frontend eliminates interface drift |
| Runtime | Node.js 22 LTS | Native ESM, stable LTS for production SaaS workloads; excellent async I/O for high-concurrency webhook and API polling patterns |
| API framework | Next.js 15 App Router + tRPC v11 | App Router provides SSR/RSC for analytics dashboards and calendar views; tRPC gives end-to-end type-safe RPC for the dashboard SPA; REST routes co-exist for the public API and MCP server |
| Database | PostgreSQL 16 | Hybrid relational + JSONB model (data-model-suggestion-3) handles platform-specific variance without schema explosion; RLS for tenant isolation; GIN indexes for JSONB filtering; TIMESTAMPTZ for cross-timezone scheduling; pg_cron for analytics snapshot collection |
| ORM | Drizzle ORM | Type-safe schema-as-code with native JSONB column support; migration generation from schema diffs; lighter than Prisma for JSONB-heavy models; zero-overhead SQL generation |
| Task queue | BullMQ 5.x on Redis 7 | Handles scheduled post publishing, analytics snapshot collection, social listening polling, sentiment analysis jobs, AI content generation, webhook delivery; repeatable jobs for sync scheduling; rate limiting per platform API quota |
| Frontend | React 19 + Next.js 15 App Router | Server Components reduce client JS bundle for analytics-heavy pages; Suspense boundaries for progressive loading of content calendar and inbox |
| UI components | shadcn/ui + Tailwind CSS 4 | Accessible, composable component library; drag-and-drop via @dnd-kit for content calendar; data tables via TanStack Table v8; rich text editing via Tiptap |
| Authentication | Auth.js v5 (NextAuth) | Built-in OAuth 2.0 providers (Google, GitHub) for operator login; JWT session strategy for API routes; extensible for enterprise SSO via OIDC |
| Social OAuth | Custom OAuth 2.0 + PKCE flows | Each social platform (Meta, X, LinkedIn, TikTok, YouTube, Pinterest) requires platform-specific OAuth; custom adapter layer per RFC 6749/7636; token refresh and rotation handled by a dedicated service |
| AI / LLM | Anthropic Claude SDK + Vercel AI SDK 4 | Claude for content generation, sentiment analysis, tone adjustment, content strategy inference; Vercel AI SDK for streaming responses and structured output; provider-swappable via adapter interface |
| Sentiment analysis | Hosted model via Claude + fallback to open-source (cardiffnlp/twitter-roberta-base-sentiment) | Claude for rich emotion/topic extraction (JSONB sentiment objects); open-source model as offline fallback; both fronted by a unified SentimentService interface |
| Media storage | S3-compatible (AWS S3, MinIO for self-hosted) | Presigned uploads for images/video; thumbnail generation via sharp; WebVTT caption storage alongside video assets |
| MCP server | @modelcontextprotocol/sdk | Official MCP TypeScript SDK; exposes scheduling, drafting, analytics, and inbox tools per MCP spec; Streamable HTTP transport for hosted, stdio for self-hosted |
| Testing | Vitest + Playwright | Vitest for unit/integration (fast, ESM-native); Playwright for E2E browser tests of calendar, inbox, and analytics dashboards |
| Containerisation | Docker + docker-compose | Single `docker compose up` for PostgreSQL, Redis, MinIO, and app; multi-stage Dockerfile for production image |
| Code quality | ESLint 9 (flat config) + Prettier | TypeScript strict mode enforced; no-explicit-any rule; consistent formatting |
| Package manager | pnpm 9 | Strict dependency resolution; workspace support for potential monorepo structure; faster installs than npm |
| API documentation | OpenAPI 3.1 (auto-generated from tRPC + Zod schemas) | Public REST API documented via OpenAPI; Scalar or Swagger UI for developer portal |

### Project Structure

```
social-media-management/
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json
├── drizzle.config.ts
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── src/
│   ├── app/                              # Next.js App Router
│   │   ├── layout.tsx
│   │   ├── page.tsx                      # Landing / dashboard redirect
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   └── callback/route.ts
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                # Sidebar + header shell
│   │   │   ├── calendar/page.tsx         # Content calendar
│   │   │   ├── posts/
│   │   │   │   ├── page.tsx              # Post list
│   │   │   │   ├── new/page.tsx          # Post composer
│   │   │   │   └── [id]/page.tsx         # Post detail / edit
│   │   │   ├── inbox/page.tsx            # Unified inbox
│   │   │   ├── listening/
│   │   │   │   ├── page.tsx              # Listening queries
│   │   │   │   └── [id]/page.tsx         # Query matches
│   │   │   ├── analytics/
│   │   │   │   ├── page.tsx              # Overview dashboard
│   │   │   │   ├── posts/page.tsx        # Post-level analytics
│   │   │   │   └── accounts/page.tsx     # Account-level analytics
│   │   │   ├── ai/
│   │   │   │   ├── generate/page.tsx     # AI content generation
│   │   │   │   └── strategy/page.tsx     # AI strategy assistant
│   │   │   ├── settings/
│   │   │   │   ├── accounts/page.tsx     # Connected social accounts
│   │   │   │   ├── team/page.tsx         # Team & roles
│   │   │   │   ├── workflows/page.tsx    # Approval workflows
│   │   │   │   └── general/page.tsx      # Tenant settings
│   │   │   └── alerts/page.tsx           # Trend alerts
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts      # tRPC handler
│   │       ├── webhooks/
│   │       │   ├── meta/route.ts         # Meta webhook receiver
│   │       │   └── x/route.ts            # X Account Activity webhook
│   │       ├── mcp/route.ts              # MCP server HTTP transport
│   │       └── v1/                       # Public REST API
│   │           ├── posts/route.ts
│   │           ├── accounts/route.ts
│   │           ├── analytics/route.ts
│   │           └── inbox/route.ts
│   ├── server/
│   │   ├── db/
│   │   │   ├── index.ts                  # Drizzle client + RLS setup
│   │   │   ├── schema/
│   │   │   │   ├── tenants.ts
│   │   │   │   ├── users.ts
│   │   │   │   ├── social-accounts.ts
│   │   │   │   ├── posts.ts
│   │   │   │   ├── post-results.ts
│   │   │   │   ├── media-assets.ts
│   │   │   │   ├── inbox-messages.ts
│   │   │   │   ├── listening-queries.ts
│   │   │   │   ├── listening-matches.ts
│   │   │   │   ├── analytics-snapshots.ts
│   │   │   │   ├── attribution-events.ts
│   │   │   │   ├── ai-interactions.ts
│   │   │   │   ├── trend-alerts.ts
│   │   │   │   └── audit-log.ts
│   │   │   └── migrations/
│   │   ├── trpc/
│   │   │   ├── router.ts                 # Root tRPC router
│   │   │   ├── context.ts
│   │   │   └── routers/
│   │   │       ├── posts.ts
│   │   │       ├── accounts.ts
│   │   │       ├── inbox.ts
│   │   │       ├── listening.ts
│   │   │       ├── analytics.ts
│   │   │       ├── ai.ts
│   │   │       ├── alerts.ts
│   │   │       └── settings.ts
│   │   ├── services/
│   │   │   ├── platforms/                # Platform adapter layer
│   │   │   │   ├── adapter.ts            # PlatformAdapter interface
│   │   │   │   ├── meta.ts               # Facebook + Instagram
│   │   │   │   ├── x.ts                  # X (Twitter)
│   │   │   │   ├── linkedin.ts
│   │   │   │   ├── tiktok.ts
│   │   │   │   ├── youtube.ts
│   │   │   │   ├── pinterest.ts
│   │   │   │   ├── mastodon.ts
│   │   │   │   ├── bluesky.ts
│   │   │   │   └── registry.ts           # Platform adapter registry
│   │   │   ├── publisher/
│   │   │   │   ├── publisher.ts          # Post publishing orchestrator
│   │   │   │   ├── scheduler.ts          # Scheduling engine
│   │   │   │   └── media-uploader.ts     # Platform media upload
│   │   │   ├── inbox/
│   │   │   │   ├── inbox-service.ts      # Unified inbox management
│   │   │   │   ├── routing-engine.ts     # Sentiment-based routing
│   │   │   │   └── sync.ts              # Inbox message polling
│   │   │   ├── listening/
│   │   │   │   ├── listener.ts           # Social listening engine
│   │   │   │   └── match-processor.ts    # Match analysis pipeline
│   │   │   ├── analytics/
│   │   │   │   ├── collector.ts          # Analytics snapshot collection
│   │   │   │   ├── aggregator.ts         # Cross-platform aggregation
│   │   │   │   └── attribution.ts        # Attribution tracking
│   │   │   ├── ai/
│   │   │   │   ├── content-generator.ts  # AI content generation
│   │   │   │   ├── sentiment.ts          # Sentiment analysis service
│   │   │   │   ├── strategy.ts           # Content strategy inference
│   │   │   │   ├── predictions.ts        # Performance predictions
│   │   │   │   └── prompts.ts            # Prompt templates
│   │   │   ├── auth/
│   │   │   │   ├── social-oauth.ts       # Social platform OAuth flows
│   │   │   │   └── token-manager.ts      # Token refresh/rotation
│   │   │   ├── media/
│   │   │   │   ├── storage.ts            # S3-compatible storage
│   │   │   │   └── processor.ts          # Thumbnails, EXIF stripping
│   │   │   ├── approval/
│   │   │   │   └── workflow-engine.ts    # Approval workflow engine
│   │   │   └── mcp/
│   │   │       ├── server.ts
│   │   │       ├── resources.ts
│   │   │       ├── tools.ts
│   │   │       └── prompts.ts
│   │   └── workers/
│   │       ├── index.ts                  # BullMQ worker bootstrap
│   │       ├── publish-worker.ts         # Scheduled post publisher
│   │       ├── analytics-worker.ts       # Analytics snapshot collector
│   │       ├── inbox-worker.ts           # Inbox sync poller
│   │       ├── listening-worker.ts       # Social listening poller
│   │       ├── sentiment-worker.ts       # Sentiment analysis pipeline
│   │       └── ai-worker.ts             # AI content generation queue
│   ├── lib/
│   │   ├── constants.ts
│   │   ├── errors.ts                     # Custom error types
│   │   ├── rate-limiter.ts               # Per-platform rate limiting
│   │   ├── crypto.ts                     # Token encryption/decryption
│   │   ├── validators.ts                 # Zod schemas for API validation
│   │   └── types.ts                      # Shared type definitions
│   └── components/
│       ├── ui/                           # shadcn/ui components
│       ├── calendar/
│       │   ├── content-calendar.tsx       # Drag-and-drop calendar
│       │   └── calendar-post-card.tsx
│       ├── composer/
│       │   ├── post-composer.tsx          # Post creation form
│       │   ├── platform-preview.tsx       # Per-platform preview
│       │   └── media-picker.tsx
│       ├── inbox/
│       │   ├── message-list.tsx
│       │   ├── message-thread.tsx
│       │   └── sentiment-badge.tsx
│       ├── analytics/
│       │   ├── metrics-card.tsx
│       │   ├── engagement-chart.tsx
│       │   └── platform-breakdown.tsx
│       └── shared/
│           ├── platform-icon.tsx
│           ├── account-selector.tsx
│           └── date-range-picker.tsx
├── tests/
│   ├── unit/
│   │   ├── services/
│   │   ├── lib/
│   │   └── db/
│   ├── integration/
│   │   ├── api/
│   │   ├── workers/
│   │   └── platforms/
│   ├── e2e/
│   │   ├── calendar.spec.ts
│   │   ├── post-creation.spec.ts
│   │   ├── inbox.spec.ts
│   │   └── analytics.spec.ts
│   └── fixtures/
│       ├── meta-api-responses/
│       ├── x-api-responses/
│       ├── linkedin-api-responses/
│       └── sample-media/
├── scripts/
│   ├── seed.ts                           # Development seed data
│   └── migrate.ts                        # Migration runner
└── docs/
    ├── api.md                            # Public API documentation
    └── self-hosting.md                   # Self-hosting guide
```

---

## Phase 1: Foundation & Project Scaffold

### Purpose

Establish the project skeleton, database schema, authentication system, and development tooling. After this phase, a developer can run `docker compose up`, log in to an empty dashboard, and see the shell of the application with tenant isolation working end-to-end.

### Tasks

#### 1.1 — Project Initialisation & Docker Setup

**What**: Create the Next.js 15 project with TypeScript strict mode, pnpm, ESLint, Prettier, and Docker Compose for PostgreSQL, Redis, and MinIO.

**Design**:

```typescript
// package.json (key dependencies)
{
  "dependencies": {
    "next": "^15.0.0",
    "@trpc/server": "^11.0.0",
    "@trpc/client": "^11.0.0",
    "@trpc/react-query": "^11.0.0",
    "drizzle-orm": "^0.38.0",
    "postgres": "^3.4.0",
    "bullmq": "^5.0.0",
    "ioredis": "^5.4.0",
    "@auth/core": "^0.36.0",
    "next-auth": "^5.0.0",
    "zod": "^3.24.0",
    "@aws-sdk/client-s3": "^3.700.0",
    "@aws-sdk/s3-request-presigner": "^3.700.0"
  }
}
```

```yaml
# docker-compose.yml
version: "3.9"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: smm
      POSTGRES_USER: smm
      POSTGRES_PASSWORD: smm_dev_password
    ports: ["5432:5432"]
    volumes: ["pg_data:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["minio_data:/data"]

  app:
    build: .
    depends_on: [postgres, redis, minio]
    ports: ["3000:3000"]
    env_file: .env

volumes:
  pg_data:
  minio_data:
```

```typescript
// .env.example
DATABASE_URL=postgresql://smm:smm_dev_password@localhost:5432/smm
REDIS_URL=redis://localhost:6379
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=smm-media
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=dev-secret-change-in-production
```

**Testing**:
- `Unit: .env.example contains all required environment variables`
- `Integration: docker compose up → all services healthy within 30 seconds`
- `Integration: Next.js dev server starts without errors on port 3000`
- `Integration: PostgreSQL connection via Drizzle client succeeds`
- `Integration: Redis connection via ioredis succeeds`
- `Integration: MinIO bucket creation succeeds via AWS SDK`

---

#### 1.2 — Database Schema (Core Tables)

**What**: Implement the hybrid relational + JSONB schema from data-model-suggestion-3, covering tenants, users, social_accounts, posts, post_results, media_assets, and audit_log tables.

**Design**:

```typescript
// src/server/db/schema/tenants.ts
import { pgTable, uuid, varchar, jsonb, timestamp, uniqueIndex } from "drizzle-orm/pg-core";

export const tenants = pgTable("tenants", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 100 }).notNull().unique(),
  planTier: varchar("plan_tier", { length: 50 }).notNull().default("free"),
  settings: jsonb("settings").notNull().default({}),
  // settings: { default_timezone, ai_provider, branding: {primary_color, logo_url}, features_enabled[] }
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// src/server/db/schema/users.ts
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id, { onDelete: "cascade" }),
  email: varchar("email", { length: 320 }).notNull(),
  displayName: varchar("display_name", { length: 255 }).notNull(),
  avatarUrl: varchar("avatar_url", { length: 2048 }),
  role: varchar("role", { length: 50 }).notNull().default("editor"),
  // role: admin | editor | approver | viewer
  permissions: jsonb("permissions").notNull().default([]),
  // permissions: [{resource: string, actions: string[]}]
  preferences: jsonb("preferences").notNull().default({}),
  authProvider: varchar("auth_provider", { length: 50 }).notNull().default("email"),
  authSubject: varchar("auth_subject", { length: 512 }),
  isActive: boolean("is_active").notNull().default(true),
  lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueTenantEmail: uniqueIndex("uq_users_tenant_email").on(table.tenantId, table.email),
}));

// src/server/db/schema/posts.ts
export const posts = pgTable("posts", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id, { onDelete: "cascade" }),
  status: varchar("status", { length: 30 }).notNull().default("draft"),
  // status: draft | pending_approval | approved | scheduled | publishing | published | failed | archived
  contentType: varchar("content_type", { length: 50 }).notNull(),
  // contentType: text_post | image_post | video_post | carousel | story | reel | poll | thread
  bodyText: text("body_text"),
  scheduledAt: timestamp("scheduled_at", { withTimezone: true }),
  publishedAt: timestamp("published_at", { withTimezone: true }),
  timezone: varchar("timezone", { length: 50 }).notNull().default("UTC"),
  createdBy: uuid("created_by").notNull().references(() => users.id),
  platformConfigs: jsonb("platform_configs").notNull().default({}),
  // platformConfigs: { [platform]: { social_account_id, ...platform-specific fields } }
  approvalStatus: varchar("approval_status", { length: 30 }),
  approvedBy: uuid("approved_by").references(() => users.id),
  approvedAt: timestamp("approved_at", { withTimezone: true }),
  abTestGroup: varchar("ab_test_group", { length: 10 }),
  abParentId: uuid("ab_parent_id").references(() => posts.id),
  labels: text("labels").array().default([]),
  metadata: jsonb("metadata").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

Row-Level Security policy applied via migration:

```sql
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_users ON users
  FOR ALL USING (tenant_id = current_setting('app.current_tenant')::UUID);

CREATE POLICY tenant_isolation_posts ON posts
  FOR ALL USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

**Testing**:
- `Unit: Drizzle schema compiles with zero type errors`
- `Integration: migration runs successfully against clean PostgreSQL database`
- `Integration: insert tenant → insert user with tenant_id → user row visible`
- `Integration: RLS policy → query with wrong tenant_id returns zero rows`
- `Integration: insert post with JSONB platform_configs → round-trip read matches`
- `Unit: post status values match PostStatus enum`
- `Integration: cascade delete tenant → all users and posts for tenant deleted`

---

#### 1.3 — Authentication & Tenant Context

**What**: Implement Auth.js v5 for operator login (email/password + Google OAuth), tenant resolution from session, and middleware that injects `app.current_tenant` for RLS.

**Design**:

```typescript
// src/server/auth.ts
import NextAuth from "next-auth";
import Google from "next-auth/providers/google";
import Credentials from "next-auth/providers/credentials";

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    Credentials({
      credentials: { email: {}, password: {} },
      authorize: async (credentials) => {
        // Validate against users table with bcrypt
      },
    }),
  ],
  callbacks: {
    session({ session, token }) {
      session.user.tenantId = token.tenantId as string;
      session.user.role = token.role as string;
      return session;
    },
    jwt({ token, user }) {
      if (user) {
        token.tenantId = user.tenantId;
        token.role = user.role;
      }
      return token;
    },
  },
});

// src/server/db/index.ts — RLS-aware query helper
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";

const client = postgres(process.env.DATABASE_URL!);
const db = drizzle(client);

export async function withTenant<T>(
  tenantId: string,
  fn: (tx: typeof db) => Promise<T>
): Promise<T> {
  return db.transaction(async (tx) => {
    await tx.execute(sql`SET LOCAL app.current_tenant = ${tenantId}`);
    return fn(tx);
  });
}

// src/server/trpc/context.ts
export async function createContext(opts: { req: NextRequest }) {
  const session = await auth();
  return {
    session,
    tenantId: session?.user?.tenantId ?? null,
    userId: session?.user?.id ?? null,
    db,
  };
}
```

**Testing**:
- `Unit: JWT callback injects tenantId and role into token`
- `Unit: session callback exposes tenantId and role on session.user`
- `Integration: login with valid credentials → session created with correct tenantId`
- `Integration: login with invalid credentials → 401 returned`
- `Integration: Google OAuth flow → user created if first login, session established`
- `Integration: withTenant() sets app.current_tenant → RLS filters correctly`
- `Integration: unauthenticated request to tRPC route → 401`
- `Integration: request with tenant A credentials cannot read tenant B data`

---

#### 1.4 — Dashboard Shell & Navigation

**What**: Build the authenticated dashboard layout with sidebar navigation, header with user menu, and placeholder pages for all major sections.

**Design**:

```typescript
// src/app/(dashboard)/layout.tsx
interface DashboardLayoutProps {
  children: React.ReactNode;
}

// Sidebar navigation items
const navItems = [
  { label: "Calendar", href: "/calendar", icon: CalendarIcon },
  { label: "Posts", href: "/posts", icon: PenSquareIcon },
  { label: "Inbox", href: "/inbox", icon: InboxIcon },
  { label: "Listening", href: "/listening", icon: EarIcon },
  { label: "Analytics", href: "/analytics", icon: BarChartIcon },
  { label: "AI Studio", href: "/ai/generate", icon: SparklesIcon },
  { label: "Alerts", href: "/alerts", icon: BellIcon },
  { label: "Settings", href: "/settings/accounts", icon: SettingsIcon },
] as const;

// src/components/shared/account-selector.tsx
interface AccountSelectorProps {
  selectedAccountIds: string[];
  onChange: (ids: string[]) => void;
  platformFilter?: string;
}
// Multi-select dropdown showing connected social accounts with platform icons
```

**Testing**:
- `E2E: authenticated user sees dashboard layout with sidebar navigation`
- `E2E: sidebar links navigate to correct pages`
- `E2E: unauthenticated user redirected to /login`
- `E2E: user menu shows display name and logout option`
- `Unit: navItems array contains all required routes`
- `E2E: responsive layout collapses sidebar on mobile viewport`

---

## Phase 2: Social Account Connection & Platform Adapter Layer

### Purpose

Build the platform adapter abstraction and implement OAuth connection flows for the initial set of social platforms (Meta, X, LinkedIn). After this phase, users can connect their social accounts and see them listed in settings. The adapter layer provides the foundation for all subsequent publishing, inbox, and analytics features.

### Tasks

#### 2.1 — Platform Adapter Interface

**What**: Define the `PlatformAdapter` interface and platform registry that all social platform integrations implement.

**Design**:

```typescript
// src/server/services/platforms/adapter.ts
import { z } from "zod";

export const PlatformCapability = z.enum([
  "publish", "schedule", "analytics", "inbox",
  "listening", "stories", "reels", "polls", "threads",
]);
export type PlatformCapability = z.infer<typeof PlatformCapability>;

export const PlatformCode = z.enum([
  "facebook", "instagram", "x", "linkedin", "tiktok",
  "youtube", "pinterest", "mastodon", "bluesky", "threads",
]);
export type PlatformCode = z.infer<typeof PlatformCode>;

export interface OAuthConfig {
  authorizationUrl: string;
  tokenUrl: string;
  scopes: string[];
  usePKCE: boolean;
}

export interface PublishRequest {
  bodyText: string;
  mediaAssetIds?: string[];
  platformConfig: Record<string, unknown>;  // platform-specific options
  scheduledAt?: Date;
}

export interface PublishResult {
  platformPostId: string;
  platformUrl: string;
  publishedAt: Date;
}

export interface AnalyticsSnapshot {
  impressions: number;
  reach: number;
  engagements: number;
  likes: number;
  comments: number;
  shares: number;
  clicks: number;
  videoViews?: number;
  videoWatchTimeMs?: number;
  platformSpecific?: Record<string, unknown>;
}

export interface InboxMessage {
  platformMessageId: string;
  messageType: "comment" | "direct_message" | "mention" | "reply" | "review";
  direction: "inbound" | "outbound";
  authorName: string;
  authorHandle: string;
  authorAvatarUrl?: string;
  authorPlatformUid: string;
  bodyText: string;
  parentMessageId?: string;
  receivedAt: Date;
}

export interface PlatformAdapter {
  readonly code: PlatformCode;
  readonly displayName: string;
  readonly capabilities: PlatformCapability[];

  // OAuth
  getOAuthConfig(): OAuthConfig;
  exchangeCodeForTokens(code: string, redirectUri: string, codeVerifier?: string): Promise<{
    accessToken: string;
    refreshToken?: string;
    expiresAt?: Date;
    scopes: string[];
    accountInfo: { platformUid: string; accountName: string; accountHandle?: string; accountType: string; avatarUrl?: string };
  }>;
  refreshAccessToken(refreshToken: string): Promise<{
    accessToken: string;
    refreshToken?: string;
    expiresAt?: Date;
  }>;

  // Publishing
  publish(accessToken: string, request: PublishRequest): Promise<PublishResult>;
  deletePost(accessToken: string, platformPostId: string): Promise<void>;

  // Media
  uploadMedia(accessToken: string, file: Buffer, mimeType: string, altText?: string): Promise<string>;

  // Analytics
  getPostAnalytics(accessToken: string, platformPostId: string): Promise<AnalyticsSnapshot>;
  getAccountAnalytics(accessToken: string, platformUid: string, since: Date, until: Date): Promise<AnalyticsSnapshot>;

  // Inbox
  getInboxMessages(accessToken: string, since?: Date, cursor?: string): Promise<{
    messages: InboxMessage[];
    nextCursor?: string;
  }>;
  replyToMessage(accessToken: string, platformMessageId: string, text: string): Promise<void>;
}

// src/server/services/platforms/registry.ts
export class PlatformRegistry {
  private adapters = new Map<PlatformCode, PlatformAdapter>();

  register(adapter: PlatformAdapter): void {
    this.adapters.set(adapter.code, adapter);
  }

  get(code: PlatformCode): PlatformAdapter {
    const adapter = this.adapters.get(code);
    if (!adapter) throw new PlatformNotSupportedError(code);
    return adapter;
  }

  getAll(): PlatformAdapter[] {
    return Array.from(this.adapters.values());
  }

  getWithCapability(capability: PlatformCapability): PlatformAdapter[] {
    return this.getAll().filter(a => a.capabilities.includes(capability));
  }
}
```

**Testing**:
- `Unit: PlatformRegistry.register() adds adapter, get() retrieves it`
- `Unit: PlatformRegistry.get() with unknown code throws PlatformNotSupportedError`
- `Unit: PlatformRegistry.getWithCapability("publish") returns only adapters with publish capability`
- `Unit: PlatformCode enum contains all 10 supported platforms`
- `Unit: PlatformCapability enum contains all capability types`

---

#### 2.2 — Meta Platform Adapter (Facebook + Instagram)

**What**: Implement the PlatformAdapter for Facebook Pages and Instagram Business accounts using the Meta Graph API.

**Design**:

```typescript
// src/server/services/platforms/meta.ts
export class MetaAdapter implements PlatformAdapter {
  readonly code = "facebook" as const;  // Separate instance for Instagram
  readonly displayName = "Facebook";
  readonly capabilities: PlatformCapability[] = [
    "publish", "schedule", "analytics", "inbox", "stories",
  ];

  private readonly graphApiBase = "https://graph.facebook.com/v22.0";

  getOAuthConfig(): OAuthConfig {
    return {
      authorizationUrl: "https://www.facebook.com/v22.0/dialog/oauth",
      tokenUrl: `${this.graphApiBase}/oauth/access_token`,
      scopes: [
        "pages_manage_posts", "pages_read_engagement",
        "pages_messaging", "pages_read_user_content",
        "instagram_basic", "instagram_content_publish",
        "instagram_manage_comments", "instagram_manage_messages",
      ],
      usePKCE: false,
    };
  }

  async publish(accessToken: string, request: PublishRequest): Promise<PublishResult> {
    // POST /{page-id}/feed for Facebook
    // POST /{ig-user-id}/media + POST /{ig-user-id}/media_publish for Instagram
    // Handle carousel via /{ig-user-id}/media (children) + /{ig-user-id}/media_publish
  }

  async getPostAnalytics(accessToken: string, platformPostId: string): Promise<AnalyticsSnapshot> {
    // GET /{post-id}/insights?metric=post_impressions,post_engaged_users,...
  }

  async getInboxMessages(accessToken: string, since?: Date, cursor?: string) {
    // GET /{page-id}/conversations?fields=messages{message,from,created_time}
  }
}

// Instagram variant
export class InstagramAdapter extends MetaAdapter {
  readonly code = "instagram" as const;
  readonly displayName = "Instagram";
  readonly capabilities: PlatformCapability[] = [
    "publish", "schedule", "analytics", "inbox", "stories", "reels", "threads",
  ];
}
```

**Testing**:
- `Unit: getOAuthConfig() returns correct Meta authorization URL and scopes`
- `Integration (mocked): exchangeCodeForTokens() with valid code → returns access token and page info`
- `Integration (mocked): exchangeCodeForTokens() with invalid code → throws OAuthError`
- `Integration (mocked): publish() text post → POST to /{page-id}/feed, returns platformPostId`
- `Integration (mocked): publish() with media → uploads media first, then creates post with media_id`
- `Integration (mocked): getPostAnalytics() → returns parsed AnalyticsSnapshot from /insights`
- `Integration (mocked): getInboxMessages() → returns parsed InboxMessage array from /conversations`
- `Integration (mocked): refreshAccessToken() → returns new token from /oauth/access_token`
- `Integration (mocked): API returns 429 rate limit → adapter throws RateLimitError with retryAfter`
- `Fixture: meta-api-responses/publish-success.json, publish-rate-limited.json, insights-response.json`

---

#### 2.3 — X (Twitter) Platform Adapter

**What**: Implement the PlatformAdapter for X using API v2 with OAuth 2.0 + PKCE (RFC 7636).

**Design**:

```typescript
// src/server/services/platforms/x.ts
export class XAdapter implements PlatformAdapter {
  readonly code = "x" as const;
  readonly displayName = "X (Twitter)";
  readonly capabilities: PlatformCapability[] = [
    "publish", "analytics", "inbox", "listening", "polls", "threads",
  ];

  private readonly apiBase = "https://api.x.com/2";

  getOAuthConfig(): OAuthConfig {
    return {
      authorizationUrl: "https://x.com/i/oauth2/authorize",
      tokenUrl: "https://api.x.com/2/oauth2/token",
      scopes: [
        "tweet.read", "tweet.write", "users.read",
        "dm.read", "dm.write", "offline.access",
      ],
      usePKCE: true,  // RFC 7636 required by X API v2
    };
  }

  async publish(accessToken: string, request: PublishRequest): Promise<PublishResult> {
    // POST /2/tweets with { text, media: { media_ids }, poll: { options, duration_minutes } }
    // Thread support: chain tweets via reply_settings and in_reply_to_tweet_id
  }

  async getPostAnalytics(accessToken: string, platformPostId: string): Promise<AnalyticsSnapshot> {
    // GET /2/tweets/{id}?tweet.fields=public_metrics,non_public_metrics,organic_metrics
  }
}
```

**Testing**:
- `Unit: getOAuthConfig() returns X authorization URL with PKCE enabled`
- `Integration (mocked): publish() single tweet → POST to /2/tweets, returns tweet ID`
- `Integration (mocked): publish() thread → chains tweets via in_reply_to_tweet_id`
- `Integration (mocked): publish() with poll → includes poll options in request body`
- `Integration (mocked): API returns 403 forbidden → adapter throws PlatformPermissionError`
- `Integration (mocked): PKCE flow → code_verifier and code_challenge sent correctly`
- `Fixture: x-api-responses/tweet-created.json, tweet-metrics.json, rate-limited.json`

---

#### 2.4 — LinkedIn Platform Adapter

**What**: Implement the PlatformAdapter for LinkedIn Pages using the Marketing API.

**Design**:

```typescript
// src/server/services/platforms/linkedin.ts
export class LinkedInAdapter implements PlatformAdapter {
  readonly code = "linkedin" as const;
  readonly displayName = "LinkedIn";
  readonly capabilities: PlatformCapability[] = [
    "publish", "analytics", "inbox",
  ];

  private readonly apiBase = "https://api.linkedin.com/v2";

  getOAuthConfig(): OAuthConfig {
    return {
      authorizationUrl: "https://www.linkedin.com/oauth/v2/authorization",
      tokenUrl: "https://www.linkedin.com/oauth/v2/accessToken",
      scopes: [
        "w_member_social", "r_organization_social",
        "w_organization_social", "r_organization_admin",
        "rw_organization_admin",
      ],
      usePKCE: false,
    };
  }

  async publish(accessToken: string, request: PublishRequest): Promise<PublishResult> {
    // POST /ugcPosts with { author, lifecycleState: "PUBLISHED", specificContent: { shareContent: { shareCommentary, shareMediaCategory, media } } }
    // Rest.li protocol 2.0.0 headers required
  }
}
```

**Testing**:
- `Unit: getOAuthConfig() returns LinkedIn authorization URL and scopes`
- `Integration (mocked): publish() text post → POST to /ugcPosts with correct Rest.li headers`
- `Integration (mocked): publish() with image → registers upload, uploads binary, creates post`
- `Integration (mocked): getPostAnalytics() → parses LinkedIn insights format into AnalyticsSnapshot`
- `Fixture: linkedin-api-responses/ugc-post-created.json, organization-insights.json`

---

#### 2.5 — Social Account Connection Flow

**What**: Build the UI and API for connecting social accounts via OAuth, storing encrypted tokens, and displaying connected accounts in settings.

**Design**:

```typescript
// src/server/services/auth/social-oauth.ts
export class SocialOAuthService {
  constructor(
    private registry: PlatformRegistry,
    private tokenManager: TokenManager,
    private db: DrizzleClient,
  ) {}

  generateAuthUrl(platform: PlatformCode, tenantId: string, redirectUri: string): {
    url: string;
    state: string;
    codeVerifier?: string;
  } {
    const adapter = this.registry.get(platform);
    const config = adapter.getOAuthConfig();
    const state = crypto.randomUUID();
    // Store state + codeVerifier in Redis with 10-minute TTL
    // Build authorization URL with redirect_uri, scope, state, code_challenge
  }

  async handleCallback(
    platform: PlatformCode,
    code: string,
    state: string,
    tenantId: string,
    userId: string,
  ): Promise<SocialAccount> {
    // Verify state from Redis
    // Exchange code for tokens via adapter
    // Encrypt tokens via TokenManager
    // Upsert social_accounts row
    // Write audit_log entry
  }
}

// src/server/services/auth/token-manager.ts
export class TokenManager {
  private readonly algorithm = "aes-256-gcm";

  encrypt(plaintext: string): string {
    // AES-256-GCM encryption with random IV
    // Returns base64 encoded: iv:authTag:ciphertext
  }

  decrypt(ciphertext: string): string {
    // Parse iv:authTag:ciphertext and decrypt
  }

  async refreshIfNeeded(accountId: string): Promise<string> {
    // Check token_expires_at; if within 5 minutes, refresh via adapter
    // Update social_accounts row with new encrypted tokens
    // Return valid access token
  }
}

// src/server/trpc/routers/accounts.ts
export const accountsRouter = router({
  list: protectedProcedure
    .query(async ({ ctx }) => {
      // Return all social_accounts for ctx.tenantId
    }),

  getAuthUrl: protectedProcedure
    .input(z.object({ platform: PlatformCode }))
    .mutation(async ({ ctx, input }) => {
      // Generate OAuth URL via SocialOAuthService
    }),

  disconnect: protectedProcedure
    .input(z.object({ accountId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => {
      // Soft-delete: set is_active = false, clear tokens
    }),
});
```

**Testing**:
- `Unit: SocialOAuthService.generateAuthUrl() builds correct URL with scope and state`
- `Unit: SocialOAuthService.generateAuthUrl() includes code_challenge for PKCE platforms`
- `Unit: TokenManager.encrypt() → decrypt() round-trip returns original value`
- `Unit: TokenManager.encrypt() produces different ciphertext for same plaintext (random IV)`
- `Integration (mocked): handleCallback() with valid code → social_account created with encrypted tokens`
- `Integration (mocked): handleCallback() with invalid state → throws InvalidStateError`
- `Integration: refreshIfNeeded() with expired token → calls adapter.refreshAccessToken(), updates row`
- `Integration: refreshIfNeeded() with valid token → returns existing token without API call`
- `E2E: click "Connect Facebook" → OAuth redirect → callback → account appears in list`
- `E2E: click "Disconnect" on connected account → account removed from list`
- `Integration: audit_log entry created for account connection and disconnection`

---

## Phase 3: Post Composition & Scheduling Engine

### Purpose

Build the core content creation and scheduling pipeline. After this phase, users can compose posts with text and media, preview them for each target platform, schedule them for future publication, and have them automatically published at the scheduled time by the background worker.

### Tasks

#### 3.1 — Post Composer UI

**What**: Build the post creation form with multi-platform targeting, platform-specific previews, media attachment, and character count enforcement.

**Design**:

```typescript
// src/components/composer/post-composer.tsx
interface PostComposerProps {
  initialPost?: PostDraft;
  onSave: (draft: PostDraft) => void;
  onSchedule: (draft: PostDraft, scheduledAt: Date) => void;
  onPublishNow: (draft: PostDraft) => void;
}

interface PostDraft {
  bodyText: string;
  contentType: ContentType;
  targetAccountIds: string[];
  mediaAssetIds: string[];
  platformConfigs: Record<PlatformCode, PlatformSpecificConfig>;
  labels: string[];
  scheduledAt?: Date;
  timezone: string;
}

// Platform-specific character limits and constraints
const PLATFORM_CONSTRAINTS: Record<PlatformCode, PlatformConstraints> = {
  x: { maxChars: 280, maxMedia: 4, maxVideoLengthMs: 140_000, supportsPolls: true },
  facebook: { maxChars: 63_206, maxMedia: 10, supportsPolls: false },
  instagram: { maxChars: 2_200, maxMedia: 10, requiresMedia: true, supportsCarousel: true },
  linkedin: { maxChars: 3_000, maxMedia: 9, supportsArticles: true },
  tiktok: { maxChars: 2_200, maxMedia: 1, videoOnly: true },
  youtube: { maxChars: 5_000, maxMedia: 1, videoOnly: true },
  mastodon: { maxChars: 500, maxMedia: 4, supportsContentWarning: true },
  bluesky: { maxChars: 300, maxMedia: 4 },
  pinterest: { maxChars: 500, maxMedia: 1, requiresMedia: true },
  threads: { maxChars: 500, maxMedia: 10 },
};

// src/components/composer/platform-preview.tsx
interface PlatformPreviewProps {
  platform: PlatformCode;
  bodyText: string;
  mediaAssets: MediaAsset[];
  accountHandle: string;
  accountAvatarUrl: string;
}
// Renders a visual approximation of how the post will appear on each platform
```

**Testing**:
- `E2E: type post text → character count updates in real-time for each targeted platform`
- `E2E: exceed X character limit → warning shown, platform preview shows truncation`
- `E2E: select Instagram target → media attachment becomes required`
- `E2E: attach image → thumbnail preview shown, alt text input appears (WCAG 2.2)`
- `E2E: select multiple target platforms → previews shown for each`
- `E2E: save as draft → post appears in posts list with "draft" status`
- `Unit: PLATFORM_CONSTRAINTS contains entries for all PlatformCode values`
- `Unit: character count calculation handles Unicode emoji correctly (grapheme clusters)`
- `E2E: platform-specific config fields appear when platform selected (e.g., content warning for Mastodon)`

---

#### 3.2 — Media Upload & Processing

**What**: Implement presigned S3 uploads, server-side image/video processing (thumbnail generation, EXIF stripping), and media asset management.

**Design**:

```typescript
// src/server/services/media/storage.ts
export class MediaStorageService {
  constructor(private s3Client: S3Client, private bucket: string) {}

  async generatePresignedUpload(
    tenantId: string,
    fileName: string,
    mimeType: string,
    maxSizeBytes: number,
  ): Promise<{ uploadUrl: string; assetId: string; key: string }> {
    const assetId = crypto.randomUUID();
    const key = `${tenantId}/${assetId}/${fileName}`;
    // PutObject presigned URL with 15-minute expiry
  }

  async confirmUpload(assetId: string): Promise<MediaAsset> {
    // Verify file exists in S3
    // Run MediaProcessor for thumbnail + EXIF stripping
    // Insert media_assets row
  }
}

// src/server/services/media/processor.ts
import sharp from "sharp";

export class MediaProcessor {
  async processImage(buffer: Buffer): Promise<ProcessedMedia> {
    const metadata = await sharp(buffer).metadata();
    const thumbnail = await sharp(buffer).resize(400, 400, { fit: "cover" }).toBuffer();
    const stripped = await sharp(buffer).rotate().withMetadata({ exif: {} }).toBuffer();
    // Remove EXIF geolocation per IPTC/EXIF privacy guidelines
    return { width: metadata.width, height: metadata.height, thumbnail, processed: stripped };
  }

  async processVideo(buffer: Buffer): Promise<ProcessedMedia> {
    // Extract thumbnail from first frame via ffmpeg
    // Strip geolocation metadata
    // Return dimensions and duration
  }
}

// src/server/db/schema/media-assets.ts
export const mediaAssets = pgTable("media_assets", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id, { onDelete: "cascade" }),
  fileName: varchar("file_name", { length: 500 }).notNull(),
  fileType: varchar("file_type", { length: 100 }).notNull(),
  fileSizeBytes: bigint("file_size_bytes", { mode: "number" }).notNull(),
  storageUrl: text("storage_url").notNull(),
  thumbnailUrl: text("thumbnail_url"),
  dimensions: jsonb("dimensions"),  // { width, height, duration_ms, aspect_ratio }
  altText: text("alt_text"),
  captionVtt: text("caption_vtt"),   // WebVTT for video captions
  iptcMetadata: jsonb("iptc_metadata").default({}),
  uploadedBy: uuid("uploaded_by").notNull().references(() => users.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- `Unit: generatePresignedUpload() returns URL with correct expiry and content-type constraint`
- `Unit: processImage() strips EXIF geolocation data`
- `Unit: processImage() generates 400x400 thumbnail from various aspect ratios`
- `Unit: processImage() preserves orientation via auto-rotate`
- `Integration: full upload flow → presigned URL → S3 upload → confirmUpload() → media_assets row created`
- `Integration: upload non-image file type → file_type correctly detected`
- `Integration: upload exceeding max size → rejected before processing`
- `Unit: video processing extracts duration_ms and first-frame thumbnail`
- `Fixture: sample-media/landscape.jpg, portrait.jpg, with-exif-gps.jpg, short-video.mp4`

---

#### 3.3 — Scheduling Engine & Publish Worker

**What**: Build the scheduling engine that enqueues posts at their scheduled time and the BullMQ worker that publishes them to target platforms via the adapter layer.

**Design**:

```typescript
// src/server/services/publisher/scheduler.ts
export class SchedulingEngine {
  constructor(
    private publishQueue: Queue,
    private db: DrizzleClient,
  ) {}

  async schedulePost(postId: string, scheduledAt: Date): Promise<void> {
    const delay = scheduledAt.getTime() - Date.now();
    if (delay < 0) throw new ScheduleInPastError(scheduledAt);

    await this.db.update(posts).set({
      status: "scheduled",
      scheduledAt,
      updatedAt: new Date(),
    }).where(eq(posts.id, postId));

    await this.publishQueue.add(
      "publish-post",
      { postId },
      {
        delay,
        jobId: `publish-${postId}`,  // prevents duplicate scheduling
        removeOnComplete: 100,
        removeOnFail: 200,
        attempts: 3,
        backoff: { type: "exponential", delay: 30_000 },
      },
    );
  }

  async cancelSchedule(postId: string): Promise<void> {
    const job = await this.publishQueue.getJob(`publish-${postId}`);
    if (job) await job.remove();
    await this.db.update(posts).set({
      status: "draft",
      scheduledAt: null,
      updatedAt: new Date(),
    }).where(eq(posts.id, postId));
  }
}

// src/server/workers/publish-worker.ts
export class PublishWorker {
  constructor(
    private registry: PlatformRegistry,
    private tokenManager: TokenManager,
    private db: DrizzleClient,
  ) {}

  async processJob(job: Job<{ postId: string }>): Promise<void> {
    const post = await this.db.select().from(posts).where(eq(posts.id, job.data.postId)).get();
    if (!post || post.status !== "scheduled") return;

    await this.db.update(posts).set({ status: "publishing" }).where(eq(posts.id, post.id));

    const platformConfigs = post.platformConfigs as Record<string, any>;
    const results: PostTargetResult[] = [];

    for (const [platform, config] of Object.entries(platformConfigs)) {
      try {
        const adapter = this.registry.get(platform as PlatformCode);
        const accessToken = await this.tokenManager.refreshIfNeeded(config.social_account_id);
        const result = await adapter.publish(accessToken, {
          bodyText: post.bodyText ?? "",
          mediaAssetIds: config.media_ids ?? [],
          platformConfig: config,
        });
        results.push({ platform, status: "published", ...result });
      } catch (error) {
        results.push({ platform, status: "failed", error: error.message });
      }
    }

    // Insert post_results rows for each platform
    // Update post status: "published" if any succeeded, "failed" if all failed
  }
}

// Post lifecycle states
type PostStatus =
  | "draft"             // Initial creation
  | "pending_approval"  // Submitted for review
  | "approved"          // Approved, ready to schedule
  | "scheduled"         // Scheduled for future publish
  | "publishing"        // Currently being published
  | "published"         // Successfully published to at least one platform
  | "failed"            // Failed on all platforms
  | "archived";         // Manually archived
```

**Testing**:
- `Unit: schedulePost() with future date → job added to BullMQ with correct delay`
- `Unit: schedulePost() with past date → throws ScheduleInPastError`
- `Unit: schedulePost() called twice for same post → second call replaces job (no duplicates)`
- `Unit: cancelSchedule() removes BullMQ job and resets post to draft`
- `Integration (mocked): PublishWorker processes job → calls adapter.publish() for each platform in platformConfigs`
- `Integration (mocked): publish succeeds on Facebook, fails on X → post status "published", post_results show mixed results`
- `Integration (mocked): publish fails on all platforms → post status "failed"`
- `Integration (mocked): token expired → tokenManager.refreshIfNeeded() refreshes before publish`
- `Integration (mocked): adapter throws RateLimitError → job retried with exponential backoff`
- `Integration: publish job with non-existent postId → job completes without error (idempotent)`
- `E2E: schedule post for 1 minute in future → post appears as "scheduled" → status changes to "published" after delay`

---

#### 3.4 — Content Calendar View

**What**: Build the visual content calendar with drag-and-drop rescheduling, week/month views, and post status colouring.

**Design**:

```typescript
// src/components/calendar/content-calendar.tsx
interface ContentCalendarProps {
  view: "week" | "month";
  currentDate: Date;
  timezone: string;
}

// Calendar displays posts grouped by scheduledAt date
// Each post card shows: status colour, body preview (truncated), target platform icons, author avatar
// Drag-and-drop a card to a different time slot → calls scheduler.reschedule(postId, newDate)
// Click a card → navigates to post detail/edit page

// Post status colours
const STATUS_COLORS: Record<PostStatus, string> = {
  draft: "gray",
  pending_approval: "yellow",
  approved: "blue",
  scheduled: "purple",
  publishing: "indigo",
  published: "green",
  failed: "red",
  archived: "slate",
};

// src/server/trpc/routers/posts.ts
export const postsRouter = router({
  calendarRange: protectedProcedure
    .input(z.object({
      startDate: z.date(),
      endDate: z.date(),
      accountIds: z.array(z.string().uuid()).optional(),
      statuses: z.array(PostStatusSchema).optional(),
    }))
    .query(async ({ ctx, input }) => {
      // SELECT posts WHERE scheduled_at BETWEEN startDate AND endDate
      // JOIN post_results for status per platform
      // Filter by accountIds and statuses if provided
    }),

  reschedule: protectedProcedure
    .input(z.object({
      postId: z.string().uuid(),
      newScheduledAt: z.date(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Cancel existing BullMQ job
      // Update scheduledAt
      // Create new BullMQ job with new delay
    }),
});
```

**Testing**:
- `E2E: calendar displays scheduled posts on correct dates`
- `E2E: switch between week and month views → layout changes correctly`
- `E2E: drag post from Monday to Wednesday → scheduledAt updated, confirmation toast shown`
- `E2E: drag published post → drag rejected (cannot reschedule published posts)`
- `E2E: posts coloured by status (green for published, purple for scheduled, red for failed)`
- `E2E: click post card → navigates to post detail page`
- `Unit: calendarRange query filters by date range correctly`
- `Integration: reschedule mutation cancels old BullMQ job and creates new one`
- `E2E: calendar respects user's timezone setting for post placement`

---

## Phase 4: Unified Inbox & Engagement

### Purpose

Build the unified inbox that aggregates comments, DMs, and mentions from all connected platforms into a single actionable stream. After this phase, users can read, reply to, and manage engagement across all their social accounts from one view.

### Tasks

#### 4.1 — Inbox Message Sync Worker

**What**: Build the background worker that polls each connected social account for new comments, DMs, and mentions, normalises them into the inbox_messages table.

**Design**:

```typescript
// src/server/workers/inbox-worker.ts
export class InboxSyncWorker {
  constructor(
    private registry: PlatformRegistry,
    private tokenManager: TokenManager,
    private db: DrizzleClient,
  ) {}

  async syncAccount(accountId: string): Promise<number> {
    const account = await this.db.select().from(socialAccounts).where(eq(socialAccounts.id, accountId)).get();
    const adapter = this.registry.get(account.platform as PlatformCode);
    const accessToken = await this.tokenManager.refreshIfNeeded(accountId);

    // Get last sync cursor from account metadata
    const lastCursor = account.profile?.lastInboxCursor;
    const { messages, nextCursor } = await adapter.getInboxMessages(accessToken, undefined, lastCursor);

    // Upsert inbox_messages (deduplicate by platform_message_id)
    for (const msg of messages) {
      await this.db.insert(inboxMessages).values({
        tenantId: account.tenantId,
        socialAccountId: accountId,
        platform: account.platform,
        platformMessageId: msg.platformMessageId,
        messageType: msg.messageType,
        direction: msg.direction,
        author: {
          name: msg.authorName,
          handle: msg.authorHandle,
          avatar_url: msg.authorAvatarUrl,
          platform_uid: msg.authorPlatformUid,
        },
        bodyText: msg.bodyText,
        status: "open",
        receivedAt: msg.receivedAt,
      }).onConflictDoNothing();
    }

    // Update cursor
    if (nextCursor) {
      await this.db.update(socialAccounts).set({
        profile: sql`jsonb_set(profile, '{lastInboxCursor}', ${JSON.stringify(nextCursor)}::jsonb)`,
      }).where(eq(socialAccounts.id, accountId));
    }

    return messages.length;
  }
}

// BullMQ repeatable job configuration
// Poll every 2 minutes for each active social account
```

**Testing**:
- `Integration (mocked): sync account with 5 new messages → 5 inbox_messages rows created`
- `Integration (mocked): sync account with 0 new messages → no rows created, cursor updated`
- `Integration (mocked): sync account with duplicate messages → deduplication via ON CONFLICT DO NOTHING`
- `Integration (mocked): token expired during sync → auto-refreshed, sync completes`
- `Integration (mocked): platform API returns 429 → job retried with backoff`
- `Unit: message normalisation maps platform-specific formats to InboxMessage interface`
- `Integration: cursor persistence → second sync starts from where first left off`

---

#### 4.2 — Inbox UI & Reply Flow

**What**: Build the unified inbox view with message list, threaded conversation view, reply composer, and assignment/status management.

**Design**:

```typescript
// src/components/inbox/message-list.tsx
interface MessageListProps {
  filters: InboxFilters;
  onSelect: (messageId: string) => void;
}

interface InboxFilters {
  status: ("open" | "replied" | "resolved" | "archived")[];
  platforms: PlatformCode[];
  accountIds: string[];
  sentiment: ("positive" | "negative" | "neutral")[];
  assignedTo: string | "unassigned" | "me";
  search: string;
  dateRange: { from: Date; to: Date };
}

// src/components/inbox/message-thread.tsx
interface MessageThreadProps {
  rootMessageId: string;
  onReply: (text: string) => void;
  onAssign: (userId: string) => void;
  onResolve: () => void;
}

// src/server/trpc/routers/inbox.ts
export const inboxRouter = router({
  list: protectedProcedure
    .input(InboxFiltersSchema)
    .query(async ({ ctx, input }) => {
      // Filtered, paginated query against inbox_messages
      // Include author JSONB, sentiment JSONB, platform info
    }),

  reply: protectedProcedure
    .input(z.object({
      messageId: z.string().uuid(),
      replyText: z.string().min(1).max(10_000),
    }))
    .mutation(async ({ ctx, input }) => {
      // Get original message's platform and account
      // Call adapter.replyToMessage()
      // Insert outbound inbox_message row
      // Update original message status to "replied"
    }),

  assign: protectedProcedure
    .input(z.object({
      messageId: z.string().uuid(),
      assignTo: z.string().uuid(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Update assigned_to on inbox_message
      // Write audit_log entry
    }),

  resolve: protectedProcedure
    .input(z.object({ messageId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => {
      // Update status to "resolved"
    }),

  bulkAction: protectedProcedure
    .input(z.object({
      messageIds: z.array(z.string().uuid()),
      action: z.enum(["resolve", "archive", "mark_read"]),
    }))
    .mutation(async ({ ctx, input }) => {
      // Batch update
    }),
});
```

**Testing**:
- `E2E: inbox displays messages from all connected platforms with correct platform icons`
- `E2E: filter by platform → only messages from selected platform shown`
- `E2E: filter by sentiment → only messages with matching sentiment shown`
- `E2E: click message → thread view opens with conversation history`
- `E2E: type reply and submit → reply appears in thread, message status changes to "replied"`
- `Integration (mocked): reply mutation calls adapter.replyToMessage() and creates outbound message row`
- `E2E: assign message to team member → assignee name shown on message`
- `E2E: resolve message → message moves to "resolved" filter`
- `E2E: bulk select messages → bulk resolve updates all selected`
- `E2E: search by keyword → messages containing keyword shown`

---

## Phase 5: Analytics Dashboard

### Purpose

Build the analytics collection pipeline and dashboard that displays per-post and per-account metrics from all connected platforms. After this phase, users can track impressions, engagement, reach, and follower growth across all accounts with time-series visualisation.

### Tasks

#### 5.1 — Analytics Snapshot Collector

**What**: Build the background worker that periodically fetches analytics from each platform API and stores them as timestamped snapshots.

**Design**:

```typescript
// src/server/workers/analytics-worker.ts
export class AnalyticsCollectorWorker {
  constructor(
    private registry: PlatformRegistry,
    private tokenManager: TokenManager,
    private db: DrizzleClient,
  ) {}

  async collectPostAnalytics(postResultId: string): Promise<void> {
    const result = await this.db.select().from(postResults)
      .where(eq(postResults.id, postResultId)).get();
    if (!result?.platformPostId) return;

    const adapter = this.registry.get(result.platform as PlatformCode);
    const accessToken = await this.tokenManager.refreshIfNeeded(result.socialAccountId);
    const snapshot = await adapter.getPostAnalytics(accessToken, result.platformPostId);

    await this.db.insert(analyticsSnapshots).values({
      tenantId: result.tenantId,
      targetType: "post",
      targetId: postResultId,
      snapshotAt: new Date(),
      metrics: snapshot,
    });
  }

  async collectAccountAnalytics(accountId: string): Promise<void> {
    const account = await this.db.select().from(socialAccounts).where(eq(socialAccounts.id, accountId)).get();
    const adapter = this.registry.get(account.platform as PlatformCode);
    const accessToken = await this.tokenManager.refreshIfNeeded(accountId);

    const today = new Date();
    const yesterday = new Date(today.getTime() - 86_400_000);
    const snapshot = await adapter.getAccountAnalytics(accessToken, account.platformUid, yesterday, today);

    await this.db.insert(analyticsSnapshots).values({
      tenantId: account.tenantId,
      targetType: "account",
      targetId: accountId,
      snapshotAt: today,
      metrics: snapshot,
    });
  }
}

// Collection schedule:
// - Post analytics: every 4 hours for first 7 days, then daily for 30 days
// - Account analytics: daily at 00:05 UTC

// src/server/db/schema/analytics-snapshots.ts
export const analyticsSnapshots = pgTable("analytics_snapshots", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull(),
  targetType: varchar("target_type", { length: 30 }).notNull(), // "post" | "account"
  targetId: uuid("target_id").notNull(),
  snapshotAt: timestamp("snapshot_at", { withTimezone: true }).notNull(),
  metrics: jsonb("metrics").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- `Integration (mocked): collectPostAnalytics() → calls adapter.getPostAnalytics(), inserts snapshot row`
- `Integration (mocked): collectAccountAnalytics() → calls adapter.getAccountAnalytics(), inserts snapshot row`
- `Integration (mocked): API returns 429 → job retried with backoff`
- `Integration: multiple snapshots for same post → each gets unique timestamp, all stored`
- `Unit: collection schedule — posts within 7 days get 4-hour intervals, older posts get daily`
- `Integration: account with expired token → token refreshed before collection`

---

#### 5.2 — Analytics Dashboard UI

**What**: Build the analytics overview page with time-series charts, metric cards, platform breakdown, and per-post ranking.

**Design**:

```typescript
// src/server/trpc/routers/analytics.ts
export const analyticsRouter = router({
  overview: protectedProcedure
    .input(z.object({
      dateRange: z.object({ from: z.date(), to: z.date() }),
      accountIds: z.array(z.string().uuid()).optional(),
      granularity: z.enum(["hourly", "daily", "weekly"]),
    }))
    .query(async ({ ctx, input }) => {
      // Aggregate analytics_snapshots for the tenant
      // Return: totals, time-series arrays, platform breakdown
      return {
        totals: { impressions, reach, engagements, followers },
        timeSeries: [{ date, impressions, reach, engagements }],
        byPlatform: [{ platform, impressions, reach, engagements }],
      };
    }),

  topPosts: protectedProcedure
    .input(z.object({
      dateRange: z.object({ from: z.date(), to: z.date() }),
      sortBy: z.enum(["impressions", "engagements", "clicks"]),
      limit: z.number().min(1).max(50).default(10),
    }))
    .query(async ({ ctx, input }) => {
      // Latest snapshot for each published post, ranked by sortBy metric
    }),

  postDetail: protectedProcedure
    .input(z.object({ postId: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      // All snapshots for a post over time — enables engagement curve chart
    }),
});

// src/components/analytics/metrics-card.tsx
interface MetricsCardProps {
  label: string;
  value: number;
  previousValue: number;  // for delta calculation
  format: "number" | "percentage";
}

// src/components/analytics/engagement-chart.tsx
// Recharts-based line chart showing impressions, reach, engagements over time
// Supports overlay of multiple metrics on same axes
// Granularity selector (hourly/daily/weekly)
```

**Testing**:
- `E2E: analytics page shows metric cards with current period totals`
- `E2E: metric cards show delta percentage vs previous period`
- `E2E: time-series chart renders data points for selected date range`
- `E2E: change date range → chart and cards update`
- `E2E: platform breakdown shows per-platform engagement bars`
- `E2E: top posts table shows ranked posts with engagement metrics`
- `E2E: click post row → navigates to post detail with engagement curve`
- `Integration: overview query with no data → returns zeroes, no errors`
- `Integration: overview query aggregates JSONB metrics correctly across snapshots`
- `Unit: MetricsCard calculates delta percentage correctly (positive, negative, zero previous)`

---

#### 5.3 — Reporting & Export

**What**: Build scheduled team reports and CSV/PDF export of analytics data.

**Design**:

```typescript
// src/server/services/analytics/reports.ts
export class ReportService {
  async generateReport(
    tenantId: string,
    config: ReportConfig,
  ): Promise<{ csv: string; pdf?: Buffer }> {
    const data = await this.analyticsRouter.overview({
      dateRange: config.dateRange,
      accountIds: config.accountIds,
      granularity: config.granularity,
    });

    const csv = this.formatCSV(data);
    const pdf = config.includePdf ? await this.generatePDF(data, config) : undefined;
    return { csv, pdf };
  }

  private formatCSV(data: AnalyticsOverview): string {
    // RFC 4180 compliant CSV with headers:
    // Date, Platform, Impressions, Reach, Engagements, Likes, Comments, Shares, Clicks
  }

  private async generatePDF(data: AnalyticsOverview, config: ReportConfig): Promise<Buffer> {
    // Use @react-pdf/renderer for branded PDF report with charts
  }
}

interface ReportConfig {
  dateRange: { from: Date; to: Date };
  accountIds?: string[];
  granularity: "daily" | "weekly";
  includePdf: boolean;
  recipientEmails?: string[];
}
```

**Testing**:
- `Unit: formatCSV() produces valid RFC 4180 CSV with correct headers`
- `Unit: formatCSV() handles special characters (commas, quotes) in data`
- `Unit: generatePDF() produces valid PDF buffer with non-zero length`
- `Integration: generateReport() with date range → CSV contains data for each day in range`
- `E2E: click "Export CSV" → file downloads with correct filename`
- `E2E: click "Export PDF" → branded PDF downloads`

---

## Phase 6: AI Content Generation

### Purpose

Integrate AI-powered content creation features: text generation with tone control, content rewriting, hashtag suggestions, and multi-variant generation. After this phase, users can use AI to draft, improve, and customise post content directly in the composer.

### Tasks

#### 6.1 — AI Content Service

**What**: Build the AI content generation service with provider abstraction (Claude primary, OpenAI fallback), tone control, and multi-variant output.

**Design**:

```typescript
// src/server/services/ai/content-generator.ts
import Anthropic from "@anthropic-ai/sdk";

export class AIContentGenerator {
  constructor(
    private client: Anthropic,
    private db: DrizzleClient,
  ) {}

  async generate(request: AIContentRequest): Promise<AIContentResponse> {
    const systemPrompt = this.buildSystemPrompt(request);
    const userPrompt = this.buildUserPrompt(request);

    const startTime = Date.now();
    const response = await this.client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 2048,
      system: systemPrompt,
      messages: [{ role: "user", content: userPrompt }],
    });

    const result = this.parseResponse(response, request);

    // Log to ai_interactions table
    await this.db.insert(aiInteractions).values({
      tenantId: request.tenantId,
      requestedBy: request.userId,
      interactionType: request.type,
      request: { type: request.type, prompt: userPrompt, tone: request.tone, model: "claude-sonnet-4" },
      response: { outputText: result.text, variants: result.variants, tokensUsed: response.usage.output_tokens, latencyMs: Date.now() - startTime },
      status: "completed",
      completedAt: new Date(),
    });

    return result;
  }

  private buildSystemPrompt(request: AIContentRequest): string {
    return `You are a social media content creator. Generate engaging content for ${request.targetPlatforms.join(", ")}.
Tone: ${request.tone ?? "professional"}
${request.brandVoice ? `Brand voice guidelines: ${request.brandVoice}` : ""}
${request.platformConstraints ? `Character limits: ${JSON.stringify(request.platformConstraints)}` : ""}
Respond with a JSON object: { "text": "...", "variants": [{"text": "...", "tone": "..."}], "hashtags": ["..."] }`;
  }

  private buildUserPrompt(request: AIContentRequest): string {
    switch (request.type) {
      case "generate": return `Create a post about: ${request.topic}`;
      case "rewrite": return `Rewrite this post with a ${request.tone} tone:\n\n${request.inputText}`;
      case "tone_adjust": return `Adjust this post to be more ${request.tone}:\n\n${request.inputText}`;
      case "suggest_hashtags": return `Suggest relevant hashtags for this post:\n\n${request.inputText}`;
    }
  }
}

interface AIContentRequest {
  tenantId: string;
  userId: string;
  type: "generate" | "rewrite" | "tone_adjust" | "suggest_hashtags";
  topic?: string;
  inputText?: string;
  tone?: "formal" | "casual" | "witty" | "professional" | "empathetic";
  targetPlatforms: PlatformCode[];
  platformConstraints?: Record<PlatformCode, { maxChars: number }>;
  brandVoice?: string;
  variantCount?: number;
}

interface AIContentResponse {
  text: string;
  variants: { text: string; tone: string }[];
  hashtags: string[];
  tokensUsed: number;
  latencyMs: number;
}
```

**Testing**:
- `Integration (mocked LLM): generate request → returns text, variants, and hashtags`
- `Integration (mocked LLM): rewrite request → returns modified text preserving meaning`
- `Integration (mocked LLM): tone_adjust request → returns text with adjusted tone`
- `Integration (mocked LLM): suggest_hashtags request → returns relevant hashtag array`
- `Unit: buildSystemPrompt() includes platform constraints when provided`
- `Unit: buildSystemPrompt() includes brand voice when provided`
- `Integration: ai_interactions row created with request, response, latency, and token count`
- `Integration (mocked LLM): LLM returns malformed JSON → graceful error, status "failed" logged`
- `Unit: parseResponse() handles JSON embedded in markdown code blocks`

---

#### 6.2 — AI Content UI Integration

**What**: Add AI content generation controls to the post composer: generate button, tone selector, variant picker, and hashtag suggestions.

**Design**:

```typescript
// src/components/composer/ai-assistant.tsx
interface AIAssistantProps {
  currentText: string;
  targetPlatforms: PlatformCode[];
  onInsert: (text: string) => void;
  onAppendHashtags: (hashtags: string[]) => void;
}

// UI flow:
// 1. User clicks "AI Generate" button in composer toolbar
// 2. Modal/drawer opens with:
//    - Topic input (for new generation) or shows current text (for rewrite/adjust)
//    - Tone selector dropdown (formal, casual, witty, professional, empathetic)
//    - "Generate" / "Rewrite" / "Adjust Tone" action buttons
// 3. Generated content shows primary text + variant cards
// 4. User clicks "Use This" on preferred variant → inserts into composer
// 5. "Suggest Hashtags" button adds hashtags to post body or first comment

// src/server/trpc/routers/ai.ts
export const aiRouter = router({
  generate: protectedProcedure
    .input(z.object({
      type: z.enum(["generate", "rewrite", "tone_adjust", "suggest_hashtags"]),
      topic: z.string().optional(),
      inputText: z.string().optional(),
      tone: z.enum(["formal", "casual", "witty", "professional", "empathetic"]).optional(),
      targetPlatforms: z.array(PlatformCode),
      variantCount: z.number().min(1).max(5).default(3),
    }))
    .mutation(async ({ ctx, input }) => {
      // Call AIContentGenerator.generate()
      // Rate limit: max 50 AI requests per tenant per day on free tier
    }),
});
```

**Testing**:
- `E2E: click "AI Generate" in composer → modal opens with topic input and tone selector`
- `E2E: enter topic, select tone, click Generate → loading state shown → variants displayed`
- `E2E: click "Use This" on variant → text inserted into post body`
- `E2E: click "Suggest Hashtags" → hashtags appended to post`
- `E2E: rewrite mode → current post text shown, rewritten variants displayed`
- `Integration: rate limit exceeded → user-friendly error message shown`
- `E2E: AI assistant respects platform character limits in generated content`

---

## Phase 7: Sentiment Analysis & Message Routing

### Purpose

Add sentiment analysis to all incoming inbox messages and listening matches, and build automated routing rules that assign messages to team members based on sentiment and other criteria. After this phase, negative messages are automatically escalated to senior team members.

### Tasks

#### 7.1 — Sentiment Analysis Pipeline

**What**: Build the sentiment analysis service that processes inbox messages and listening matches, storing rich sentiment data (label, score, emotions, topics) in the JSONB sentiment column.

**Design**:

```typescript
// src/server/services/ai/sentiment.ts
export class SentimentService {
  constructor(
    private client: Anthropic,
    private db: DrizzleClient,
  ) {}

  async analyze(text: string, language?: string): Promise<SentimentResult> {
    const response = await this.client.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 512,
      system: `You are a sentiment analysis system. Analyze the following social media message.
Return a JSON object with:
- label: "positive" | "negative" | "neutral" | "mixed"
- score: number between -1.0 (most negative) and 1.0 (most positive)
- emotions: object with emotion names as keys and confidence 0-1 as values (e.g. {"anger": 0.8, "frustration": 0.6})
- topics: array of topic keywords detected in the message
- language: ISO 639-1 code of the detected language
Respond ONLY with valid JSON.`,
      messages: [{ role: "user", content: text }],
    });

    return this.parseSentimentResponse(response);
  }

  async batchAnalyze(messageIds: string[]): Promise<void> {
    const messages = await this.db.select()
      .from(inboxMessages)
      .where(inArray(inboxMessages.id, messageIds))
      .where(isNull(inboxMessages.sentiment));

    for (const msg of messages) {
      const result = await this.analyze(msg.bodyText, msg.author?.language);
      await this.db.update(inboxMessages).set({
        sentiment: result,
        updatedAt: new Date(),
      }).where(eq(inboxMessages.id, msg.id));
    }
  }
}

interface SentimentResult {
  label: "positive" | "negative" | "neutral" | "mixed";
  score: number;         // -1.0 to 1.0
  emotions: Record<string, number>;  // emotion → confidence
  topics: string[];
  language: string;      // ISO 639-1
  modelVersion: string;
}

// src/server/workers/sentiment-worker.ts
// BullMQ worker that processes new inbox messages through SentimentService
// Triggered by inbox-worker after each sync batch
// Rate limited to prevent LLM cost spikes: max 100 analyses per minute per tenant
```

**Testing**:
- `Integration (mocked LLM): analyze() with positive text → returns label "positive", score > 0`
- `Integration (mocked LLM): analyze() with negative text → returns label "negative", score < 0, anger emotion > 0.5`
- `Integration (mocked LLM): analyze() with mixed text → returns label "mixed", score near 0`
- `Unit: parseSentimentResponse() handles malformed JSON gracefully`
- `Integration: batchAnalyze() skips messages that already have sentiment`
- `Integration: batchAnalyze() updates inbox_messages.sentiment JSONB column`
- `Integration: sentiment worker processes queue within rate limits`
- `Unit: SentimentResult conforms to the JSONB schema documented in data model`

---

#### 7.2 — Inbox Routing Engine

**What**: Build the automated routing engine that evaluates rules against incoming messages and executes actions (assign, label, escalate, auto-reply).

**Design**:

```typescript
// src/server/services/inbox/routing-engine.ts
interface RoutingRule {
  id: string;
  name: string;
  priority: number;
  conditions: RoutingCondition[];
  action: RoutingAction;
  isActive: boolean;
}

interface RoutingCondition {
  field: "sentiment.label" | "sentiment.score" | "platform" | "message_type" | "author.follower_count" | "body_text";
  operator: "equals" | "not_equals" | "contains" | "greater_than" | "less_than" | "in";
  value: string | number | string[];
}

interface RoutingAction {
  type: "assign_to" | "add_label" | "escalate" | "auto_reply";
  config: {
    userId?: string;         // for assign_to
    label?: string;          // for add_label
    escalationLevel?: number; // for escalate
    templateId?: string;     // for auto_reply
  };
}

export class RoutingEngine {
  async evaluateMessage(messageId: string): Promise<void> {
    const message = await this.db.select().from(inboxMessages).where(eq(inboxMessages.id, messageId)).get();
    const rules = await this.db.select().from(inboxRoutingRules)
      .where(eq(inboxRoutingRules.tenantId, message.tenantId))
      .where(eq(inboxRoutingRules.isActive, true))
      .orderBy(asc(inboxRoutingRules.priority));

    for (const rule of rules) {
      if (this.matchesConditions(message, rule.conditions)) {
        await this.executeAction(message, rule.action);
        // Update message routing_metadata JSONB with matched rule info
        break; // First matching rule wins
      }
    }
  }

  private matchesConditions(message: InboxMessage, conditions: RoutingCondition[]): boolean {
    return conditions.every(c => this.evaluateCondition(message, c));
  }

  private evaluateCondition(message: any, condition: RoutingCondition): boolean {
    const value = this.resolveField(message, condition.field);
    switch (condition.operator) {
      case "equals": return value === condition.value;
      case "not_equals": return value !== condition.value;
      case "contains": return String(value).includes(String(condition.value));
      case "greater_than": return Number(value) > Number(condition.value);
      case "less_than": return Number(value) < Number(condition.value);
      case "in": return (condition.value as string[]).includes(String(value));
    }
  }
}
```

**Testing**:
- `Unit: matchesConditions() with sentiment.label equals "negative" → matches negative message`
- `Unit: matchesConditions() with sentiment.score less_than -0.5 → matches strongly negative message`
- `Unit: matchesConditions() with platform equals "x" → matches X messages only`
- `Unit: matchesConditions() with multiple conditions → all must match (AND logic)`
- `Integration: evaluateMessage() with matching assign_to rule → message assigned_to updated`
- `Integration: evaluateMessage() with no matching rules → message unchanged`
- `Integration: rules evaluated in priority order → first match wins`
- `Integration: routing_metadata JSONB updated with matched_rules array`
- `E2E: create routing rule "Assign negative X messages to Senior Manager" → negative X message auto-assigned`

---

#### 7.3 — Sentiment Dashboard

**What**: Add sentiment visualisation to the analytics dashboard: sentiment distribution, sentiment over time, and message-level sentiment badges in the inbox.

**Design**:

```typescript
// src/components/inbox/sentiment-badge.tsx
interface SentimentBadgeProps {
  sentiment: SentimentResult | null;
  size: "sm" | "md";
}
// Colour-coded badge: green (positive), red (negative), grey (neutral), yellow (mixed)
// Tooltip shows score, top emotion, and detected topics

// src/server/trpc/routers/analytics.ts (additions)
export const analyticsRouter = router({
  // ... existing routes ...

  sentimentOverview: protectedProcedure
    .input(z.object({
      dateRange: z.object({ from: z.date(), to: z.date() }),
      accountIds: z.array(z.string().uuid()).optional(),
    }))
    .query(async ({ ctx, input }) => {
      // Aggregate sentiment from inbox_messages
      return {
        distribution: { positive: number, negative: number, neutral: number, mixed: number },
        timeSeries: [{ date: string, positive: number, negative: number, neutral: number }],
        topEmotions: [{ emotion: string, count: number }],
        topNegativeTopics: [{ topic: string, count: number }],
      };
    }),
});
```

**Testing**:
- `E2E: sentiment badge shows on each inbox message with correct colour`
- `E2E: hover over sentiment badge → tooltip shows score, emotions, topics`
- `E2E: analytics dashboard shows sentiment distribution pie chart`
- `E2E: sentiment over time chart shows daily breakdown`
- `E2E: top negative topics list shows most common complaint themes`
- `Integration: sentimentOverview query aggregates JSONB sentiment data correctly`
- `Integration: sentimentOverview with no analysed messages → returns zero distribution`

---

## Phase 8: Social Listening

### Purpose

Build the social listening feature that monitors brand mentions, competitor activity, and keyword tracking across platforms. After this phase, users can set up listening queries and receive real-time match notifications.

### Tasks

#### 8.1 — Listening Query Management

**What**: Build the CRUD for listening queries and the background worker that polls platform APIs for matches.

**Design**:

```typescript
// src/server/services/listening/listener.ts
export class SocialListeningService {
  constructor(
    private registry: PlatformRegistry,
    private tokenManager: TokenManager,
    private sentimentService: SentimentService,
    private db: DrizzleClient,
  ) {}

  async executeQuery(queryId: string): Promise<number> {
    const query = await this.db.select().from(listeningQueries).where(eq(listeningQueries.id, queryId)).get();
    const config = query.config as ListeningConfig;
    let totalMatches = 0;

    for (const platformCode of config.platforms) {
      const adapter = this.registry.get(platformCode as PlatformCode);
      if (!adapter.capabilities.includes("listening")) continue;

      // Platform-specific search:
      // X: GET /2/tweets/search/recent?query=...
      // Mastodon: GET /api/v1/timelines/tag/:hashtag or search API
      // Bluesky: app.bsky.feed.searchPosts via XRPC

      const matches = await this.searchPlatform(adapter, config);

      for (const match of matches) {
        const sentiment = await this.sentimentService.analyze(match.bodyText);
        await this.db.insert(listeningMatches).values({
          queryId,
          tenantId: query.tenantId,
          platform: platformCode,
          content: {
            platform_content_id: match.platformContentId,
            author_name: match.authorName,
            author_handle: match.authorHandle,
            body_text: match.bodyText,
            content_url: match.contentUrl,
            engagement: match.engagement,
            reach_estimate: match.reachEstimate,
          },
          sentiment,
          matchedAt: match.publishedAt,
        });
        totalMatches++;
      }
    }

    return totalMatches;
  }
}

interface ListeningConfig {
  queryType: "brand_mention" | "competitor" | "keyword" | "hashtag";
  keywords: string[];
  excludedKeywords: string[];
  platforms: string[];
  languageFilter?: string[];   // ISO 639-1
  countryFilter?: string[];    // ISO 3166-1
  minFollowerCount?: number;
}
```

**Testing**:
- `Integration (mocked): executeQuery() with brand mention keywords → matches found and stored`
- `Integration (mocked): executeQuery() with excluded keywords → matching but excluded content filtered out`
- `Integration (mocked): executeQuery() → sentiment analysed for each match`
- `Integration (mocked): duplicate match (same platform_content_id) → not inserted twice`
- `Integration: listening worker runs on 5-minute repeatable schedule`
- `Unit: ListeningConfig validation rejects empty keywords array`
- `Integration (mocked): platform without "listening" capability → skipped without error`

---

#### 8.2 — Listening Dashboard UI

**What**: Build the listening queries list and match results view with sentiment filtering, trend charts, and match detail cards.

**Design**:

```typescript
// src/server/trpc/routers/listening.ts
export const listeningRouter = router({
  listQueries: protectedProcedure
    .query(async ({ ctx }) => {
      // Return all listening queries for tenant with match counts
    }),

  createQuery: protectedProcedure
    .input(ListeningConfigSchema)
    .mutation(async ({ ctx, input }) => {
      // Insert listening_queries row
      // Trigger immediate first execution
    }),

  getMatches: protectedProcedure
    .input(z.object({
      queryId: z.string().uuid(),
      dateRange: z.object({ from: z.date(), to: z.date() }),
      sentimentFilter: z.array(z.enum(["positive", "negative", "neutral", "mixed"])).optional(),
      platform: PlatformCode.optional(),
      cursor: z.string().optional(),
      limit: z.number().min(1).max(100).default(20),
    }))
    .query(async ({ ctx, input }) => {
      // Paginated, filtered listening matches
    }),

  matchTrend: protectedProcedure
    .input(z.object({
      queryId: z.string().uuid(),
      dateRange: z.object({ from: z.date(), to: z.date() }),
      granularity: z.enum(["hourly", "daily"]),
    }))
    .query(async ({ ctx, input }) => {
      // Time-series of match count and sentiment distribution
    }),
});
```

**Testing**:
- `E2E: listening page shows list of configured queries with match counts`
- `E2E: click "New Query" → form with keyword, platform, and filter inputs`
- `E2E: click query → match results page with cards showing author, text, sentiment, platform`
- `E2E: filter matches by sentiment → list updates`
- `E2E: match trend chart shows volume over time with sentiment stacking`
- `E2E: match card links to original post on platform (opens in new tab)`
- `Integration: createQuery mutation → immediate first poll executed`

---

## Phase 9: Approval Workflows

### Purpose

Add content approval workflows so teams can review and approve posts before publication. After this phase, posts flow through configurable approval chains with step-by-step review, comments, and approval/rejection actions.

### Tasks

#### 9.1 — Approval Workflow Engine

**What**: Build the approval workflow system with multi-step review chains and status management.

**Design**:

```typescript
// src/server/services/approval/workflow-engine.ts
export class ApprovalWorkflowEngine {
  constructor(private db: DrizzleClient) {}

  async submitForApproval(postId: string, workflowId: string, requestedBy: string): Promise<void> {
    const workflow = await this.db.select().from(approvalWorkflows).where(eq(approvalWorkflows.id, workflowId)).get();
    const steps = workflow.steps as ApprovalStep[];

    await this.db.update(posts).set({
      status: "pending_approval",
      approvalStatus: "pending",
    }).where(eq(posts.id, postId));

    // Notify first step's approver(s)
  }

  async recordDecision(
    postId: string,
    decidedBy: string,
    decision: "approved" | "rejected" | "returned",
    comment?: string,
  ): Promise<void> {
    // Record decision
    // If approved and more steps remain → advance to next step
    // If approved and final step → update post status to "approved"
    // If rejected → update post status to "draft", notify author
  }
}

interface ApprovalStep {
  name: string;
  approverRole?: string;   // "admin" | "approver"
  approverUserId?: string; // specific user
  requireAll: boolean;     // all approvers must approve vs any one
}

// Database tables (from data-model-suggestion-1)
// approval_workflows: { id, tenant_id, name, steps JSONB, is_default }
// approval_requests: { id, post_id, workflow_id, current_step, status }
// approval_decisions: { id, request_id, step_index, decided_by, decision, comment }
```

**Testing**:
- `Integration: submitForApproval() → post status changes to "pending_approval"`
- `Integration: recordDecision("approved") on single-step workflow → post status "approved"`
- `Integration: recordDecision("approved") on multi-step workflow → advances to next step`
- `Integration: recordDecision("rejected") → post status back to "draft"`
- `Integration: recordDecision("returned") with comment → comment stored, post back to "draft"`
- `Unit: ApprovalStep with requireAll=true requires all approvers, not just one`
- `E2E: submit post for approval → post shows "Pending Approval" status`
- `E2E: approver sees pending post → can approve or reject with comment`
- `E2E: approved post can be scheduled or published immediately`

---

## Phase 10: Trend Alerts & Content Strategy AI

### Purpose

Build the proactive intelligence layer: trend detection from listening data, real-time alerts, content performance prediction before publishing, and AI-powered content strategy suggestions. These features represent the AI-native advantage over incumbents.

### Tasks

#### 10.1 — Trend Detection Engine

**What**: Build the background analysis that detects spikes in mentions, sentiment shifts, and competitor activity, generating trend alerts.

**Design**:

```typescript
// src/server/services/ai/trend-detector.ts
export class TrendDetector {
  constructor(
    private db: DrizzleClient,
    private aiClient: Anthropic,
  ) {}

  async detectTrends(tenantId: string): Promise<TrendAlert[]> {
    const alerts: TrendAlert[] = [];

    // 1. Volume spike detection
    const recentMatches = await this.getMatchCounts(tenantId, "24h");
    const baselineMatches = await this.getMatchCounts(tenantId, "7d_avg");
    for (const [queryId, count] of Object.entries(recentMatches)) {
      const baseline = baselineMatches[queryId] ?? 0;
      if (baseline > 0 && count > baseline * 3) {
        alerts.push({
          tenantId,
          alertType: "trending_topic",
          title: `Mention spike: ${count} mentions in 24h (${Math.round(count/baseline)}x normal)`,
          severity: count > baseline * 10 ? "critical" : "warning",
        });
      }
    }

    // 2. Sentiment shift detection
    const recentSentiment = await this.getAverageSentiment(tenantId, "24h");
    const baselineSentiment = await this.getAverageSentiment(tenantId, "7d");
    if (recentSentiment.score - baselineSentiment.score < -0.3) {
      alerts.push({
        tenantId,
        alertType: "sentiment_shift",
        title: "Negative sentiment shift detected",
        severity: "warning",
      });
    }

    // 3. Competitor activity spike
    // Compare competitor listening query volumes

    return alerts;
  }
}
```

**Testing**:
- `Unit: volume spike detected when 24h count > 3x 7d average`
- `Unit: no alert when volume is within normal range`
- `Unit: sentiment shift detected when 24h average drops > 0.3 from 7d average`
- `Unit: severity "critical" assigned for > 10x volume spikes`
- `Integration: detectTrends() inserts trend_alerts rows`
- `Integration: trend detector runs on hourly schedule`
- `E2E: trend alert appears on alerts page with severity badge`

---

#### 10.2 — Content Performance Prediction

**What**: Build the pre-publish prediction service that estimates expected impressions, engagements, and clicks based on historical post performance.

**Design**:

```typescript
// src/server/services/ai/predictions.ts
export class PerformancePredictionService {
  constructor(
    private db: DrizzleClient,
    private aiClient: Anthropic,
  ) {}

  async predictPerformance(
    tenantId: string,
    draft: { bodyText: string; contentType: string; targetPlatforms: PlatformCode[]; scheduledAt?: Date },
  ): Promise<PerformancePrediction> {
    // 1. Gather historical data: past 90 days of post analytics for this tenant
    const historicalPosts = await this.getHistoricalPerformance(tenantId, 90);

    // 2. Extract features: post length, content type, day of week, hour, platform, has media, hashtag count
    const features = this.extractFeatures(draft, historicalPosts);

    // 3. Use Claude to predict with historical context
    const prediction = await this.aiClient.messages.create({
      model: "claude-sonnet-4-20250514",
      max_tokens: 512,
      system: `You are a social media analytics expert. Based on the historical performance data and the draft post details, predict the expected performance.
Return JSON: { "predicted_impressions": number, "predicted_engagements": number, "predicted_clicks": number, "confidence": 0-1, "reasoning": "...", "suggestions": ["..."] }`,
      messages: [{ role: "user", content: JSON.stringify(features) }],
    });

    return this.parsePrediction(prediction);
  }
}

interface PerformancePrediction {
  predictedImpressions: number;
  predictedEngagements: number;
  predictedClicks: number;
  confidence: number;        // 0-1
  reasoning: string;
  suggestions: string[];     // improvement suggestions
}
```

**Testing**:
- `Integration (mocked LLM): predictPerformance() returns prediction with all fields`
- `Unit: extractFeatures() computes correct day-of-week, hour, post length, hashtag count`
- `Integration: prediction stored in ai_interactions table`
- `Integration: prediction with no historical data → low confidence score`
- `E2E: composer shows predicted engagement before publishing`
- `E2E: prediction card shows reasoning and improvement suggestions`

---

#### 10.3 — AI Strategy Assistant

**What**: Build the content strategy page where AI analyses historical performance, competitor activity, and audience patterns to suggest a content plan.

**Design**:

```typescript
// src/server/trpc/routers/ai.ts (additions)
export const aiRouter = router({
  // ... existing routes ...

  generateStrategy: protectedProcedure
    .input(z.object({
      dateRange: z.object({ from: z.date(), to: z.date() }),
      targetPlatforms: z.array(PlatformCode),
      goals: z.array(z.enum(["engagement", "reach", "followers", "traffic", "sales"])),
    }))
    .mutation(async ({ ctx, input }) => {
      // Gather: top performing posts, sentiment trends, competitor mentions, posting frequency
      // AI generates: recommended posting schedule, content themes, platform mix, tone guidelines
      return {
        recommendations: ContentStrategyRecommendation[],
        suggestedCalendar: SuggestedPost[],
        insights: string[],
      };
    }),
});

interface ContentStrategyRecommendation {
  category: "timing" | "content_type" | "platform_mix" | "tone" | "frequency";
  recommendation: string;
  rationale: string;
  confidence: number;
}

interface SuggestedPost {
  suggestedDate: Date;
  platform: PlatformCode;
  contentType: string;
  topicSuggestion: string;
  toneSuggestion: string;
}
```

**Testing**:
- `Integration (mocked LLM): generateStrategy() returns recommendations and suggested calendar`
- `Unit: strategy considers top-performing post patterns (time, type, tone)`
- `E2E: strategy page shows recommendation cards grouped by category`
- `E2E: suggested calendar shows proposed posts on a timeline`
- `E2E: click "Create Draft" on suggested post → opens composer pre-filled`

---

## Phase 11: Additional Platform Adapters

### Purpose

Expand platform coverage by implementing adapters for TikTok, YouTube, Pinterest, Mastodon, and Bluesky. After this phase, users can publish, read analytics, and manage inbox messages across all 10 supported platforms.

### Tasks

#### 11.1 — TikTok Adapter

**What**: Implement the PlatformAdapter for TikTok using the Content Posting API and Business API.

**Design**:

```typescript
// src/server/services/platforms/tiktok.ts
export class TikTokAdapter implements PlatformAdapter {
  readonly code = "tiktok" as const;
  readonly displayName = "TikTok";
  readonly capabilities: PlatformCapability[] = ["publish", "analytics"];

  private readonly apiBase = "https://open.tiktokapis.com/v2";

  async publish(accessToken: string, request: PublishRequest): Promise<PublishResult> {
    // 1. POST /post/publish/inbox/video/init/ with video source
    // 2. Upload via PULL_FROM_URL or FILE_UPLOAD
    // 3. Poll status until published
    // TikTok requires partner approval for Content Posting scopes
  }

  async getPostAnalytics(accessToken: string, platformPostId: string): Promise<AnalyticsSnapshot> {
    // GET /video/query/?filters[video_ids]=...&fields=...
  }
}
```

**Testing**:
- `Integration (mocked): publish() uploads video and returns TikTok video ID`
- `Integration (mocked): publish() non-video content → throws PlatformConstraintError("TikTok requires video")`
- `Integration (mocked): getPostAnalytics() returns parsed TikTok metrics`
- `Fixture: tiktok-api-responses/video-published.json, video-metrics.json`

---

#### 11.2 — YouTube, Pinterest, Mastodon, and Bluesky Adapters

**What**: Implement PlatformAdapter for each remaining platform.

**Design**:

```typescript
// src/server/services/platforms/youtube.ts
export class YouTubeAdapter implements PlatformAdapter {
  readonly code = "youtube" as const;
  readonly capabilities: PlatformCapability[] = ["publish", "analytics", "inbox"];
  // YouTube Data API v3 + resumable uploads
}

// src/server/services/platforms/pinterest.ts
export class PinterestAdapter implements PlatformAdapter {
  readonly code = "pinterest" as const;
  readonly capabilities: PlatformCapability[] = ["publish", "analytics"];
  // Pinterest API v5 with OAuth 2.0 + PKCE
}

// src/server/services/platforms/mastodon.ts
export class MastodonAdapter implements PlatformAdapter {
  readonly code = "mastodon" as const;
  readonly capabilities: PlatformCapability[] = ["publish", "analytics", "inbox", "listening"];
  // Mastodon REST API + ActivityPub
  // Instance URL stored in credentials JSONB (per-instance OAuth registration)
  // Supports content warnings, visibility levels, language tags
}

// src/server/services/platforms/bluesky.ts
export class BlueskyAdapter implements PlatformAdapter {
  readonly code = "bluesky" as const;
  readonly capabilities: PlatformCapability[] = ["publish", "listening"];
  // AT Protocol / XRPC
  // app.bsky.feed.post for publishing
  // app.bsky.feed.searchPosts for listening
  // DID-based identity
}
```

**Testing**:
- `Integration (mocked): each adapter's publish() → correct API call format for its platform`
- `Integration (mocked): each adapter's getOAuthConfig() → returns correct URLs and scopes`
- `Integration (mocked): Mastodon adapter handles instance-specific OAuth registration`
- `Integration (mocked): Mastodon publish with content_warning → CW field set in API call`
- `Integration (mocked): Bluesky publish → creates AT Protocol record via XRPC`
- `Integration (mocked): YouTube resumable upload → handles chunked upload protocol`
- `Fixture: response fixtures for each platform (youtube, pinterest, mastodon, bluesky)`

---

## Phase 12: MCP Server, Attribution & Self-Hosting

### Purpose

Build the MCP server for AI-assisted scheduling, implement cross-platform attribution tracking, and prepare the self-hosting package. After this phase, the platform is feature-complete for v1.0 and can be deployed both as managed SaaS and self-hosted.

### Tasks

#### 12.1 — MCP Server Implementation

**What**: Build the MCP server exposing scheduling, drafting, analytics, and inbox tools for LLM clients (Claude Desktop, IDEs, agent runtimes).

**Design**:

```typescript
// src/server/services/mcp/server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";

export function createMCPServer(services: ServiceContainer): Server {
  const server = new Server({ name: "social-media-management", version: "1.0.0" }, {
    capabilities: { tools: {}, resources: {}, prompts: {} },
  });

  // Tools (actions)
  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: [
      {
        name: "schedule_post",
        description: "Schedule a social media post for publishing",
        inputSchema: {
          type: "object",
          properties: {
            bodyText: { type: "string", description: "Post text content" },
            platforms: { type: "array", items: { type: "string", enum: ["facebook", "instagram", "x", "linkedin"] } },
            scheduledAt: { type: "string", format: "date-time", description: "ISO 8601 scheduled time" },
            tone: { type: "string", enum: ["formal", "casual", "witty", "professional"] },
          },
          required: ["bodyText", "platforms", "scheduledAt"],
        },
      },
      {
        name: "reply_to_mention",
        description: "Reply to an inbox message or mention",
        inputSchema: {
          type: "object",
          properties: {
            messageId: { type: "string" },
            replyText: { type: "string" },
          },
          required: ["messageId", "replyText"],
        },
      },
      {
        name: "get_analytics_summary",
        description: "Get analytics summary for a date range",
        inputSchema: {
          type: "object",
          properties: {
            dateRange: { type: "string", enum: ["today", "7d", "30d", "90d"] },
          },
          required: ["dateRange"],
        },
      },
    ],
  }));

  // Resources (read-only data)
  server.setRequestHandler(ListResourcesRequestSchema, async () => ({
    resources: [
      { uri: "smm://posts/drafts", name: "Draft Posts", description: "Current draft posts" },
      { uri: "smm://inbox/unread", name: "Unread Messages", description: "Unread inbox messages" },
      { uri: "smm://analytics/today", name: "Today's Analytics", description: "Today's performance metrics" },
      { uri: "smm://alerts/unread", name: "Unread Alerts", description: "Unread trend alerts" },
    ],
  }));

  // Prompts (pre-baked workflows)
  server.setRequestHandler(ListPromptsRequestSchema, async () => ({
    prompts: [
      { name: "weekly_recap", description: "Generate a weekly performance recap" },
      { name: "brand_voice_rewrite", description: "Rewrite content in brand voice", arguments: [{ name: "text", required: true }] },
      { name: "sentiment_digest", description: "Summarize inbox sentiment trends" },
    ],
  }));

  return server;
}
```

**Testing**:
- `Integration: MCP server lists all tools with correct input schemas`
- `Integration: schedule_post tool creates post and schedules it`
- `Integration: reply_to_mention tool sends reply via platform adapter`
- `Integration: get_analytics_summary tool returns formatted analytics`
- `Integration: resources return current data (drafts, unread messages)`
- `Integration: prompts return formatted prompt content`
- `Integration: Streamable HTTP transport handles request/response correctly`
- `Unit: MCP tool input validation rejects malformed requests`

---

#### 12.2 — Attribution Tracking

**What**: Implement cross-platform attribution that correlates social posts with downstream website visits and conversions via UTM parameters and a tracking pixel.

**Design**:

```typescript
// src/server/services/analytics/attribution.ts
export class AttributionService {
  constructor(private db: DrizzleClient) {}

  async trackEvent(event: AttributionEvent): Promise<void> {
    // Parse UTM parameters from landing URL
    // Match utm_campaign + utm_content to post_results via metadata.utm_params
    await this.db.insert(attributionEvents).values({
      tenantId: event.tenantId,
      postResultId: event.postResultId, // resolved from UTM match
      eventType: event.eventType,
      eventData: {
        source_url: event.sourceUrl,
        landing_url: event.landingUrl,
        utm: event.utmParams,
        conversion: event.conversion,
        session_id: event.sessionId,
      },
      occurredAt: event.occurredAt,
    });
  }

  async getAttributionReport(
    tenantId: string,
    dateRange: { from: Date; to: Date },
  ): Promise<AttributionReport> {
    // Aggregate attribution_events by post, platform, and conversion type
    // Calculate: traffic driven, conversions, revenue attributed
    return {
      byPost: [{ postId, platform, pageViews, conversions, revenue }],
      byPlatform: [{ platform, pageViews, conversions, revenue }],
      totalRevenue: number,
      totalConversions: number,
    };
  }
}

// Public endpoint for tracking pixel / JS snippet
// POST /api/v1/track
// Body: { tenant_id, event_type, source_url, landing_url, session_id, conversion? }
```

**Testing**:
- `Integration: trackEvent() with UTM params → attribution_events row created`
- `Integration: trackEvent() matches UTM campaign to correct post_result`
- `Integration: getAttributionReport() aggregates by post and platform correctly`
- `Unit: UTM parsing handles encoded characters and edge cases`
- `Integration: tracking endpoint accepts events and stores them`
- `E2E: analytics dashboard shows attribution data alongside engagement metrics`

---

#### 12.3 — Self-Hosting Package & Documentation

**What**: Prepare Docker images, docker-compose configuration, and documentation for self-hosted deployment.

**Design**:

```dockerfile
# Dockerfile (multi-stage production build)
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

```yaml
# docker-compose.production.yml
version: "3.9"
services:
  app:
    image: ghcr.io/social-media-management/smm:latest
    ports: ["3000:3000"]
    env_file: .env
    depends_on: [postgres, redis, minio]

  worker:
    image: ghcr.io/social-media-management/smm:latest
    command: ["node", "workers/index.js"]
    env_file: .env
    depends_on: [postgres, redis]

  postgres:
    image: postgres:16-alpine
    volumes: ["pg_data:/var/lib/postgresql/data"]
    environment:
      POSTGRES_DB: smm
      POSTGRES_USER: smm
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  redis:
    image: redis:7-alpine
    volumes: ["redis_data:/data"]

  minio:
    image: minio/minio:latest
    command: server /data
    volumes: ["minio_data:/data"]
    environment:
      MINIO_ROOT_USER: ${S3_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${S3_SECRET_KEY}

volumes:
  pg_data:
  redis_data:
  minio_data:
```

**Testing**:
- `Integration: Docker build completes without errors`
- `Integration: docker compose up with production config → all services start`
- `Integration: app container serves Next.js on port 3000`
- `Integration: worker container processes BullMQ jobs`
- `Integration: database migrations run on first boot`
- `Integration: MinIO bucket auto-created on startup`
- `E2E: full self-hosted deployment → create tenant, connect account, schedule post`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Project Scaffold       ─── required by everything
    │
Phase 2: Social Account Connection           ─── requires Phase 1
    │
Phase 3: Post Composition & Scheduling       ─── requires Phase 2
    │
    ├── Phase 4: Unified Inbox & Engagement   ─── requires Phase 2; can parallel with Phase 3
    │
    ├── Phase 5: Analytics Dashboard          ─── requires Phase 3; can parallel with Phase 4
    │
    └── Phase 6: AI Content Generation        ─── requires Phase 3; can parallel with Phases 4, 5
         │
Phase 7: Sentiment Analysis & Routing        ─── requires Phases 4, 6
    │
Phase 8: Social Listening                    ─── requires Phase 7; can parallel with Phase 9
    │
Phase 9: Approval Workflows                 ─── requires Phase 3; can parallel with Phases 7, 8
    │
Phase 10: Trend Alerts & Strategy AI         ─── requires Phases 6, 7, 8
    │
Phase 11: Additional Platform Adapters       ─── requires Phase 2; can run any time after Phase 2
    │
Phase 12: MCP Server, Attribution & Self-Hosting ─── requires Phases 3, 4, 5; can parallel with Phases 10, 11
```

### Parallelism Opportunities

- **Phases 3, 4 can be developed concurrently** after Phase 2 (post scheduling and inbox are independent)
- **Phases 4, 5, 6 can be developed concurrently** after Phase 3 (inbox, analytics, and AI content are independent)
- **Phases 8, 9 can be developed concurrently** after their respective dependencies are met
- **Phase 11 (additional adapters) can run at any time** after Phase 2, as each adapter is independent
- **Phase 12 can parallel with Phases 10, 11** since MCP server, attribution, and self-hosting are independent features

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented with production-quality code.
2. All unit tests pass (`pnpm test:unit`).
3. All integration tests pass (`pnpm test:integration`).
4. ESLint passes with zero errors (`pnpm lint`).
5. TypeScript compiles with strict mode and zero errors (`pnpm typecheck`).
6. Docker build completes successfully (`docker build .`).
7. All new database schema changes have corresponding Drizzle migrations.
8. All new tRPC routes have Zod input validation.
9. All new API endpoints are accessible through the tRPC client with full type safety.
10. All new JSONB columns have documented JSON Schema structures in code comments.
11. All new background workers are registered in the worker bootstrap and have rate limiting configured.
12. Feature works end-to-end: a developer can demonstrate the new capability in the running application.
13. Sensitive data (OAuth tokens, API keys) is encrypted at rest and never logged.
14. Audit log entries are created for all security-relevant actions (account connections, approvals, role changes).
15. No regressions: all tests from previous phases continue to pass.
