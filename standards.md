# Standards & API Reference

> Project: Social Media Management · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022 — Information security management systems** — https://www.iso.org/standard/27001 — Baseline security controls for any SaaS handling customer social account credentials, OAuth tokens, and audience PII.
- **ISO/IEC 27018:2019 — Protection of PII in public clouds acting as PII processors** — https://www.iso.org/standard/76559.html — Relevant when storing audience profiles, engagement data, or DM contents on cloud infrastructure.
- **ISO/IEC 27701:2019 — Privacy Information Management** — https://www.iso.org/standard/71670.html — Extension of 27001 for managing PII; pertinent for audience data and inbox/CRM features.
- **ISO 8601:2019 — Date and time format** — https://www.iso.org/iso-8601-date-and-time-format.html — Required for cross-timezone scheduling, queue ordering, and analytics rollups.
- **ISO 639-1 / ISO 639-2 — Language codes** — https://www.iso.org/iso-639-language-code — Used in localised post variants and language detection for sentiment analysis pipelines.
- **ISO 3166-1 — Country codes** — https://www.iso.org/iso-3166-country-codes.html — Geo-targeting of posts and audience analytics segmentation.
- **ISO 4217 — Currency codes** — https://www.iso.org/iso-4217-currency-codes.html — Ad spend, ROAS, and revenue attribution reporting.

### W3C & IETF Standards

- **W3C ActivityPub** — https://www.w3.org/TR/activitypub/ — Decentralised federated social protocol used by Mastodon, Threads (federation), PeerTube, Pixelfed; required for any Fediverse publishing/listening capability.
- **W3C Activity Streams 2.0** — https://www.w3.org/TR/activitystreams-core/ — JSON data format underlying ActivityPub objects (Note, Article, Image, Video).
- **W3C WebSub** — https://www.w3.org/TR/websub/ — Pub/sub protocol useful for receiving real-time mention/reply notifications from federated networks.
- **W3C Web Content Accessibility Guidelines (WCAG) 2.2** — https://www.w3.org/TR/WCAG22/ — Accessibility requirements for the management dashboard and for guidance on accessible content (alt text, captions) being scheduled.
- **RFC 6749 — OAuth 2.0 Authorization Framework** — https://datatracker.ietf.org/doc/html/rfc6749 — Universal mechanism for connecting Meta, X, LinkedIn, TikTok, Pinterest, YouTube, Reddit accounts.
- **RFC 7636 — PKCE for OAuth Public Clients** — https://datatracker.ietf.org/doc/html/rfc7636 — Required by X API v2 and recommended by Meta for public/native clients.
- **RFC 6750 — OAuth 2.0 Bearer Token Usage** — https://datatracker.ietf.org/doc/html/rfc6750 — Token transport for all platform APIs.
- **RFC 8628 — OAuth 2.0 Device Authorization Grant** — https://datatracker.ietf.org/doc/html/rfc8628 — For CLI / TV-style account linking flows.
- **RFC 7519 — JSON Web Token (JWT)** — https://datatracker.ietf.org/doc/html/rfc7519 — Used by LinkedIn OIDC, internal session tokens, and webhook signing.
- **RFC 8259 — JSON** — https://datatracker.ietf.org/doc/html/rfc8259 — Wire format for all platform APIs and the project's own API surface.
- **RFC 5545 — iCalendar (iCal)** — https://datatracker.ietf.org/doc/html/rfc5545 — Recommended export format for content calendars.
- **RFC 5988 / 8288 — Web Linking** — https://datatracker.ietf.org/doc/html/rfc8288 — Pagination headers used by Meta Graph and LinkedIn APIs.
- **RFC 9110 — HTTP Semantics** — https://datatracker.ietf.org/doc/html/rfc9110 — Conditional requests, caching, and rate-limit handling against platform APIs.
- **RFC 6585 — Additional HTTP Status Codes (429 Too Many Requests)** — https://datatracker.ietf.org/doc/html/rfc6585 — Rate-limit response handling, central to platform integration robustness.

### Data Model & API Specifications

- **OpenAPI Specification 3.1** — https://spec.openapis.org/oas/v3.1.0 — Recommended for documenting the project's own REST API; aligns with JSON Schema 2020-12.
- **JSON Schema 2020-12** — https://json-schema.org/specification — Validation of post payloads, scheduling rules, and platform-specific constraints (character limits, media specs).
- **GraphQL (October 2021 spec)** — https://spec.graphql.org/October2021/ — Used by Meta Graph API; relevant when consuming and when exposing flexible analytics queries.
- **AsyncAPI 3.0** — https://www.asyncapi.com/docs/reference/specification/v3.0.0 — Documenting event-driven flows (webhook receivers, publish queues, real-time mention streams).
- **CloudEvents 1.0** — https://github.com/cloudevents/spec/blob/v1.0.2/cloudevents/spec.md — Standard envelope for internal events (post.published, mention.received, schedule.failed).
- **OpenTelemetry** — https://opentelemetry.io/docs/specs/otel/ — Tracing and metrics; especially valuable given rate-limit, queue, and webhook complexity.
- **Schema.org SocialMediaPosting / Article / VideoObject** — https://schema.org/SocialMediaPosting — Canonical content typing useful for cross-platform normalisation.
- **IPTC Photo Metadata Standard 2024.1** — https://www.iptc.org/std/photometadata/specification/IPTC-PhotoMetadata — Captions, credits, and rights info preserved through media uploads.
- **Open Graph Protocol** — https://ogp.me/ — Link preview generation and prediction.
- **oEmbed 1.0** — https://oembed.com/ — Embedding posts/cards in dashboards and approval workflows.
- **WebVTT** — https://www.w3.org/TR/webvtt1/ — Caption format for short-form video uploads (TikTok, Reels, Shorts).
- **EXIF / XMP** — https://www.cipa.jp/std/documents/e/DC-X008-Translation-2019-E.pdf — Stripping geolocation/PII from uploaded media before publishing.

### Security & Authentication Standards

- **OpenID Connect Core 1.0** — https://openid.net/specs/openid-connect-core-1_0.html — Used by LinkedIn, Google (YouTube), Apple sign-in flows for operator login.
- **OAuth 2.1 (draft)** — https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ — Consolidated security best practices including mandatory PKCE.
- **RFC 8414 — OAuth 2.0 Authorization Server Metadata** — https://datatracker.ietf.org/doc/html/rfc8414 — Discovery for SSO with enterprise IdPs.
- **OWASP API Security Top 10 (2023)** — https://owasp.org/API-Security/editions/2023/en/0x11-t10/ — Critical for an API exposed to agencies/teams managing many brand accounts.
- **OWASP ASVS 4.0** — https://owasp.org/www-project-application-security-verification-standard/ — Verification standard for the dashboard and API.
- **NIST SP 800-63B — Digital Identity Guidelines** — https://pages.nist.gov/800-63-3/sp800-63b.html — Authentication assurance for operator accounts.
- **NIST SP 800-53 Rev. 5** — https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final — Control catalogue commonly required by enterprise procurement.
- **SOC 2 Trust Services Criteria** — https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services — Routinely required for SMB+ buyers; informs logging, change management, access controls.
- **GDPR (Regulation (EU) 2016/679)** — https://gdpr-info.eu/ — Lawful basis, DSAR, and data residency for EU audience data and EU operators.
- **CCPA / CPRA** — https://oag.ca.gov/privacy/ccpa — California consumer privacy obligations for audience data.
- **Digital Services Act (Regulation (EU) 2022/2065)** — https://eur-lex.europa.eu/eli/reg/2022/2065/oj — Affects publishing and moderation features when interacting with EU platforms.
- **EU AI Act (Regulation (EU) 2024/1689)** — https://eur-lex.europa.eu/eli/reg/2024/1689/oj — Transparency obligations for AI-generated content (labelling) and risk classification of content moderation/sentiment systems.
- **C2PA Content Credentials 2.x** — https://c2pa.org/specifications/specifications/2.1/index.html — Provenance/watermarking for AI-generated images and video; aligns with Meta and TikTok labelling requirements.

### MCP Server Specifications

- **Model Context Protocol Specification** — https://modelcontextprotocol.io/specification/ — Base protocol for exposing scheduling, drafting, analytics, and inbox tools to LLM clients (Claude Desktop, IDEs, agent runtimes).
- **MCP Tools concept** — https://modelcontextprotocol.io/specification/server/tools — Recommended for actions (`schedule_post`, `reply_to_mention`, `pause_campaign`).
- **MCP Resources concept** — https://modelcontextprotocol.io/specification/server/resources — Surface read-only objects (post drafts, analytics snapshots, audience segments).
- **MCP Prompts concept** — https://modelcontextprotocol.io/specification/server/prompts — Pre-baked content workflows (brand-voice rewrite, weekly recap, sentiment digest).
- **MCP Transports (stdio, Streamable HTTP)** — https://modelcontextprotocol.io/specification/basic/transports — Streamable HTTP is the right fit for a hosted MCP server; stdio for self-hosted/local agents.
- **Reference servers (Anthropic)** — https://github.com/modelcontextprotocol/servers — Useful patterns for Slack/GitHub adapters that mirror social-platform integration shape.

## Similar Products — Developer Documentation & APIs

### Meta Graph API (Facebook Pages, Instagram, Threads)
- **Description:** Primary API for publishing to Facebook Pages, Instagram (Business/Creator), and Threads, plus reading insights, comments, and DMs.
- **API Documentation:** https://developers.facebook.com/docs/graph-api / https://developers.facebook.com/docs/instagram-platform / https://developers.facebook.com/docs/threads
- **SDKs/Libraries:** Official PHP, JS, Python (Business SDK) — https://developers.facebook.com/docs/business-sdk
- **Developer Guide:** https://developers.facebook.com/docs/development
- **Standards:** GraphQL-flavoured REST, JSON, OAuth 2.0, webhooks
- **Authentication:** OAuth 2.0 with long-lived Page/User access tokens; App Review required for most permissions

### X (Twitter) API v2
- **Description:** Posting, reading, and streaming X content; tier-based pricing affects scope.
- **API Documentation:** https://docs.x.com/x-api/introduction
- **SDKs/Libraries:** Community-maintained: tweepy (Python), twitter-api-v2 (Node) — https://docs.x.com/x-api/tools-and-libraries/overview
- **Developer Guide:** https://docs.x.com/x-api/getting-started/about-x-api
- **Standards:** REST/JSON, OAuth 2.0 with PKCE, OAuth 1.0a (legacy), filtered streams
- **Authentication:** OAuth 2.0 (User Context with PKCE) and OAuth 1.0a; App-only Bearer Tokens for read endpoints

### LinkedIn Marketing & Community Management APIs
- **Description:** Publishing on Pages, organic and paid analytics, comments and reactions management.
- **API Documentation:** https://learn.microsoft.com/en-us/linkedin/marketing/
- **SDKs/Libraries:** No first-party SDKs; community libraries (e.g. linkedin-api-python) — https://github.com/linkedin-developers
- **Developer Guide:** https://learn.microsoft.com/en-us/linkedin/marketing/getting-started
- **Standards:** REST/JSON, REST 2.0 (Rest.li protocol 2.0.0), OpenID Connect, JSON Schema-described entities
- **Authentication:** OAuth 2.0 (3-legged) with Marketing Developer Platform partner approval; OIDC for Sign-In with LinkedIn

### TikTok for Developers (Content Posting, Display, Business APIs)
- **Description:** Content Posting API for scheduled video uploads; Business API for analytics; Display API for embeds.
- **API Documentation:** https://developers.tiktok.com/doc/overview / https://business-api.tiktok.com/portal/docs
- **SDKs/Libraries:** Server SDKs (Node, Python, Java) for the Business API — https://business-api.tiktok.com/portal/docs?id=1738373164380162
- **Developer Guide:** https://developers.tiktok.com/doc/getting-started-create-an-app
- **Standards:** REST/JSON, OAuth 2.0, multipart resumable uploads (PULL_FROM_URL or FILE_UPLOAD)
- **Authentication:** OAuth 2.0; partner approval required for Content Posting scopes

### YouTube Data API v3
- **Description:** Video uploads, scheduling, comments, analytics, channel management.
- **API Documentation:** https://developers.google.com/youtube/v3
- **SDKs/Libraries:** Google API client libraries for Python, Node, Java, Go, .NET, Ruby, PHP — https://developers.google.com/api-client-library
- **Developer Guide:** https://developers.google.com/youtube/v3/getting-started
- **Standards:** REST/JSON, OAuth 2.0, resumable uploads, OpenAPI discovery documents
- **Authentication:** OAuth 2.0; quota system enforced per project

### Pinterest API v5
- **Description:** Pins, boards, audience insights, ad analytics.
- **API Documentation:** https://developers.pinterest.com/docs/api/v5/introduction/
- **SDKs/Libraries:** Official Python SDK — https://github.com/pinterest/pinterest-python-sdk
- **Developer Guide:** https://developers.pinterest.com/docs/getting-started/introduction/
- **Standards:** REST/JSON, OpenAPI 3 spec published, OAuth 2.0
- **Authentication:** OAuth 2.0 with PKCE

### Reddit Data API
- **Description:** Submission, comment, modmail, and listing endpoints for subreddit management and listening.
- **API Documentation:** https://www.reddit.com/dev/api/
- **SDKs/Libraries:** PRAW (Python, official) — https://praw.readthedocs.io/, snoowrap (Node, community)
- **Developer Guide:** https://github.com/reddit-archive/reddit/wiki/OAuth2
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0; commercial usage requires Data API agreement

### Mastodon API (ActivityPub)
- **Description:** Posting, timeline reading, notifications across the Fediverse via the Mastodon REST surface.
- **API Documentation:** https://docs.joinmastodon.org/api/
- **SDKs/Libraries:** mastodon.py, megalodon (Node), tuskyapi — https://docs.joinmastodon.org/client/libraries/
- **Developer Guide:** https://docs.joinmastodon.org/client/intro/
- **Standards:** REST/JSON, OAuth 2.0, ActivityPub federation, WebSocket streaming
- **Authentication:** OAuth 2.0 per-instance app registration

### Bluesky / AT Protocol
- **Description:** Posting, feeds, and listening on the AT Protocol-based Bluesky network.
- **API Documentation:** https://docs.bsky.app/ / https://atproto.com/specs/xrpc
- **SDKs/Libraries:** @atproto/api (TypeScript, official), atproto (Python, community) — https://github.com/bluesky-social/atproto
- **Developer Guide:** https://docs.bsky.app/docs/get-started
- **Standards:** XRPC (HTTP+JSON RPC), DID-based identity, Lexicon schema language
- **Authentication:** App passwords / OAuth (rolling out)

### Buffer Public API
- **Description:** Programmatic post scheduling and analytics retrieval across connected channels.
- **API Documentation:** https://buffer.com/developers/api
- **SDKs/Libraries:** Community wrappers; no current first-party SDK
- **Developer Guide:** https://buffer.com/developers/api/oauth
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0

### Hootsuite Platform API
- **Description:** Schedule posts, retrieve analytics, manage social profiles for partner integrations.
- **API Documentation:** https://developer.hootsuite.com/docs/platform-api-overview
- **SDKs/Libraries:** None first-party; HTTP-only
- **Developer Guide:** https://developer.hootsuite.com/docs/getting-started
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0; partner approval required

### Sprout Social API
- **Description:** Reporting and listening exports for enterprise integrations; limited write access.
- **API Documentation:** https://developers.sproutsocial.com/reference
- **SDKs/Libraries:** None first-party
- **Developer Guide:** https://developers.sproutsocial.com/docs
- **Standards:** REST/JSON
- **Authentication:** Bearer API tokens scoped to a customer

### Postiz (open source) API
- **Description:** Self-hostable scheduling platform with REST API for posts, channels, and analytics.
- **API Documentation:** https://docs.postiz.com/public-api
- **SDKs/Libraries:** None first-party; HTTP
- **Developer Guide:** https://docs.postiz.com/
- **Standards:** REST/JSON, OAuth 2.0 for downstream platform connections
- **Authentication:** API keys

### Mixpost (open source) API
- **Description:** Self-hosted Laravel-based scheduler with workspace-scoped REST API.
- **API Documentation:** https://docs.mixpost.app/api/
- **SDKs/Libraries:** None first-party
- **Developer Guide:** https://docs.mixpost.app/
- **Standards:** REST/JSON
- **Authentication:** API tokens

## Notes

- **Platform API churn is the dominant integration risk.** X (2023), TikTok (rolling), and Meta (annual deprecations) ship breaking changes far more often than typical SaaS APIs. Architectural patterns (adapter pattern, contract tests, capability negotiation) matter more than the specific spec versions cited above.
- **Partner approval is gating.** Meta, LinkedIn Marketing, and TikTok Content Posting all require app review before write/publish scopes are granted; plan a 4–8 week onboarding window per platform.
- **Federated networks (ActivityPub, AT Protocol) are still consolidating.** Expect schema additions; pin to a Lexicon/AS2 version and watch for OAuth rollout on Bluesky.
- **AI content labelling regulation is moving.** EU AI Act Article 50 transparency obligations and Meta/TikTok platform-level labelling for synthetic media intersect — adopting C2PA Content Credentials early is defensible.
- **No single ISO standard governs "social media management."** The applicable standards are a composition of security (27001/27018/27701), data interchange (8601, 639, 3166, 4217), and content (Schema.org, IPTC, oEmbed) standards plus per-platform proprietary APIs.
