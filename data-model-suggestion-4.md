# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Social Media Management · Created: 2026-05-19

## Philosophy

This model combines a relational core for operational CRUD with a property graph layer for relationship-heavy queries. Social media management is fundamentally about networks: social graphs, influence chains, brand-audience relationships, content-to-outcome attribution paths, and competitor landscapes. A graph layer makes these relationship queries natural and performant, while the relational core handles transactional operations (scheduling, publishing, inbox management).

The graph layer can be implemented either as a dedicated graph database (Neo4j, Amazon Neptune) queried alongside PostgreSQL, or as PostgreSQL tables (`graph_nodes` / `graph_edges`) with recursive CTE queries. This proposal uses the PostgreSQL-native approach for simplicity, with notes on where a dedicated graph database would add value.

Graph-relational patterns are used by LinkedIn (social graph + relational data), Neo4j-powered social listening platforms (Brandwatch), and recommendation engines. The key advantage for a social media management tool is enabling queries that are awkward in pure relational models: "find influencers in my audience who also follow my competitor," "trace how a viral post spread through retweets," or "identify conflicts of interest in brand ambassador networks."

**Best for:** Teams building advanced features like influencer identification, content virality tracking, audience network analysis, and cross-platform social graph unification.

**Trade-offs:**
- Pro: Relationship queries (shortest path, influence chains, community detection) are natural
- Pro: Social graph traversal is O(relationships) not O(rows) as with JOIN-heavy relational queries
- Pro: Cross-platform identity linking (same person on X, LinkedIn, Instagram) is native
- Pro: Content attribution chains (post -> share -> click -> purchase) are first-class paths
- Pro: Supports AI-powered features like influencer scoring and audience clustering natively
- Con: Two query paradigms (SQL for CRUD + graph queries for relationships) increase complexity
- Con: Graph consistency with relational data requires careful sync
- Con: Developers need graph query skills (Cypher or recursive CTEs)
- Con: PostgreSQL-native graph queries (recursive CTEs) are less performant than dedicated graph DBs for deep traversals
- Con: Schema is more complex to document and onboard new developers

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C Activity Streams 2.0 | Graph node types align with AS2 object types (Person, Note, Article, Video) |
| W3C ActivityPub | Federation relationships modelled as graph edges (Follow, Like, Announce) |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| ISO 639-1 | Language codes on content nodes |
| ISO 3166-1 | Country codes on person/audience nodes |
| Schema.org SocialMediaPosting | Content node types map to Schema.org vocabulary |
| OAuth 2.0 (RFC 6749) | Account credentials in relational tables (not graph) |
| CloudEvents 1.0 | Audit events as graph edges with CloudEvents metadata |
| OpenTelemetry | Trace context on graph edge metadata for distributed tracing |

---

## Relational Core (Operational CRUD)

```sql
-- ============================================================
-- TENANTS & USERS (relational — low-change operational data)
-- ============================================================
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'editor',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
CREATE INDEX idx_users_tenant ON users (tenant_id);

-- ============================================================
-- SOCIAL ACCOUNTS (relational — credentials must be transactional)
-- ============================================================
CREATE TABLE social_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    platform        VARCHAR(50) NOT NULL,
    account_name    VARCHAR(255) NOT NULL,
    account_handle  VARCHAR(255),
    platform_uid    VARCHAR(512) NOT NULL,
    account_type    VARCHAR(50) NOT NULL,
    access_token    TEXT NOT NULL,  -- encrypted
    refresh_token   TEXT,           -- encrypted
    token_expires_at TIMESTAMPTZ,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    connected_by    UUID NOT NULL REFERENCES users(id),
    connected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (platform, platform_uid)
);
CREATE INDEX idx_social_accounts_tenant ON social_accounts (tenant_id);

-- ============================================================
-- POSTS (relational — scheduling requires transactional guarantees)
-- ============================================================
CREATE TABLE posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    content_type    VARCHAR(50) NOT NULL,
    body_text       TEXT,
    scheduled_at    TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    platform_configs JSONB NOT NULL DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    labels          TEXT[] DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_posts_tenant_status ON posts (tenant_id, status);
CREATE INDEX idx_posts_scheduled ON posts (scheduled_at) WHERE status = 'scheduled';

-- Post targets (per-platform publish results)
CREATE TABLE post_targets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    social_account_id UUID NOT NULL REFERENCES social_accounts(id),
    platform        VARCHAR(50) NOT NULL,
    platform_post_id VARCHAR(512),
    platform_url    TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    error_message   TEXT,
    published_at    TIMESTAMPTZ,
    UNIQUE (post_id, social_account_id)
);

-- Media assets
CREATE TABLE media_assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    file_name       VARCHAR(500) NOT NULL,
    file_type       VARCHAR(100) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    storage_url     TEXT NOT NULL,
    thumbnail_url   TEXT,
    alt_text        TEXT,
    dimensions      JSONB,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE post_media (
    post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    media_asset_id  UUID NOT NULL REFERENCES media_assets(id),
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    PRIMARY KEY (post_id, media_asset_id)
);

-- ============================================================
-- INBOX (relational — message routing is transactional)
-- ============================================================
CREATE TABLE inbox_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    social_account_id UUID NOT NULL REFERENCES social_accounts(id),
    platform        VARCHAR(50) NOT NULL,
    platform_message_id VARCHAR(512) NOT NULL,
    message_type    VARCHAR(30) NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    author_platform_uid VARCHAR(512),
    author_name     VARCHAR(255),
    author_handle   VARCHAR(255),
    body_text       TEXT,
    parent_id       UUID REFERENCES inbox_messages(id),
    post_target_id  UUID REFERENCES post_targets(id),
    sentiment       VARCHAR(20),
    sentiment_score NUMERIC(5,4),
    language_code   VARCHAR(10),
    is_read         BOOLEAN NOT NULL DEFAULT false,
    assigned_to     UUID REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    received_at     TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inbox_tenant_status ON inbox_messages (tenant_id, status, received_at DESC);
CREATE INDEX idx_inbox_sentiment ON inbox_messages (tenant_id, sentiment);

-- ============================================================
-- ANALYTICS (relational — time-series data)
-- ============================================================
CREATE TABLE post_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_target_id  UUID NOT NULL REFERENCES post_targets(id) ON DELETE CASCADE,
    snapshot_at     TIMESTAMPTZ NOT NULL,
    metrics         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_post_analytics_target ON post_analytics (post_target_id, snapshot_at DESC);

CREATE TABLE account_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    social_account_id UUID NOT NULL REFERENCES social_accounts(id) ON DELETE CASCADE,
    snapshot_date   DATE NOT NULL,
    metrics         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (social_account_id, snapshot_date)
);
```

---

## Graph Layer (Relationship Analysis)

```sql
-- ============================================================
-- GRAPH NODES — entities that participate in relationships
-- ============================================================
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    /*  Node types (aligned with Activity Streams 2.0):
        'Person'          — social media person/profile (external)
        'Organization'    — brand, company, competitor
        'Note'            — post/tweet/toot (maps to our posts table)
        'Article'         — long-form content
        'Video'           — video content
        'Hashtag'         — hashtag entity
        'Topic'           — detected topic/theme
        'Campaign'        — marketing campaign
        'Platform'        — social platform
    */
    external_id     VARCHAR(512),       -- links to relational table ID or platform ID
    external_source VARCHAR(100),       -- 'posts', 'social_accounts', 'x', 'instagram', etc.
    label           VARCHAR(500) NOT NULL,  -- display name
    properties      JSONB NOT NULL DEFAULT '{}',
    /*  properties example (Person):
        {
            "handle": "@techinfluencer",
            "platform": "x",
            "platform_uid": "123456",
            "follower_count": 50000,
            "verified": true,
            "bio": "Tech reviewer and content creator",
            "country": "US",
            "language": "en",
            "influencer_score": 0.87,
            "audience_overlap_pct": 0.12
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_graph_nodes_tenant_type ON graph_nodes (tenant_id, node_type);
CREATE INDEX idx_graph_nodes_external ON graph_nodes (external_source, external_id);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN (properties jsonb_path_ops);
CREATE INDEX idx_graph_nodes_label ON graph_nodes USING GIN (to_tsvector('english', label));

-- ============================================================
-- GRAPH EDGES — relationships between nodes
-- ============================================================
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    /*  Edge types (aligned with Activity Streams 2.0 and social media):
        'FOLLOWS'         — Person follows Person/Organization
        'LIKED'           — Person liked Note/Article/Video
        'SHARED'          — Person shared/retweeted Note
        'REPLIED_TO'      — Person replied to Note
        'MENTIONED'       — Note mentions Person/Organization
        'AUTHORED'        — Person authored Note
        'TAGGED_WITH'     — Note tagged with Hashtag
        'ABOUT_TOPIC'     — Note/match is about Topic
        'MEMBER_OF'       — Person is member of Organization
        'COMPETES_WITH'   — Organization competes with Organization
        'SAME_PERSON_AS'  — Cross-platform identity link (Person on X = Person on LinkedIn)
        'INFLUENCED_BY'   — Content influenced by another content
        'ATTRIBUTED_TO'   — Conversion attributed to Note (attribution chain)
        'PART_OF'         — Note is part of Campaign
    */
    weight          NUMERIC(8,4) DEFAULT 1.0,  -- relationship strength/confidence
    properties      JSONB NOT NULL DEFAULT '{}',
    /*  properties example (SHARED):
        {
            "shared_at": "2026-05-19T14:30:00Z",
            "platform": "x",
            "share_type": "retweet",
            "reach_at_time_of_share": 5000
        }
    */
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),  -- temporal validity
    valid_to        TIMESTAMPTZ,                          -- null = currently valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_graph_edges_source ON graph_edges (source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_node_id, edge_type);
CREATE INDEX idx_graph_edges_tenant_type ON graph_edges (tenant_id, edge_type);
CREATE INDEX idx_graph_edges_temporal ON graph_edges (valid_from, valid_to);
CREATE INDEX idx_graph_edges_properties ON graph_edges USING GIN (properties jsonb_path_ops);

-- ============================================================
-- CROSS-PLATFORM IDENTITY RESOLUTION
-- ============================================================
CREATE TABLE identity_clusters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    cluster_label   VARCHAR(500),   -- resolved display name
    confidence      NUMERIC(5,4),    -- overall confidence score
    properties      JSONB NOT NULL DEFAULT '{}',
    /*  properties example:
        {
            "platforms": ["x", "linkedin", "instagram"],
            "handles": {"x": "@jsmith", "linkedin": "john-smith", "instagram": "jsmith_photo"},
            "total_reach": 75000,
            "is_influencer": true,
            "influencer_tier": "micro"
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_identity_clusters_tenant ON identity_clusters (tenant_id);

-- Links graph_nodes (Person) to their identity cluster
CREATE TABLE identity_cluster_members (
    cluster_id      UUID NOT NULL REFERENCES identity_clusters(id) ON DELETE CASCADE,
    node_id         UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    confidence      NUMERIC(5,4) NOT NULL,  -- confidence that this node belongs to the cluster
    linked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (cluster_id, node_id)
);
```

---

## Listening & Trend Detection (Graph-Enhanced)

```sql
CREATE TABLE listening_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Listening matches are stored both relationally AND as graph nodes
CREATE TABLE listening_matches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query_id        UUID NOT NULL REFERENCES listening_queries(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL,
    platform        VARCHAR(50) NOT NULL,
    author_handle   VARCHAR(255),
    body_text       TEXT,
    content_url     TEXT,
    sentiment       VARCHAR(20),
    sentiment_score NUMERIC(5,4),
    reach_estimate  INTEGER,
    graph_node_id   UUID REFERENCES graph_nodes(id),  -- link to graph for relationship analysis
    matched_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_listening_matches_query ON listening_matches (query_id, matched_at DESC);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    specversion     VARCHAR(10) NOT NULL DEFAULT '1.0',
    type            VARCHAR(200) NOT NULL,
    source          VARCHAR(500) NOT NULL,
    subject         VARCHAR(500),
    actor_id        UUID,
    data            JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant_time ON audit_log (tenant_id, occurred_at DESC);
```

---

## Example Graph Queries

### Find influencers who follow your brand AND your competitor

```sql
-- Using recursive CTE for graph traversal
WITH brand_followers AS (
    SELECT e.source_node_id AS person_node_id
    FROM graph_edges e
    JOIN graph_nodes brand ON brand.id = e.target_node_id
    WHERE e.edge_type = 'FOLLOWS'
      AND e.tenant_id = '...'
      AND brand.node_type = 'Organization'
      AND brand.label = 'My Brand'
      AND e.valid_to IS NULL
),
competitor_followers AS (
    SELECT e.source_node_id AS person_node_id
    FROM graph_edges e
    JOIN graph_nodes comp ON comp.id = e.target_node_id
    WHERE e.edge_type = 'FOLLOWS'
      AND e.tenant_id = '...'
      AND comp.node_type = 'Organization'
      AND comp.label = 'Competitor Brand'
      AND e.valid_to IS NULL
)
SELECT
    n.label AS person_name,
    n.properties->>'handle' AS handle,
    (n.properties->>'follower_count')::INTEGER AS follower_count,
    (n.properties->>'influencer_score')::NUMERIC AS influencer_score
FROM brand_followers bf
JOIN competitor_followers cf ON bf.person_node_id = cf.person_node_id
JOIN graph_nodes n ON n.id = bf.person_node_id
WHERE (n.properties->>'follower_count')::INTEGER > 1000
ORDER BY (n.properties->>'influencer_score')::NUMERIC DESC
LIMIT 20;
```

### Trace content virality (retweet chain depth)

```sql
-- Recursive CTE to trace how a post spread through shares
WITH RECURSIVE share_chain AS (
    -- Base: original post
    SELECT
        n.id AS node_id,
        n.label,
        n.properties->>'handle' AS handle,
        0 AS depth,
        ARRAY[n.id] AS path
    FROM graph_nodes n
    WHERE n.external_id = 'post-uuid-here'
      AND n.external_source = 'posts'

    UNION ALL

    -- Recursive: find who shared it
    SELECT
        sharer.id,
        sharer.label,
        sharer.properties->>'handle',
        sc.depth + 1,
        sc.path || sharer.id
    FROM share_chain sc
    JOIN graph_edges e ON e.target_node_id = sc.node_id
    JOIN graph_nodes sharer ON sharer.id = e.source_node_id
    WHERE e.edge_type = 'SHARED'
      AND sc.depth < 10  -- limit traversal depth
      AND NOT sharer.id = ANY(sc.path)  -- prevent cycles
)
SELECT depth, COUNT(*) AS shares_at_depth,
       SUM((properties->>'follower_count')::INTEGER) AS total_reach
FROM share_chain sc
JOIN graph_nodes n ON n.id = sc.node_id
GROUP BY depth
ORDER BY depth;
```

### Cross-platform identity: find all accounts for a person

```sql
SELECT
    ic.cluster_label AS resolved_name,
    n.properties->>'platform' AS platform,
    n.properties->>'handle' AS handle,
    (n.properties->>'follower_count')::INTEGER AS followers,
    icm.confidence AS match_confidence
FROM identity_clusters ic
JOIN identity_cluster_members icm ON icm.cluster_id = ic.id
JOIN graph_nodes n ON n.id = icm.node_id
WHERE ic.id = 'cluster-uuid-here'
ORDER BY (n.properties->>'follower_count')::INTEGER DESC;
```

### Attribution path: post -> engagement -> website visit -> purchase

```sql
WITH attribution_path AS (
    SELECT
        post_node.label AS post_title,
        e1.edge_type AS engagement_type,
        person_node.label AS person_name,
        e2.edge_type AS conversion_type,
        e2.properties->>'revenue' AS revenue,
        e2.properties->>'currency' AS currency
    FROM graph_nodes post_node
    JOIN graph_edges e1 ON e1.target_node_id = post_node.id
    JOIN graph_nodes person_node ON person_node.id = e1.source_node_id
    JOIN graph_edges e2 ON e2.source_node_id = person_node.id
    WHERE post_node.external_source = 'posts'
      AND post_node.tenant_id = '...'
      AND e1.edge_type IN ('LIKED', 'SHARED', 'REPLIED_TO')
      AND e2.edge_type = 'ATTRIBUTED_TO'
      AND e2.properties->>'conversion_type' = 'purchase'
)
SELECT post_title, COUNT(*) AS conversions, SUM(revenue::NUMERIC) AS total_revenue
FROM attribution_path
GROUP BY post_title
ORDER BY total_revenue DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | tenants, users |
| Social Accounts | 1 | social_accounts |
| Content & Publishing | 4 | posts, post_targets, media_assets, post_media |
| Inbox | 1 | inbox_messages |
| Analytics | 2 | post_analytics, account_analytics |
| Graph Layer | 4 | graph_nodes, graph_edges, identity_clusters, identity_cluster_members |
| Listening | 2 | listening_queries, listening_matches |
| Audit | 1 | audit_log |
| **Total** | **17** | Graph layer adds 4 tables over a basic relational model |

---

## Key Design Decisions

1. **PostgreSQL-native graph layer** — Rather than requiring a separate graph database (Neo4j), the graph is implemented as `graph_nodes` and `graph_edges` tables in the same PostgreSQL database. This simplifies deployment and enables transactional consistency between the relational core and the graph. For teams needing deep traversals (>5 hops), a dedicated graph database with Cypher queries would be more performant.

2. **Activity Streams 2.0 node types** — Graph node types (`Person`, `Note`, `Article`, `Video`, `Organization`) align with W3C Activity Streams 2.0 vocabulary. This makes the graph layer naturally compatible with ActivityPub federation and Mastodon/Bluesky data models.

3. **Temporal edges with `valid_from` / `valid_to`** — Social relationships change over time (follows, unfollows). The `valid_from` / `valid_to` columns on `graph_edges` enable temporal queries ("who was following us last quarter?") without deleting historical data.

4. **Cross-platform identity resolution** — The `identity_clusters` and `identity_cluster_members` tables solve a core challenge in social media management: recognising that @jsmith on X, john-smith on LinkedIn, and jsmith_photo on Instagram are the same person. The graph's `SAME_PERSON_AS` edges feed into the clustering algorithm, which assigns confidence scores.

5. **Weighted edges for influence scoring** — The `weight` column on `graph_edges` enables influence scoring algorithms. A share from a 100K-follower account has a higher weight than a like from a new account. This powers influencer identification features.

6. **Relational core for ACID operations** — Scheduling, publishing, and inbox routing require transactional guarantees that the graph layer doesn't provide as naturally. These stay in fully relational tables. The graph layer is used for analysis and insights, not operational workflows.

7. **Bidirectional index strategy** — Both `source_node_id` and `target_node_id` are indexed on `graph_edges`, enabling efficient traversal in both directions (who does this person follow? who follows this person?).

8. **Graph nodes link to relational entities** — The `external_id` and `external_source` columns on `graph_nodes` link graph entities back to relational tables (posts, social_accounts) or external platform IDs. This avoids data duplication while enabling graph queries that reference operational data.

9. **Listening matches bridge both worlds** — Listening matches are stored in a relational table for inbox-style browsing AND linked to graph nodes (`graph_node_id`) for relationship analysis. When a mention is found, a graph node is created for the author and edges are created for the interaction.
