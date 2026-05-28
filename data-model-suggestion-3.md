# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Social Media Management · Created: 2026-05-19

## Philosophy

This model uses relational tables with strongly typed columns for core, universal fields, and PostgreSQL JSONB columns for platform-specific, variable, or rapidly evolving data. The key insight is that social media management spans many platforms, each with unique post formats, metrics, and metadata — trying to normalise everything into typed columns creates either a sparse schema or an unmanageable number of platform-specific tables.

The hybrid approach is used by platforms like Postiz and Buffer, where a post has common fields (body text, scheduled time, status) but also needs platform-specific data (Instagram carousel settings, TikTok stitch permissions, LinkedIn article thumbnails, Mastodon content warnings). Rather than ALTER TABLE for every new platform or feature, the variable data lives in JSONB columns with JSON Schema validation at the application layer.

This is the pragmatic "startup" model: fast to develop, easy to extend, and still queryable (PostgreSQL's JSONB operators and GIN indexes make JSON data searchable). It sacrifices some referential integrity for velocity and flexibility.

**Best for:** Teams building an MVP or multi-platform tool where platform-specific data varies widely and the schema needs to evolve rapidly without frequent migrations.

**Trade-offs:**
- Pro: Fast schema evolution; new platform-specific fields don't require migrations
- Pro: Lower table count (~20 tables) reduces JOIN complexity
- Pro: JSONB GIN indexes make variable data queryable at reasonable performance
- Pro: Natural fit for multi-platform variance (each platform's quirks live in JSONB)
- Pro: JSON Schema validation at app layer provides structure without rigidity
- Con: No foreign key constraints on JSONB contents; referential integrity is app-enforced
- Con: JSONB queries are slower than indexed column queries for high-cardinality filters
- Con: Schema documentation must be maintained outside the database (JSON Schema files)
- Con: Type safety is weaker; bugs in JSONB structure are caught at runtime, not migration time
- Con: Reporting on JSONB fields requires JSONB path extraction, which is less intuitive for analysts

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JSON Schema 2020-12 | Application-layer validation of all JSONB columns; schemas versioned alongside code |
| ISO 8601 | All timestamps as TIMESTAMPTZ; JSONB date fields stored as ISO 8601 strings |
| ISO 639-1 | Language codes in post metadata and sentiment JSONB |
| ISO 3166-1 | Country codes in audience and targeting JSONB |
| ISO 4217 | Currency codes in attribution JSONB |
| OAuth 2.0 (RFC 6749) | Token data in social_accounts.credentials JSONB |
| CloudEvents 1.0 | Audit entries follow CloudEvents structure in JSONB |
| Schema.org SocialMediaPosting | Content type mapping in post.platform_config |
| OpenAPI 3.1 / JSON Schema | API response schemas documented with OpenAPI, matching JSONB structures |

---

## Core Tables

```sql
-- Tenants
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    /*  settings example:
        {
            "default_timezone": "America/New_York",
            "ai_provider": "claude",
            "branding": {"primary_color": "#1DA1F2", "logo_url": "..."},
            "features_enabled": ["sentiment", "listening", "attribution"]
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    role            VARCHAR(50) NOT NULL DEFAULT 'editor',  -- admin, editor, approver, viewer
    permissions     JSONB NOT NULL DEFAULT '[]',
    /*  permissions example (overrides for fine-grained control):
        [
            {"resource": "posts", "actions": ["create", "edit", "publish"]},
            {"resource": "inbox", "actions": ["view", "reply"]},
            {"resource": "analytics", "actions": ["view", "export"]}
        ]
    */
    preferences     JSONB NOT NULL DEFAULT '{}',
    /*  preferences example:
        {
            "notifications": {"email": true, "push": false},
            "dashboard_layout": "compact",
            "timezone": "Europe/London"
        }
    */
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'email',
    auth_subject    VARCHAR(512),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
CREATE INDEX idx_users_tenant ON users (tenant_id);
```

---

## Social Accounts

```sql
CREATE TABLE social_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    platform        VARCHAR(50) NOT NULL,  -- facebook, instagram, x, linkedin, tiktok, youtube, mastodon, bluesky, threads, pinterest
    account_name    VARCHAR(255) NOT NULL,
    account_handle  VARCHAR(255),
    platform_uid    VARCHAR(512) NOT NULL,
    account_type    VARCHAR(50) NOT NULL,
    credentials     JSONB NOT NULL,
    /*  credentials example (encrypted at application layer):
        {
            "access_token": "enc:...",
            "refresh_token": "enc:...",
            "token_expires_at": "2026-06-19T00:00:00Z",
            "scopes": ["pages_manage_posts", "pages_read_engagement"],
            "api_version": "v22.0"
        }
    */
    profile         JSONB NOT NULL DEFAULT '{}',
    /*  profile example:
        {
            "avatar_url": "https://...",
            "follower_count": 15420,
            "bio": "Official brand account",
            "verified": true,
            "platform_specific": {
                "facebook": {"page_category": "Brand", "page_id": "123456"},
                "mastodon": {"instance_url": "https://mastodon.social", "bot": false}
            }
        }
    */
    capabilities    TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    connected_by    UUID NOT NULL REFERENCES users(id),
    connected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (platform, platform_uid)
);
CREATE INDEX idx_social_accounts_tenant ON social_accounts (tenant_id);
CREATE INDEX idx_social_accounts_platform ON social_accounts (platform);
```

---

## Posts & Content

```sql
CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    content_type    VARCHAR(50) NOT NULL,  -- text_post, image_post, video_post, carousel, story, reel, poll, thread
    -- Universal fields (every post has these)
    body_text       TEXT,
    scheduled_at    TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_by      UUID NOT NULL REFERENCES users(id),
    -- Platform-specific configuration per target
    platform_configs JSONB NOT NULL DEFAULT '{}',
    /*  platform_configs example:
        {
            "instagram": {
                "social_account_id": "uuid-here",
                "carousel_items": [
                    {"media_id": "uuid-1", "alt_text": "Product shot"},
                    {"media_id": "uuid-2", "alt_text": "Lifestyle photo"}
                ],
                "location_tag": "New York, NY",
                "product_tags": [{"product_id": "sku-123", "x": 0.5, "y": 0.3}],
                "first_comment": "#brand #lifestyle #fashion"
            },
            "x": {
                "social_account_id": "uuid-here",
                "reply_settings": "everyone",
                "quote_tweet_of": null,
                "poll": {
                    "options": ["Option A", "Option B", "Option C"],
                    "duration_minutes": 1440
                }
            },
            "linkedin": {
                "social_account_id": "uuid-here",
                "visibility": "PUBLIC",
                "article_thumbnail_url": "https://...",
                "commentary": "Check out our latest insights..."
            },
            "mastodon": {
                "social_account_id": "uuid-here",
                "content_warning": "Mild spoilers",
                "visibility": "public",
                "language": "en",
                "sensitive": false
            }
        }
    */
    -- Approval state
    approval_status VARCHAR(30),  -- null (no approval needed), pending, approved, rejected
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    -- A/B testing
    ab_test_group   VARCHAR(10),
    ab_parent_id    UUID REFERENCES posts(id),
    -- Metadata
    labels          TEXT[] DEFAULT '{}',  -- internal tags for organisation
    metadata        JSONB NOT NULL DEFAULT '{}',
    /*  metadata example:
        {
            "ai_generated": true,
            "ai_model": "claude-4-sonnet",
            "campaign_id": "summer-2026",
            "notes": "Part of the Q3 campaign",
            "utm_params": {"source": "linkedin", "medium": "social", "campaign": "summer2026"}
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_posts_tenant_status ON posts (tenant_id, status);
CREATE INDEX idx_posts_scheduled ON posts (scheduled_at) WHERE status = 'scheduled';
CREATE INDEX idx_posts_published ON posts (published_at DESC) WHERE status = 'published';
CREATE INDEX idx_posts_labels ON posts USING GIN (labels);
CREATE INDEX idx_posts_platform_configs ON posts USING GIN (platform_configs jsonb_path_ops);

-- Published post results (one per platform target)
CREATE TABLE post_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    social_account_id UUID NOT NULL REFERENCES social_accounts(id),
    platform        VARCHAR(50) NOT NULL,
    platform_post_id VARCHAR(512),
    platform_url    TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    error_details   JSONB,
    /*  error_details example:
        {
            "error_code": "RATE_LIMITED",
            "message": "Too many requests",
            "retry_at": "2026-05-19T15:30:00Z",
            "api_response": {"status": 429, "headers": {"x-rate-limit-reset": "..."}}
        }
    */
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_post_results_post ON post_results (post_id);

-- Media assets
CREATE TABLE media_assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    file_name       VARCHAR(500) NOT NULL,
    file_type       VARCHAR(100) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    storage_url     TEXT NOT NULL,
    thumbnail_url   TEXT,
    dimensions      JSONB,
    /*  dimensions example:
        {
            "width": 1920, "height": 1080,
            "duration_ms": 30000,
            "aspect_ratio": "16:9",
            "format": "mp4",
            "codec": "h264"
        }
    */
    alt_text        TEXT,
    caption_vtt     TEXT,  -- WebVTT for video captions
    iptc_metadata   JSONB DEFAULT '{}',  -- IPTC Photo Metadata fields
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
```

---

## Unified Inbox

```sql
CREATE TABLE inbox_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    social_account_id UUID NOT NULL REFERENCES social_accounts(id),
    platform        VARCHAR(50) NOT NULL,
    platform_message_id VARCHAR(512) NOT NULL,
    message_type    VARCHAR(30) NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    -- Author info (denormalised from platform)
    author          JSONB NOT NULL,
    /*  author example:
        {
            "name": "Jane Smith",
            "handle": "@janesmith",
            "avatar_url": "https://...",
            "platform_uid": "123456",
            "follower_count": 5200,
            "is_verified": true
        }
    */
    body_text       TEXT,
    parent_id       UUID REFERENCES inbox_messages(id),
    post_result_id  UUID REFERENCES post_results(id),
    -- Sentiment analysis
    sentiment       JSONB,
    /*  sentiment example:
        {
            "label": "negative",
            "score": -0.7823,
            "language": "en",
            "model_version": "sentiment-v3.1",
            "emotions": {"anger": 0.6, "frustration": 0.3, "sadness": 0.1},
            "topics": ["shipping", "delay", "customer_service"]
        }
    */
    -- State
    is_read         BOOLEAN NOT NULL DEFAULT false,
    assigned_to     UUID REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    routing_metadata JSONB DEFAULT '{}',
    /*  routing_metadata example:
        {
            "matched_rules": ["rule-uuid-1"],
            "auto_assigned": true,
            "escalated": false,
            "sla_deadline": "2026-05-19T18:00:00Z"
        }
    */
    received_at     TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inbox_tenant_status ON inbox_messages (tenant_id, status, received_at DESC);
CREATE INDEX idx_inbox_sentiment ON inbox_messages USING GIN (sentiment jsonb_path_ops);
CREATE INDEX idx_inbox_assigned ON inbox_messages (assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_inbox_platform ON inbox_messages (platform, received_at DESC);
```

---

## Social Listening

```sql
CREATE TABLE listening_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    /*  config example:
        {
            "query_type": "brand_mention",
            "keywords": ["mybrand", "my brand"],
            "excluded_keywords": ["unrelated"],
            "platforms": ["x", "mastodon", "bluesky"],
            "language_filter": ["en", "es"],
            "country_filter": ["US", "GB", "ES"],
            "sentiment_filter": null,
            "min_follower_count": 100
        }
    */
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE listening_matches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query_id        UUID NOT NULL REFERENCES listening_queries(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    platform        VARCHAR(50) NOT NULL,
    content         JSONB NOT NULL,
    /*  content example:
        {
            "platform_content_id": "1234567890",
            "author_name": "Tech Reviewer",
            "author_handle": "@techreviewer",
            "body_text": "Just tried @mybrand — impressive performance!",
            "content_url": "https://x.com/techreviewer/status/1234567890",
            "media_urls": [],
            "engagement": {"likes": 42, "replies": 5, "shares": 12},
            "reach_estimate": 15000
        }
    */
    sentiment       JSONB,  -- same structure as inbox_messages.sentiment
    matched_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_listening_matches_query ON listening_matches (query_id, matched_at DESC);
CREATE INDEX idx_listening_matches_sentiment ON listening_matches USING GIN (sentiment jsonb_path_ops);
```

---

## Analytics & Attribution

```sql
CREATE TABLE analytics_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    target_type     VARCHAR(30) NOT NULL,  -- post, account
    target_id       UUID NOT NULL,         -- post_result.id or social_account.id
    snapshot_at     TIMESTAMPTZ NOT NULL,
    metrics         JSONB NOT NULL,
    /*  metrics example (post):
        {
            "impressions": 12500, "reach": 8200,
            "engagements": 450, "likes": 320, "comments": 45,
            "shares": 85, "saves": 32, "clicks": 120,
            "video_views": 5000, "video_watch_time_ms": 180000,
            "follower_delta": 15,
            "platform_specific": {
                "instagram": {"profile_visits": 89, "sticker_taps": 12},
                "tiktok": {"full_video_watched_rate": 0.45}
            }
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_analytics_target ON analytics_snapshots (target_type, target_id, snapshot_at DESC);
CREATE INDEX idx_analytics_tenant ON analytics_snapshots (tenant_id, snapshot_at DESC);

CREATE TABLE attribution_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    post_result_id  UUID REFERENCES post_results(id),
    event_type      VARCHAR(50) NOT NULL,
    event_data      JSONB NOT NULL,
    /*  event_data example:
        {
            "source_url": "https://x.com/mybrand/status/123",
            "landing_url": "https://mybrand.com/products/summer",
            "utm": {"source": "x", "medium": "social", "campaign": "summer2026"},
            "conversion": {"type": "purchase", "revenue": 49.99, "currency": "USD"},
            "session_id": "sess_abc123",
            "user_agent": "Mozilla/5.0..."
        }
    */
    occurred_at     TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_attribution_tenant ON attribution_events (tenant_id, occurred_at DESC);
CREATE INDEX idx_attribution_post ON attribution_events (post_result_id) WHERE post_result_id IS NOT NULL;
```

---

## AI Content & Alerts

```sql
CREATE TABLE ai_interactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    requested_by    UUID NOT NULL REFERENCES users(id),
    interaction_type VARCHAR(50) NOT NULL,
    request         JSONB NOT NULL,
    /*  request example:
        {
            "type": "generate",
            "prompt": "Write a LinkedIn post about our new feature",
            "tone": "professional",
            "language": "en",
            "model": "claude-4-sonnet",
            "context": {"brand_voice_id": "uuid", "input_post_id": "uuid"},
            "constraints": {"max_length": 3000, "include_hashtags": true}
        }
    */
    response        JSONB,
    /*  response example:
        {
            "output_text": "Excited to announce...",
            "variants": [
                {"text": "We're thrilled to share...", "tone": "enthusiastic"},
                {"text": "Introducing our latest...", "tone": "informative"}
            ],
            "tokens_used": 850,
            "latency_ms": 1200,
            "model_id": "claude-4-sonnet-20260501"
        }
    */
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_ai_interactions_tenant ON ai_interactions (tenant_id, created_at DESC);

CREATE TABLE trend_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL DEFAULT 'info',
    content         JSONB NOT NULL,
    /*  content example:
        {
            "title": "Competitor @rival launched new feature",
            "description": "Detected 340% increase in mentions of @rival...",
            "related_query_id": "uuid",
            "data_points": [
                {"date": "2026-05-18", "mentions": 45},
                {"date": "2026-05-19", "mentions": 198}
            ],
            "suggested_action": "Consider publishing a response post highlighting your advantages"
        }
    */
    is_read         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alerts_tenant ON trend_alerts (tenant_id, created_at DESC);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    event           JSONB NOT NULL,
    /*  event example (CloudEvents-aligned):
        {
            "specversion": "1.0",
            "type": "com.smm.post.published",
            "source": "/tenants/uuid/posts/uuid",
            "subject": "post-uuid",
            "time": "2026-05-19T14:30:00Z",
            "data": {
                "platforms": ["instagram", "x"],
                "body_preview": "Excited to announce..."
            },
            "actor": {"user_id": "uuid", "email": "editor@brand.com", "ip": "192.168.1.1"}
        }
    */
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant_time ON audit_log (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_event_type ON audit_log ((event->>'type'));
```

---

## Example JSONB Queries

### Find all posts targeting Instagram with carousel items

```sql
SELECT id, body_text, platform_configs->'instagram'->'carousel_items' AS carousel
FROM posts
WHERE tenant_id = '...'
  AND platform_configs ? 'instagram'
  AND jsonb_array_length(platform_configs->'instagram'->'carousel_items') > 1;
```

### Filter inbox by negative sentiment with anger emotion

```sql
SELECT id, body_text, author->>'handle' AS author_handle, sentiment->>'label' AS sentiment_label
FROM inbox_messages
WHERE tenant_id = '...'
  AND sentiment @> '{"label": "negative"}'
  AND (sentiment->'emotions'->>'anger')::NUMERIC > 0.5
ORDER BY received_at DESC;
```

### Aggregate platform-specific analytics

```sql
SELECT
    target_id,
    snapshot_at,
    (metrics->>'impressions')::BIGINT AS impressions,
    (metrics->>'engagements')::BIGINT AS engagements,
    metrics->'platform_specific'->'tiktok'->>'full_video_watched_rate' AS tiktok_watch_rate
FROM analytics_snapshots
WHERE tenant_id = '...'
  AND target_type = 'post'
  AND snapshot_at >= now() - INTERVAL '7 days'
ORDER BY snapshot_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenants, users (RBAC in JSONB) |
| Social Accounts | 1 | social_accounts (credentials + profile in JSONB) |
| Content & Publishing | 4 | posts, post_results, media_assets, post_media |
| Engagement & Inbox | 1 | inbox_messages (author, sentiment, routing in JSONB) |
| Social Listening | 2 | listening_queries, listening_matches |
| Analytics & Attribution | 2 | analytics_snapshots, attribution_events |
| AI & Alerts | 2 | ai_interactions, trend_alerts |
| Audit | 1 | audit_log |
| **Total** | **15** | |

---

## Key Design Decisions

1. **JSONB for platform-specific variance** — The `platform_configs` column on `posts` is the centrepiece of this design. Each platform (Instagram, X, LinkedIn, Mastodon, TikTok) has wildly different post options (carousel items, poll options, content warnings, article thumbnails). Rather than creating `post_instagram_config`, `post_x_config`, etc., all platform-specific data lives in a single JSONB column keyed by platform code.

2. **GIN indexes on JSONB columns** — Every JSONB column that will be filtered (platform_configs, sentiment, metrics) has a GIN index with `jsonb_path_ops` for efficient containment queries (`@>` operator).

3. **Roles as a simple column, permissions as JSONB** — Instead of a full RBAC table structure (roles, permissions, junction tables), the `users` table has a `role` VARCHAR column for the primary role and a `permissions` JSONB array for fine-grained overrides. This reduces the table count by 4 tables and is sufficient for teams under 100 users.

4. **Unified analytics_snapshots table** — Rather than separate `post_analytics` and `account_analytics` tables, a single `analytics_snapshots` table uses `target_type` and `target_id` polymorphism. Platform-specific metrics (TikTok watch rate, Instagram sticker taps) go in the `metrics.platform_specific` JSONB subtree.

5. **Rich sentiment JSONB** — Sentiment is stored as a JSONB object with label, score, language, model version, emotions breakdown, and detected topics. This is far richer than a single VARCHAR column and enables queries like "show me messages with anger > 0.5" without schema changes.

6. **Author denormalised as JSONB** — Inbox message authors are stored as a JSONB object rather than a separate `contacts` table. This reflects the reality that social media "contacts" are transient platform identities, not CRM contacts, and avoids a large, sparsely populated contacts table.

7. **JSON Schema validation at application layer** — Every JSONB column has a corresponding JSON Schema definition in the codebase (not shown). The application validates data against these schemas before INSERT/UPDATE. This provides structure without database-level rigidity.

8. **15 tables total** — Half the table count of the normalized model, with the same functional coverage. The trade-off is that JSONB contents are not database-enforced, but the practical impact is minimal for a team using TypeScript/Python with typed SDKs.
