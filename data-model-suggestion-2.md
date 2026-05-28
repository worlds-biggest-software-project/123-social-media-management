# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Social Media Management · Created: 2026-05-19

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The event store is the single source of truth; all read-optimised views (posts, analytics, inbox) are materialised projections rebuilt from the event stream. This is the CQRS (Command Query Responsibility Segregation) pattern, where writes go to the event store and reads come from purpose-built materialised views.

This approach is natural for social media management because the domain is inherently event-driven: posts are scheduled, approved, published, edited; messages arrive, get routed, get replied to; sentiment shifts over time. An event-sourced model captures the full history of every entity, enabling temporal queries ("what was the approval status at 3pm?"), complete audit trails without a separate log, and AI analytics on change patterns.

Event sourcing is used at scale by financial systems, healthcare record systems, and platforms like LinkedIn (Apache Kafka + derived views). The trade-off is increased complexity in the write path and the need to maintain materialised views, but the payoff is unmatched auditability and temporal query power.

**Best for:** Teams that need complete audit trails, temporal queries, AI-powered analytics on historical patterns, and the ability to replay events to rebuild state or backfill new features.

**Trade-offs:**
- Pro: Complete, immutable audit trail is built into the architecture, not bolted on
- Pro: Temporal queries are first-class ("show me the inbox state as of last Tuesday")
- Pro: New read models can be built retroactively by replaying events
- Pro: Natural fit for AI analytics — event streams are ideal training data
- Pro: Supports event-driven integrations (webhooks, real-time notifications) natively
- Con: Higher write complexity; every action is an event + projection update
- Con: Materialised views must be maintained and can drift if projections have bugs
- Con: Eventual consistency between event store and read models
- Con: Steeper learning curve for developers unfamiliar with CQRS
- Con: Snapshot/compaction strategies needed for entities with long event histories

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Every event in the store follows the CloudEvents envelope (specversion, type, source, subject, time, data) |
| ISO 8601 | All event timestamps and scheduling times use ISO 8601 TIMESTAMPTZ |
| ISO 639-1 | Language codes in content and sentiment events |
| ISO 3166-1 | Country codes in geo-targeting and audience events |
| ISO 4217 | Currency codes in attribution and revenue events |
| OAuth 2.0 (RFC 6749) | Token lifecycle events (granted, refreshed, revoked) tracked in event store |
| AsyncAPI 3.0 | Event channels documented using AsyncAPI spec for consumer contracts |
| Schema.org SocialMediaPosting | Content type vocabulary in post events |
| OpenTelemetry | Correlation IDs in events enable distributed tracing across projections |

---

## Event Store (Write Side)

```sql
-- Central event store — the single source of truth
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_id       UUID NOT NULL,          -- aggregate root ID (post, inbox_thread, account, etc.)
    stream_type     VARCHAR(100) NOT NULL,   -- Post, InboxThread, SocialAccount, ListeningQuery
    sequence_num    BIGINT NOT NULL,          -- per-stream ordering
    -- CloudEvents envelope fields
    specversion     VARCHAR(10) NOT NULL DEFAULT '1.0',
    event_type      VARCHAR(200) NOT NULL,   -- com.smm.post.created, com.smm.post.approved, com.smm.inbox.message_received
    source          VARCHAR(500) NOT NULL,   -- /tenants/{id}/posts/{id}
    subject         VARCHAR(500),
    correlation_id  UUID,                     -- links related events across aggregates
    causation_id    UUID,                     -- the event that caused this event
    actor_id        UUID,                     -- user who triggered this event
    actor_ip        INET,
    data            JSONB NOT NULL,           -- event payload (varies by event_type)
    metadata        JSONB DEFAULT '{}',       -- trace context, client info
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
);

-- Partitioned by month for performance
-- In production: CREATE TABLE events (...) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_events_stream ON events (stream_id, sequence_num);
CREATE INDEX idx_events_tenant_type ON events (tenant_id, event_type, occurred_at DESC);
CREATE INDEX idx_events_correlation ON events (correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_occurred ON events (occurred_at DESC);

-- Snapshots for long-lived aggregates (avoids replaying thousands of events)
CREATE TABLE event_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    sequence_num    BIGINT NOT NULL,       -- snapshot is valid up to this sequence
    state           JSONB NOT NULL,         -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_num)
);
```

### Event Type Catalogue

```
-- Post lifecycle events
com.smm.post.created           -- {content_type, body_text, created_by}
com.smm.post.updated           -- {changes: {field: {old, new}}}
com.smm.post.scheduled         -- {scheduled_at, timezone, target_accounts: [...]}
com.smm.post.approval_requested -- {workflow_id, requested_by}
com.smm.post.approved          -- {approved_by, step_index}
com.smm.post.rejected          -- {rejected_by, reason, step_index}
com.smm.post.published         -- {platform, platform_post_id, platform_url}
com.smm.post.publish_failed    -- {platform, error_code, error_message}
com.smm.post.archived          -- {archived_by}
com.smm.post.media_attached    -- {media_asset_id, sort_order}
com.smm.post.ab_test_started   -- {variant_a_id, variant_b_id}

-- Inbox events
com.smm.inbox.message_received  -- {platform, author, body_text, sentiment, message_type}
com.smm.inbox.message_replied   -- {replied_by, reply_text}
com.smm.inbox.message_assigned  -- {assigned_to, assigned_by}
com.smm.inbox.message_resolved  -- {resolved_by}
com.smm.inbox.sentiment_updated -- {old_sentiment, new_sentiment, model_version}

-- Social account events
com.smm.account.connected      -- {platform, account_name, scopes}
com.smm.account.token_refreshed -- {new_expiry}
com.smm.account.token_revoked  -- {reason}
com.smm.account.disconnected   -- {disconnected_by}

-- Analytics events
com.smm.analytics.snapshot_collected -- {post_target_id, metrics: {...}}
com.smm.analytics.account_snapshot   -- {social_account_id, metrics: {...}}
com.smm.attribution.event_tracked    -- {event_type, utm_params, revenue}

-- Listening events
com.smm.listening.match_found   -- {query_id, platform, author, body_text, sentiment}
com.smm.listening.trend_detected -- {alert_type, title, severity}

-- AI events
com.smm.ai.content_requested   -- {request_type, prompt, model}
com.smm.ai.content_generated   -- {output_text, variants, tokens_used}
com.smm.ai.prediction_created  -- {post_id, predicted_metrics, confidence}
```

---

## Read Models (Query Side — Materialised Projections)

```sql
-- ============================================================
-- READ MODEL: Posts (materialised from post.* events)
-- ============================================================
CREATE TABLE rm_posts (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    content_type    VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    title           VARCHAR(500),
    body_text       TEXT,
    scheduled_at    TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    timezone        VARCHAR(50),
    created_by      UUID,
    approved_by     UUID,
    is_ab_test      BOOLEAN NOT NULL DEFAULT false,
    ab_parent_id    UUID,
    event_sequence  BIGINT NOT NULL,  -- last applied event sequence
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_posts_tenant_status ON rm_posts (tenant_id, status);
CREATE INDEX idx_rm_posts_scheduled ON rm_posts (scheduled_at) WHERE status = 'scheduled';

-- ============================================================
-- READ MODEL: Post Targets
-- ============================================================
CREATE TABLE rm_post_targets (
    id              UUID PRIMARY KEY,
    post_id         UUID NOT NULL,
    social_account_id UUID NOT NULL,
    platform_code   VARCHAR(50) NOT NULL,
    platform_post_id VARCHAR(512),
    platform_url    TEXT,
    status          VARCHAR(30) NOT NULL,
    error_message   TEXT,
    published_at    TIMESTAMPTZ,
    event_sequence  BIGINT NOT NULL
);
CREATE INDEX idx_rm_post_targets_post ON rm_post_targets (post_id);

-- ============================================================
-- READ MODEL: Inbox (materialised from inbox.* events)
-- ============================================================
CREATE TABLE rm_inbox (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    social_account_id UUID NOT NULL,
    platform_code   VARCHAR(50) NOT NULL,
    message_type    VARCHAR(30) NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    author_name     VARCHAR(255),
    author_handle   VARCHAR(255),
    body_text       TEXT,
    parent_id       UUID,
    sentiment       VARCHAR(20),
    sentiment_score NUMERIC(5,4),
    language_code   VARCHAR(10),
    is_read         BOOLEAN NOT NULL DEFAULT false,
    assigned_to     UUID,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    received_at     TIMESTAMPTZ NOT NULL,
    event_sequence  BIGINT NOT NULL
);
CREATE INDEX idx_rm_inbox_tenant_status ON rm_inbox (tenant_id, status, received_at DESC);
CREATE INDEX idx_rm_inbox_sentiment ON rm_inbox (tenant_id, sentiment);

-- ============================================================
-- READ MODEL: Analytics (time-series optimised)
-- ============================================================
CREATE TABLE rm_post_analytics (
    post_target_id  UUID NOT NULL,
    snapshot_at     TIMESTAMPTZ NOT NULL,
    impressions     BIGINT DEFAULT 0,
    reach           BIGINT DEFAULT 0,
    engagements     BIGINT DEFAULT 0,
    likes           BIGINT DEFAULT 0,
    comments        BIGINT DEFAULT 0,
    shares          BIGINT DEFAULT 0,
    clicks          BIGINT DEFAULT 0,
    video_views     BIGINT DEFAULT 0,
    PRIMARY KEY (post_target_id, snapshot_at)
);

-- ============================================================
-- READ MODEL: Listening matches
-- ============================================================
CREATE TABLE rm_listening_matches (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    query_id        UUID NOT NULL,
    platform_code   VARCHAR(50) NOT NULL,
    author_handle   VARCHAR(255),
    body_text       TEXT,
    sentiment       VARCHAR(20),
    sentiment_score NUMERIC(5,4),
    reach_estimate  INTEGER,
    matched_at      TIMESTAMPTZ NOT NULL,
    event_sequence  BIGINT NOT NULL
);
CREATE INDEX idx_rm_listening_tenant ON rm_listening_matches (tenant_id, matched_at DESC);

-- ============================================================
-- READ MODEL: Social Accounts (current state)
-- ============================================================
CREATE TABLE rm_social_accounts (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    platform_code   VARCHAR(50) NOT NULL,
    account_name    VARCHAR(255),
    account_handle  VARCHAR(255),
    platform_uid    VARCHAR(512),
    account_type    VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    follower_count  INTEGER,
    token_status    VARCHAR(30) NOT NULL DEFAULT 'active',  -- active, expired, revoked
    token_expires_at TIMESTAMPTZ,
    connected_at    TIMESTAMPTZ,
    event_sequence  BIGINT NOT NULL
);
CREATE INDEX idx_rm_accounts_tenant ON rm_social_accounts (tenant_id);

-- ============================================================
-- READ MODEL: Trend Alerts
-- ============================================================
CREATE TABLE rm_trend_alerts (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    alert_type      VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    severity        VARCHAR(20) NOT NULL,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_alerts_tenant ON rm_trend_alerts (tenant_id, created_at DESC);
```

---

## Projection Infrastructure

```sql
-- Tracks which events each projection has consumed
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,  -- 'rm_posts', 'rm_inbox', etc.
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed projection
CREATE TABLE projection_dead_letters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Reference Data (Shared)

```sql
-- Platforms (same as normalised model — reference data is not event-sourced)
CREATE TABLE platforms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    display_name    VARCHAR(100) NOT NULL,
    capabilities    TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- Tenants (not event-sourced; low-change reference data)
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users (not event-sourced; managed by auth system)
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

---

## Example Queries

### Rebuild post state from events

```sql
-- Get all events for a specific post to replay its state
SELECT event_type, data, occurred_at
FROM events
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'Post'
ORDER BY sequence_num ASC;
```

### Temporal query: "What was the approval status at a specific time?"

```sql
-- Find the last approval-related event before a given timestamp
SELECT event_type, data, occurred_at
FROM events
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND event_type LIKE 'com.smm.post.approv%'
  AND occurred_at <= '2026-05-15 15:00:00+00'
ORDER BY sequence_num DESC
LIMIT 1;
```

### Analytics: sentiment trend over time from events

```sql
-- Aggregate sentiment from inbox events over the last 30 days
SELECT
    date_trunc('day', occurred_at) AS day,
    data->>'sentiment' AS sentiment,
    COUNT(*) AS message_count
FROM events
WHERE tenant_id = '...'
  AND event_type = 'com.smm.inbox.message_received'
  AND occurred_at >= now() - INTERVAL '30 days'
GROUP BY day, sentiment
ORDER BY day, sentiment;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, event_snapshots |
| Projection Infrastructure | 2 | projection_checkpoints, projection_dead_letters |
| Read Model: Posts | 2 | rm_posts, rm_post_targets |
| Read Model: Inbox | 1 | rm_inbox |
| Read Model: Analytics | 1 | rm_post_analytics |
| Read Model: Listening | 1 | rm_listening_matches |
| Read Model: Accounts | 1 | rm_social_accounts |
| Read Model: Alerts | 1 | rm_trend_alerts |
| Reference Data | 3 | tenants, users, platforms |
| **Total** | **14** | Plus potentially more read models as needs grow |

---

## Key Design Decisions

1. **Events table as single source of truth** — All state changes are immutable events. The `events` table is append-only; no UPDATE or DELETE operations are permitted on it. This guarantees a complete audit trail without any additional logging infrastructure.

2. **CloudEvents envelope for every event** — Every event includes `specversion`, `type`, `source`, and `subject` fields from the CloudEvents spec. This means events can be published to external systems (Kafka, webhooks, cloud event buses) without transformation.

3. **Stream-based partitioning** — Events are grouped by `stream_id` (the aggregate root, like a specific post or inbox thread) with per-stream sequence numbers. This enables optimistic concurrency control: before writing a new event, check that `sequence_num` matches your expected value.

4. **Snapshots for long-lived aggregates** — Posts with hundreds of edits or inbox threads with thousands of messages would be expensive to replay from the beginning. The `event_snapshots` table stores periodic state snapshots, so replay starts from the last snapshot.

5. **Read models are disposable** — Every `rm_*` table can be dropped and rebuilt from the event store. This means new features (e.g., a new analytics dashboard) can be added by creating a new projection that processes historical events, without any data migration.

6. **Reference data is not event-sourced** — Tenants, users, and platforms change infrequently and don't benefit from event sourcing. They use simple relational tables to avoid unnecessary complexity.

7. **Correlation and causation IDs** — Every event can reference the event that caused it (`causation_id`) and a shared `correlation_id` for tracing a business process across aggregates (e.g., a post approval triggers a publish, which triggers analytics collection).

8. **Monthly partitioning strategy** — The events table should be partitioned by `occurred_at` month in production, enabling efficient time-range queries and old-partition archival (move to cold storage after N months).

9. **Projection dead letter queue** — When a projection fails to process an event (bug, schema mismatch), the event goes to `projection_dead_letters` rather than blocking the entire projection pipeline. This enables graceful degradation.
