# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Identity & Access Management (IAM) · Created: 2026-05-19

## Philosophy

This model treats every identity change as an immutable event appended to an event store. The event store is the single source of truth; all queryable state (user profiles, role memberships, active sessions, token validity) is derived by replaying or projecting events into materialised read models. This is the Command Query Responsibility Segregation (CQRS) pattern applied to IAM.

The core insight is that identity and access management is inherently an audit-driven domain. Every action — user creation, role assignment, password change, login attempt, access review decision — must be recorded with full context for compliance (ISO 27001, SOX, HIPAA, GDPR). In a traditional relational model, audit logging is bolted on as a secondary concern. In an event-sourced model, the audit trail IS the data — there is no separate audit log because the event store itself is the complete, immutable, temporally-ordered record of every state change.

This pattern is used in financial systems (ledger-based), healthcare (HL7 FHIR event streams), and security information and event management (SIEM) platforms. AWS CloudTrail, Azure Activity Log, and Okta System Log all follow event-sourced principles for identity events. SailPoint's identity governance analytics are built on event replay for temporal queries like "what access did this user have on date X?"

**Best for:** Organisations that require complete audit trails, need temporal querying ("what was true at time T?"), want AI-powered access analytics on historical patterns, or operate in heavily regulated environments (SOX, HIPAA, FedRAMP).

**Trade-offs:**
- Pro: Complete, immutable audit trail by construction — compliance is built-in, not bolted on
- Pro: Temporal queries are natural — "what roles did user X have on 2025-12-01?" is a simple event replay
- Pro: AI/ML analytics can consume the event stream directly for anomaly detection and role mining
- Pro: Event replay enables full reconstruction of state at any point in time
- Pro: Schema evolution is simpler — new event types can be added without altering existing tables
- Con: Higher storage requirements — events accumulate indefinitely
- Con: Read queries require materialised views that must be kept in sync (eventual consistency)
- Con: More complex implementation — developers must understand event sourcing and projection patterns
- Con: Snapshotting required for performance as event history grows
- Con: Debugging can be harder — current state requires understanding the event chain

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643/7644) | SCIM operations (POST /Users, PATCH /Users/{id}, DELETE /Users/{id}) map 1:1 to events. User attributes are reconstructed from events into SCIM-compliant read models. |
| OAuth 2.1 / OIDC | Token issuance, refresh, and revocation are events. Token validity is a projection. Client registration events track OAuth client lifecycle. |
| FIDO2 / WebAuthn | Credential registration, authentication, and counter updates are discrete events with full attestation context. |
| NIST SP 800-63B | AAL level achieved at each authentication is captured in the authentication event, enabling temporal AAL queries. |
| ISO/IEC 27001:2022 A.5.15-5.18 | The event store IS the audit evidence. Every access control change is an event with actor, target, timestamp, and context. |
| OCSF (Open Cybersecurity Schema Framework) | Event types and categories align with OCSF Identity & Access Management event classes for SIEM integration. |
| NIST SP 800-207 | Zero Trust continuous evaluation is natural — every access decision is an event that can be re-evaluated. |

---

## Event Store (Core)

```sql
-- ============================================================
-- IMMUTABLE EVENT STORE
-- This is the single source of truth. All other tables are
-- materialised projections derived from these events.
-- ============================================================

CREATE TABLE identity_events (
    -- Event identity
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sequence_number     BIGSERIAL NOT NULL UNIQUE,       -- global ordering
    -- Aggregate identity (the entity this event belongs to)
    aggregate_type      VARCHAR(64) NOT NULL,             -- user, group, role, client, session, policy, campaign
    aggregate_id        UUID NOT NULL,                    -- entity UUID
    tenant_id           UUID NOT NULL,
    -- Event type (the business action)
    event_type          VARCHAR(128) NOT NULL,
    event_version       SMALLINT NOT NULL DEFAULT 1,     -- schema version for this event type
    -- Event payload (the facts)
    payload             JSONB NOT NULL,
    -- Context (who, when, where, why)
    actor_id            UUID,                             -- who caused this event
    actor_type          VARCHAR(32) NOT NULL DEFAULT 'user'
                        CHECK (actor_type IN ('user', 'admin', 'system', 'agent', 'scim', 'idp')),
    actor_ip            INET,
    actor_user_agent    TEXT,
    correlation_id      UUID,                             -- links related events (e.g. login flow)
    causation_id        UUID,                             -- the event that caused this event
    -- Immutability
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- NOTE: No updated_at column — events are immutable
    CHECK (payload IS NOT NULL)
);

-- Partitioned by month for retention management
-- CREATE TABLE identity_events_2026_05 PARTITION OF identity_events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Primary query patterns
CREATE INDEX idx_events_aggregate ON identity_events (aggregate_type, aggregate_id, sequence_number);
CREATE INDEX idx_events_tenant_time ON identity_events (tenant_id, occurred_at DESC);
CREATE INDEX idx_events_type ON identity_events (event_type, occurred_at DESC);
CREATE INDEX idx_events_actor ON identity_events (actor_id, occurred_at DESC) WHERE actor_id IS NOT NULL;
CREATE INDEX idx_events_correlation ON identity_events (correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_tenant_aggregate ON identity_events (tenant_id, aggregate_type, occurred_at DESC);
```

## Event Type Catalogue

```sql
-- ============================================================
-- EVENT TYPE REGISTRY
-- Documents all known event types and their payload schemas.
-- ============================================================

CREATE TABLE event_type_registry (
    event_type          VARCHAR(128) PRIMARY KEY,
    aggregate_type      VARCHAR(64) NOT NULL,
    description         TEXT NOT NULL,
    payload_schema      JSONB NOT NULL,                  -- JSON Schema for the payload
    current_version     SMALLINT NOT NULL DEFAULT 1,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types:
-- user.created          — payload: {username, email, given_name, family_name, ...}
-- user.updated          — payload: {changes: [{field, old_value, new_value}, ...]}
-- user.suspended        — payload: {reason, suspended_by}
-- user.deprovisioned    — payload: {reason, effective_at}
-- user.password_changed — payload: {method: "self_service"|"admin_reset", aal_level}
-- user.mfa_enrolled     — payload: {credential_type, credential_id, aal_level}
-- user.mfa_removed      — payload: {credential_type, credential_id, reason}
-- user.login_succeeded  — payload: {session_id, credential_types_used, aal_achieved, ip, device}
-- user.login_failed     — payload: {reason, credential_type_attempted, ip, device}
-- role.created          — payload: {name, description, permissions}
-- role.assigned         — payload: {user_id, role_id, granted_by, expires_at}
-- role.revoked          — payload: {user_id, role_id, revoked_by, reason}
-- group.member_added    — payload: {user_id, group_id, added_by}
-- group.member_removed  — payload: {user_id, group_id, removed_by, reason}
-- client.registered     — payload: {client_id, client_name, grant_types, redirect_uris}
-- client.secret_rotated — payload: {client_id, rotated_by}
-- token.issued          — payload: {token_type, client_id, user_id, scope, expires_at}
-- token.revoked         — payload: {token_id, revoked_by, reason}
-- session.created       — payload: {user_id, ip, user_agent, aal_achieved}
-- session.terminated    — payload: {session_id, reason: "logout"|"expired"|"admin_revoke"}
-- campaign.created      — payload: {name, type, scope, owner_id, due_at}
-- campaign.item_decided — payload: {item_id, decision, reason, risk_score}
-- policy.created        — payload: {name, type, conditions, actions}
-- policy.updated        — payload: {changes: [{field, old_value, new_value}, ...]}
-- scim.user_provisioned — payload: {directory_id, scim_payload, user_id}
-- scim.user_deprovisioned — payload: {directory_id, user_id}
-- idp.linked            — payload: {user_id, idp_id, idp_user_id}
-- agent.registered      — payload: {client_id, owner_id, agent_type, scopes}
-- agent.action_taken    — payload: {client_id, action, target, on_behalf_of}
```

## Materialised Read Models (Projections)

```sql
-- ============================================================
-- PROJECTION: CURRENT USER STATE
-- Rebuilt by replaying user.* events. Updated by event handlers.
-- ============================================================

CREATE TABLE v_users (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    username            VARCHAR(255) NOT NULL,
    email               VARCHAR(320),
    email_verified      BOOLEAN NOT NULL DEFAULT false,
    display_name        VARCHAR(255),
    given_name          VARCHAR(128),
    family_name         VARCHAR(128),
    title               VARCHAR(255),
    preferred_language  VARCHAR(16),
    locale              VARCHAR(16),
    timezone            VARCHAR(64),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    identity_state      VARCHAR(32) NOT NULL DEFAULT 'active',
    -- Enterprise extension
    employee_number     VARCHAR(64),
    organization        VARCHAR(255),
    department          VARCHAR(255),
    manager_id          UUID,
    -- Derived metrics
    total_login_count   INTEGER NOT NULL DEFAULT 0,
    failed_login_count  INTEGER NOT NULL DEFAULT 0,
    last_login_at       TIMESTAMPTZ,
    last_failed_login_at TIMESTAMPTZ,
    credential_types    VARCHAR(32)[] DEFAULT '{}',       -- currently enrolled types
    max_aal             SMALLINT DEFAULT 1,               -- highest AAL achievable
    -- Projection metadata
    last_event_id       UUID NOT NULL,                    -- last event applied
    last_event_seq      BIGINT NOT NULL,                  -- for ordering
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_v_users_tenant_username ON v_users (tenant_id, username);
CREATE INDEX idx_v_users_tenant ON v_users (tenant_id, identity_state);
CREATE INDEX idx_v_users_email ON v_users (tenant_id, email);

-- ============================================================
-- PROJECTION: CURRENT ROLE ASSIGNMENTS
-- ============================================================

CREATE TABLE v_user_roles (
    user_id     UUID NOT NULL,
    role_id     UUID NOT NULL,
    role_name   VARCHAR(128) NOT NULL,
    tenant_id   UUID NOT NULL,
    source      VARCHAR(32) NOT NULL DEFAULT 'direct',   -- direct, group
    source_id   UUID,                                     -- group_id if source='group'
    granted_by  UUID,
    granted_at  TIMESTAMPTZ NOT NULL,
    expires_at  TIMESTAMPTZ,
    last_event_id UUID NOT NULL,
    PRIMARY KEY (user_id, role_id, source, COALESCE(source_id, '00000000-0000-0000-0000-000000000000'))
);

CREATE INDEX idx_v_user_roles_tenant ON v_user_roles (tenant_id);
CREATE INDEX idx_v_user_roles_role ON v_user_roles (role_id);

-- ============================================================
-- PROJECTION: CURRENT OAUTH CLIENTS
-- ============================================================

CREATE TABLE v_oauth_clients (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    client_id           VARCHAR(255) NOT NULL UNIQUE,
    client_name         VARCHAR(255) NOT NULL,
    client_type         VARCHAR(16) NOT NULL,
    is_agent            BOOLEAN NOT NULL DEFAULT false,
    agent_owner_id      UUID,
    redirect_uris       TEXT[],
    grant_types         VARCHAR(64)[],
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_oauth_clients_tenant ON v_oauth_clients (tenant_id);

-- ============================================================
-- PROJECTION: ACTIVE SESSIONS
-- ============================================================

CREATE TABLE v_sessions (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    user_id             UUID NOT NULL,
    aal_achieved        SMALLINT NOT NULL DEFAULT 1,
    ip_address          INET,
    user_agent          TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    authenticated_at    TIMESTAMPTZ NOT NULL,
    expires_at          TIMESTAMPTZ NOT NULL,
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_sessions_user ON v_sessions (user_id, is_active);
CREATE INDEX idx_v_sessions_expires ON v_sessions (expires_at) WHERE is_active = true;

-- ============================================================
-- PROJECTION: ACTIVE TOKENS
-- ============================================================

CREATE TABLE v_access_tokens (
    id              UUID PRIMARY KEY,
    token_hash      VARCHAR(128) NOT NULL UNIQUE,
    client_id       UUID NOT NULL,
    user_id         UUID,
    scope           TEXT NOT NULL,
    issued_at       TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    is_revoked      BOOLEAN NOT NULL DEFAULT false,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_tokens_hash ON v_access_tokens (token_hash);
CREATE INDEX idx_v_tokens_expires ON v_access_tokens (expires_at) WHERE is_revoked = false;

-- ============================================================
-- PROJECTION: GOVERNANCE CAMPAIGNS
-- ============================================================

CREATE TABLE v_certification_campaigns (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    campaign_type   VARCHAR(32) NOT NULL,
    status          VARCHAR(32) NOT NULL,
    total_items     INTEGER NOT NULL DEFAULT 0,
    decided_items   INTEGER NOT NULL DEFAULT 0,
    approved_items  INTEGER NOT NULL DEFAULT 0,
    revoked_items   INTEGER NOT NULL DEFAULT 0,
    owner_id        UUID NOT NULL,
    starts_at       TIMESTAMPTZ,
    due_at          TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_v_campaigns_tenant ON v_certification_campaigns (tenant_id, status);
```

## Snapshot Management

```sql
-- ============================================================
-- AGGREGATE SNAPSHOTS
-- Periodic snapshots avoid replaying entire event history.
-- ============================================================

CREATE TABLE aggregate_snapshots (
    aggregate_type      VARCHAR(64) NOT NULL,
    aggregate_id        UUID NOT NULL,
    tenant_id           UUID NOT NULL,
    snapshot_version    INTEGER NOT NULL,
    state               JSONB NOT NULL,                  -- full aggregate state at snapshot time
    last_event_id       UUID NOT NULL,
    last_event_seq      BIGINT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, snapshot_version)
);

CREATE INDEX idx_snapshots_latest ON aggregate_snapshots
    (aggregate_type, aggregate_id, snapshot_version DESC);

-- ============================================================
-- PROJECTION CHECKPOINTS
-- Tracks how far each projection has processed the event stream.
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(128) PRIMARY KEY,
    last_event_seq      BIGINT NOT NULL DEFAULT 0,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_healthy          BOOLEAN NOT NULL DEFAULT true,
    error_message       TEXT
);
```

## Operational Support Tables

```sql
-- ============================================================
-- TENANT CONFIGURATION (projection)
-- ============================================================

CREATE TABLE v_tenants (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    settings        JSONB NOT NULL DEFAULT '{}',
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- DEAD LETTER QUEUE
-- Events that failed to process in projections.
-- ============================================================

CREATE TABLE event_dead_letters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    projection_name VARCHAR(128) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    last_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dead_letters_projection ON event_dead_letters (projection_name, created_at);
```

## Example: Temporal Query — User Roles at a Point in Time

```sql
-- What roles did user X have on 2025-12-01?
-- This is trivial in event sourcing: replay events up to that date.

WITH role_events AS (
    SELECT
        event_type,
        payload,
        occurred_at,
        ROW_NUMBER() OVER (
            PARTITION BY payload->>'role_id'
            ORDER BY sequence_number DESC
        ) AS rn
    FROM identity_events
    WHERE aggregate_type = 'user'
      AND aggregate_id = :user_id
      AND event_type IN ('role.assigned', 'role.revoked')
      AND occurred_at <= '2025-12-01T23:59:59Z'
)
SELECT
    payload->>'role_id' AS role_id,
    payload->>'role_name' AS role_name,
    payload->>'granted_by' AS granted_by,
    occurred_at AS assigned_at
FROM role_events
WHERE rn = 1
  AND event_type = 'role.assigned';
```

## Example: Anomaly Detection Feed for AI/ML

```sql
-- Extract authentication events for ML anomaly detection pipeline
-- Returns structured features for each login attempt

SELECT
    event_id,
    aggregate_id AS user_id,
    tenant_id,
    event_type,
    payload->>'credential_types_used' AS credential_types,
    payload->>'aal_achieved' AS aal_level,
    actor_ip,
    payload->>'device_fingerprint' AS device,
    payload->>'country_code' AS country,
    occurred_at,
    -- Time-based features
    EXTRACT(HOUR FROM occurred_at) AS hour_of_day,
    EXTRACT(DOW FROM occurred_at) AS day_of_week,
    -- Lag features (time since last event for this user)
    occurred_at - LAG(occurred_at) OVER (
        PARTITION BY aggregate_id ORDER BY sequence_number
    ) AS time_since_last_event
FROM identity_events
WHERE event_type IN ('user.login_succeeded', 'user.login_failed')
  AND occurred_at >= now() - INTERVAL '30 days'
ORDER BY sequence_number;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | identity_events (partitioned by month) |
| Event Registry | 1 | event_type_registry |
| Snapshots & Checkpoints | 2 | aggregate_snapshots, projection_checkpoints |
| Error Handling | 1 | event_dead_letters |
| Projection: Tenants | 1 | v_tenants |
| Projection: Users | 1 | v_users |
| Projection: Role Assignments | 1 | v_user_roles |
| Projection: OAuth Clients | 1 | v_oauth_clients |
| Projection: Sessions | 1 | v_sessions |
| Projection: Tokens | 1 | v_access_tokens |
| Projection: Governance | 1 | v_certification_campaigns |
| **Total** | **12** | Plus additional projections as needed |

---

## Key Design Decisions

1. **Single event table** — All identity events go into one table (identity_events) rather than per-aggregate-type tables. This simplifies the event streaming infrastructure and enables cross-aggregate queries (e.g., "all events for tenant X in the last hour") while monthly partitioning manages table size.

2. **JSONB payloads with schema registry** — Event payloads are JSONB with schemas documented in event_type_registry. This enables payload evolution (adding fields to events) without ALTER TABLE migrations. The event_version field supports payload schema versioning.

3. **Correlation and causation IDs** — Every event can reference a correlation_id (grouping related events in a flow, e.g., all events in a login ceremony) and a causation_id (the event that triggered this event). This supports distributed tracing and audit chain reconstruction.

4. **Projections are disposable** — All v_* tables can be dropped and rebuilt from the event store. This means schema changes to read models are zero-risk — rebuild the projection. The projection_checkpoints table tracks how far each projection has processed.

5. **No separate audit log** — Unlike the normalized model, there is no audit_events table. The event store IS the audit log. Every event captures actor, target, timestamp, IP, and user agent. Compliance queries run directly against identity_events.

6. **Snapshot strategy** — After N events per aggregate (configurable, e.g., 100), a snapshot is stored containing the full aggregate state at that point. Rebuilding current state starts from the latest snapshot rather than the first event, bounding rebuild time.

7. **Dead letter queue** — Events that fail to project (due to bugs, schema mismatches, or transient errors) go to event_dead_letters for retry or manual investigation. This prevents a single bad event from blocking all projections.

8. **AI/ML-native event stream** — The event store doubles as a feature store for ML models. Authentication events contain structured fields (IP, device, time, credential type, country) that can be directly consumed by anomaly detection pipelines without ETL.

9. **Temporal queries by construction** — Answering "what was true at time T?" requires no additional tables or bi-temporal complexity. Simply replay events up to time T. This is critical for access certification ("was this access justified on the date of the incident?").

10. **Event-driven integrations** — External systems (SIEM, analytics, notification services) can subscribe to the event stream rather than polling tables. New integrations are added by creating new projections without modifying the write path.
