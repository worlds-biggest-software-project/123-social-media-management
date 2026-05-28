# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Social Media Management · Created: 2026-05-19

## Philosophy

This model follows classical relational database design with full normalization (3NF+). Every domain concept gets its own table with explicit foreign keys, constraints, and indexes. The design prioritises data integrity, referential consistency, and clear schema documentation over flexibility.

This is the approach used by mature SaaS platforms like Sprout Social and Hootsuite, where the schema is well-understood and evolves through managed migrations. It works best when the development team values strong typing, wants SQL-native reporting, and prefers explicit schema changes over schemaless flexibility.

The model aligns closely with the Meta Graph API's node-edge-field mental model, where every object (post, comment, reaction, user) is a first-class entity with typed relationships. Reference data tables (platforms, content types, sentiment labels) are separate lookup tables rather than enums or magic strings.

**Best for:** Teams that want a fully documented, strongly typed schema with maximum query flexibility and SQL-native analytics.

**Trade-offs:**
- Pro: Maximum referential integrity; every relationship is enforced by the database
- Pro: Standard SQL reporting works out of the box; no JSONB parsing needed
- Pro: Easy to reason about; new developers can understand the schema from an ER diagram
- Pro: Migration tooling (Flyway, Alembic, Prisma Migrate) works perfectly
- Con: High table count (~45-55 tables) increases JOIN complexity
- Con: Platform-specific fields require ALTER TABLE or new tables for each platform
- Con: Schema changes require migrations; less agile for rapid prototyping
- Con: Multi-platform post metadata varies widely; normalizing it fully creates many sparse tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 8601 | All timestamps stored as TIMESTAMPTZ; scheduling uses ISO 8601 intervals |
| ISO 639-1 | `language_code` columns for post locale and sentiment analysis language |
| ISO 3166-1 alpha-2 | `country_code` for geo-targeting and audience segmentation |
| ISO 4217 | `currency_code` for ad spend and revenue attribution |
| OAuth 2.0 (RFC 6749) | `social_accounts` table stores token lifecycle per RFC 6749/6750 |
| CloudEvents 1.0 | `audit_log` table fields align with CloudEvents context attributes |
| Schema.org SocialMediaPosting | Post content types map to Schema.org vocabulary |
| W3C Activity Streams 2.0 | ActivityPub content types (Note, Article, Video) mapped via `content_types` lookup |
| OpenTelemetry | `platform_api_logs` table captures trace context for observability |

---

## Core Identity & Multi-Tenancy

```sql
-- Tenants / Organisations
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, essentials, team, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tenants_slug ON tenants (slug);

-- Users (operators of the platform)
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'email',  -- email, google, github
    auth_subject    VARCHAR(512),  -- external IdP subject identifier
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
CREATE INDEX idx_users_tenant ON users (tenant_id);
CREATE INDEX idx_users_email ON users (email);

-- Roles
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,  -- admin, editor, viewer, approver
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

-- Permissions
CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,  -- posts.create, posts.publish, analytics.view
    description     TEXT,
    category        VARCHAR(50) NOT NULL  -- posts, analytics, inbox, settings, team
);

-- Role-Permission mapping
CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- User-Role mapping
CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    assigned_by     UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);
```

### Row-Level Security

```sql
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

---

## Social Account Management

```sql
-- Supported platforms (reference data)
CREATE TABLE platforms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,  -- facebook, instagram, x, linkedin, tiktok, youtube, mastodon, bluesky
    display_name    VARCHAR(100) NOT NULL,
    icon_url        TEXT,
    api_version     VARCHAR(20),
    capabilities    TEXT[] NOT NULL DEFAULT '{}',  -- {publish, schedule, analytics, inbox, listening}
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- Connected social accounts
CREATE TABLE social_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    platform_id     UUID NOT NULL REFERENCES platforms(id),
    account_name    VARCHAR(255) NOT NULL,
    account_handle  VARCHAR(255),
    platform_uid    VARCHAR(512) NOT NULL,  -- platform-specific user/page ID
    account_type    VARCHAR(50) NOT NULL,   -- page, profile, business, creator
    access_token    TEXT NOT NULL,           -- encrypted at rest
    refresh_token   TEXT,                    -- encrypted at rest
    token_expires_at TIMESTAMPTZ,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    avatar_url      TEXT,
    follower_count  INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    connected_by    UUID NOT NULL REFERENCES users(id),
    connected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (platform_id, platform_uid)
);
CREATE INDEX idx_social_accounts_tenant ON social_accounts (tenant_id);
CREATE INDEX idx_social_accounts_platform ON social_accounts (platform_id);
```

---

## Content Publishing & Scheduling

```sql
-- Content types (reference data aligned with Schema.org)
CREATE TABLE content_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,  -- text_post, image_post, video_post, story, reel, carousel, poll, thread
    schema_org_type VARCHAR(100),                 -- SocialMediaPosting, VideoObject, ImageObject
    as2_type        VARCHAR(100),                 -- Note, Article, Video (Activity Streams 2.0)
    display_name    VARCHAR(100) NOT NULL
);

-- Posts (the core scheduling/publishing entity)
CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    content_type_id UUID NOT NULL REFERENCES content_types(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',  -- draft, pending_approval, approved, scheduled, publishing, published, failed, archived
    title           VARCHAR(500),
    body_text       TEXT,
    body_html       TEXT,
    scheduled_at    TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_ab_test      BOOLEAN NOT NULL DEFAULT false,
    ab_test_group   VARCHAR(10),  -- A, B
    ab_parent_id    UUID REFERENCES posts(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_posts_tenant_status ON posts (tenant_id, status);
CREATE INDEX idx_posts_scheduled ON posts (scheduled_at) WHERE status = 'scheduled';
CREATE INDEX idx_posts_published ON posts (published_at DESC) WHERE status = 'published';

-- Post-to-social-account targeting (which accounts a post publishes to)
CREATE TABLE post_targets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    social_account_id UUID NOT NULL REFERENCES social_accounts(id),
    platform_post_id VARCHAR(512),  -- ID returned by platform after publishing
    platform_url    TEXT,           -- permalink on the platform
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',  -- pending, published, failed
    error_message   TEXT,
    published_at    TIMESTAMPTZ,
    UNIQUE (post_id, social_account_id)
);
CREATE INDEX idx_post_targets_post ON post_targets (post_id);
CREATE INDEX idx_post_targets_account ON post_targets (social_account_id);

-- Media attachments
CREATE TABLE media_assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    file_name       VARCHAR(500) NOT NULL,
    file_type       VARCHAR(100) NOT NULL,   -- image/jpeg, video/mp4, image/png
    file_size_bytes BIGINT NOT NULL,
    storage_url     TEXT NOT NULL,
    thumbnail_url   TEXT,
    width           INTEGER,
    height          INTEGER,
    duration_ms     INTEGER,  -- for video/audio
    alt_text        TEXT,     -- WCAG 2.2 accessibility
    caption         TEXT,     -- WebVTT for video captions
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_media_tenant ON media_assets (tenant_id);

-- Post-Media junction
CREATE TABLE post_media (
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    media_asset_id  UUID NOT NULL REFERENCES media_assets(id),
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    PRIMARY KEY (post_id, media_asset_id)
);

-- Hashtags
CREATE TABLE hashtags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tag             VARCHAR(255) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hashtags_tag ON hashtags (tag);

-- Post-Hashtag junction
CREATE TABLE post_hashtags (
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    hashtag_id      UUID NOT NULL REFERENCES hashtags(id),
    PRIMARY KEY (post_id, hashtag_id)
);

-- Labels / Tags for internal organisation
CREATE TABLE labels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    colour          VARCHAR(7),  -- hex colour
    UNIQUE (tenant_id, name)
);

CREATE TABLE post_labels (
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    label_id        UUID NOT NULL REFERENCES labels(id),
    PRIMARY KEY (post_id, label_id)
);
```

---

## Engagement & Unified Inbox

```sql
-- Inbox messages (comments, DMs, mentions from all platforms)
CREATE TABLE inbox_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    social_account_id UUID NOT NULL REFERENCES social_accounts(id),
    platform_message_id VARCHAR(512) NOT NULL,
    message_type    VARCHAR(30) NOT NULL,  -- comment, direct_message, mention, reply, review
    direction       VARCHAR(10) NOT NULL,  -- inbound, outbound
    author_name     VARCHAR(255),
    author_handle   VARCHAR(255),
    author_avatar   TEXT,
    author_platform_uid VARCHAR(512),
    body_text       TEXT,
    parent_message_id UUID REFERENCES inbox_messages(id),  -- for threading
    post_target_id  UUID REFERENCES post_targets(id),      -- links to our published post
    sentiment       VARCHAR(20),    -- positive, negative, neutral, mixed
    sentiment_score NUMERIC(5,4),   -- -1.0000 to 1.0000
    language_code   VARCHAR(10),    -- ISO 639-1
    is_read         BOOLEAN NOT NULL DEFAULT false,
    assigned_to     UUID REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'open',  -- open, replied, resolved, archived
    received_at     TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inbox_tenant_status ON inbox_messages (tenant_id, status);
CREATE INDEX idx_inbox_sentiment ON inbox_messages (tenant_id, sentiment);
CREATE INDEX idx_inbox_received ON inbox_messages (received_at DESC);
CREATE INDEX idx_inbox_assigned ON inbox_messages (assigned_to) WHERE assigned_to IS NOT NULL;

-- Inbox routing rules (automated sentiment-based routing)
CREATE TABLE inbox_routing_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    priority        SMALLINT NOT NULL DEFAULT 0,
    conditions      JSONB NOT NULL,  -- {"sentiment": "negative", "platform": "x"}
    action_type     VARCHAR(50) NOT NULL,  -- assign_to, label, escalate, auto_reply
    action_config   JSONB NOT NULL,  -- {"user_id": "...", "template_id": "..."}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Social Listening

```sql
-- Listening queries (brand mentions, competitor tracking)
CREATE TABLE listening_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    query_type      VARCHAR(50) NOT NULL,  -- brand_mention, competitor, keyword, hashtag
    keywords        TEXT[] NOT NULL,
    excluded_keywords TEXT[] DEFAULT '{}',
    platforms       UUID[] DEFAULT '{}',  -- platform IDs to monitor
    language_filter VARCHAR(10)[],        -- ISO 639-1 codes
    country_filter  VARCHAR(5)[],         -- ISO 3166-1 codes
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Listening matches (mentions found by queries)
CREATE TABLE listening_matches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listening_query_id UUID NOT NULL REFERENCES listening_queries(id) ON DELETE CASCADE,
    platform_id     UUID NOT NULL REFERENCES platforms(id),
    platform_content_id VARCHAR(512),
    author_name     VARCHAR(255),
    author_handle   VARCHAR(255),
    body_text       TEXT,
    content_url     TEXT,
    sentiment       VARCHAR(20),
    sentiment_score NUMERIC(5,4),
    language_code   VARCHAR(10),
    country_code    VARCHAR(5),
    reach_estimate  INTEGER,
    matched_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_listening_matches_query ON listening_matches (listening_query_id, matched_at DESC);
CREATE INDEX idx_listening_matches_sentiment ON listening_matches (sentiment, matched_at DESC);
```

---

## Analytics & Attribution

```sql
-- Post-level analytics snapshots (collected periodically from platform APIs)
CREATE TABLE post_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_target_id  UUID NOT NULL REFERENCES post_targets(id) ON DELETE CASCADE,
    snapshot_at     TIMESTAMPTZ NOT NULL,
    impressions     BIGINT DEFAULT 0,
    reach           BIGINT DEFAULT 0,
    engagements     BIGINT DEFAULT 0,
    likes           BIGINT DEFAULT 0,
    comments        BIGINT DEFAULT 0,
    shares          BIGINT DEFAULT 0,
    saves           BIGINT DEFAULT 0,
    clicks          BIGINT DEFAULT 0,
    video_views     BIGINT DEFAULT 0,
    video_watch_time_ms BIGINT DEFAULT 0,
    follower_delta  INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_post_analytics_target ON post_analytics (post_target_id, snapshot_at DESC);

-- Account-level analytics snapshots
CREATE TABLE account_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    social_account_id UUID NOT NULL REFERENCES social_accounts(id) ON DELETE CASCADE,
    snapshot_date   DATE NOT NULL,
    followers       BIGINT DEFAULT 0,
    following       BIGINT DEFAULT 0,
    total_posts     BIGINT DEFAULT 0,
    impressions     BIGINT DEFAULT 0,
    reach           BIGINT DEFAULT 0,
    engagements     BIGINT DEFAULT 0,
    profile_views   BIGINT DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (social_account_id, snapshot_date)
);

-- Attribution tracking (post to website visit/conversion)
CREATE TABLE attribution_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    post_target_id  UUID REFERENCES post_targets(id),
    event_type      VARCHAR(50) NOT NULL,  -- page_view, signup, purchase, add_to_cart
    source_url      TEXT,
    landing_url     TEXT,
    utm_source      VARCHAR(255),
    utm_medium      VARCHAR(255),
    utm_campaign    VARCHAR(255),
    utm_content     VARCHAR(255),
    revenue_amount  NUMERIC(12,2),
    currency_code   VARCHAR(3),  -- ISO 4217
    session_id      VARCHAR(255),
    occurred_at     TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_attribution_tenant ON attribution_events (tenant_id, occurred_at DESC);
CREATE INDEX idx_attribution_post ON attribution_events (post_target_id) WHERE post_target_id IS NOT NULL;
```

---

## AI Content & Strategy

```sql
-- AI content generation requests
CREATE TABLE ai_content_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    requested_by    UUID NOT NULL REFERENCES users(id),
    request_type    VARCHAR(50) NOT NULL,  -- generate, rewrite, tone_adjust, suggest_strategy
    prompt          TEXT NOT NULL,
    model_provider  VARCHAR(50),  -- claude, openai, custom
    model_id        VARCHAR(100),
    input_post_id   UUID REFERENCES posts(id),
    output_text     TEXT,
    output_variants JSONB,  -- array of alternative suggestions
    tone            VARCHAR(50),  -- formal, casual, witty, professional
    language_code   VARCHAR(10),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',  -- pending, processing, completed, failed
    tokens_used     INTEGER,
    latency_ms      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_ai_requests_tenant ON ai_content_requests (tenant_id, created_at DESC);

-- Content performance predictions
CREATE TABLE performance_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    predicted_impressions  BIGINT,
    predicted_engagements  BIGINT,
    predicted_clicks       BIGINT,
    confidence_score       NUMERIC(5,4),
    model_version          VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Trend alerts
CREATE TABLE trend_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    alert_type      VARCHAR(50) NOT NULL,  -- trending_topic, competitor_spike, sentiment_shift, viral_content
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    severity        VARCHAR(20) NOT NULL DEFAULT 'info',  -- info, warning, critical
    related_query_id UUID REFERENCES listening_queries(id),
    data_snapshot   JSONB,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_trend_alerts_tenant ON trend_alerts (tenant_id, created_at DESC);
```

---

## Approval Workflows

```sql
-- Approval workflows
CREATE TABLE approval_workflows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    steps           JSONB NOT NULL,  -- ordered list of approver role/user requirements
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Approval requests
CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(id),
    current_step    SMALLINT NOT NULL DEFAULT 0,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',  -- pending, approved, rejected, cancelled
    requested_by    UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

-- Approval decisions
CREATE TABLE approval_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      UUID NOT NULL REFERENCES approval_requests(id) ON DELETE CASCADE,
    step_index      SMALLINT NOT NULL,
    decided_by      UUID NOT NULL REFERENCES users(id),
    decision        VARCHAR(20) NOT NULL,  -- approved, rejected, returned
    comment         TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Audit Log

```sql
-- Audit log (CloudEvents-aligned)
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    specversion     VARCHAR(10) NOT NULL DEFAULT '1.0',     -- CloudEvents spec version
    type            VARCHAR(200) NOT NULL,                   -- e.g. com.smm.post.published
    source          VARCHAR(500) NOT NULL,                   -- e.g. /tenants/{id}/posts/{id}
    subject         VARCHAR(500),
    actor_id        UUID REFERENCES users(id),
    actor_ip        INET,
    data            JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant_time ON audit_log (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_type ON audit_log (type);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 6 | tenants, users, roles, permissions, role_permissions, user_roles |
| Social Accounts | 2 | platforms, social_accounts |
| Content & Publishing | 8 | posts, post_targets, media_assets, post_media, hashtags, post_hashtags, labels, post_labels |
| Engagement & Inbox | 2 | inbox_messages, inbox_routing_rules |
| Social Listening | 2 | listening_queries, listening_matches |
| Analytics & Attribution | 3 | post_analytics, account_analytics, attribution_events |
| AI & Strategy | 3 | ai_content_requests, performance_predictions, trend_alerts |
| Approval Workflows | 3 | approval_workflows, approval_requests, approval_decisions |
| Audit | 1 | audit_log |
| **Total** | **30** | |

---

## Key Design Decisions

1. **Shared-schema multi-tenancy with RLS** — All tables include `tenant_id` with PostgreSQL Row-Level Security policies. This supports thousands of tenants without database-per-tenant complexity while maintaining strict data isolation.

2. **Separate `post_targets` junction** — A single post can target multiple social accounts. The `post_targets` table tracks per-platform publishing status independently, which mirrors how Buffer and Postiz handle cross-posting.

3. **Platform as reference data** — The `platforms` table is a lookup table with capabilities array, making it easy to add new platforms (Bluesky, Threads) without schema changes.

4. **Sentiment stored inline** — Sentiment scores are stored directly on `inbox_messages` and `listening_matches` rather than in a separate analysis table, optimising for the common read path (inbox filtered by sentiment).

5. **Analytics as periodic snapshots** — Post and account analytics are captured as timestamped snapshots rather than real-time counters, reflecting the reality that platform APIs provide batch data, not streaming updates.

6. **CloudEvents-aligned audit log** — The `audit_log` table uses CloudEvents field naming (`specversion`, `type`, `source`, `subject`) for interoperability with external log aggregation systems.

7. **UUID primary keys throughout** — Enables distributed ID generation without coordination, important for a platform that ingests data from many external APIs simultaneously.

8. **Encrypted tokens at application layer** — OAuth tokens are stored as TEXT with a comment noting encryption at rest; the database stores ciphertext, and the application handles encryption/decryption.
