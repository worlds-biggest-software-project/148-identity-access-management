# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Identity & Access Management (IAM) · Created: 2026-05-19

## Philosophy

This model combines a relational layer for operational CRUD (user profiles, credentials, tokens, sessions) with a property graph layer for relationship-heavy queries (permission resolution, ownership chains, group hierarchies, conflict-of-interest detection, access path analysis). The graph layer uses PostgreSQL's `ltree` extension for hierarchical data and a generic `graph_edges` table for arbitrary entity relationships, enabling efficient traversal queries without requiring a separate graph database.

The core insight is that IAM is fundamentally a graph problem. "Can user X access resource Y?" requires traversing a chain: user -> group memberships -> role assignments -> permissions -> resource. "Does this role assignment violate separation-of-duties?" requires finding paths between conflicting roles through the user. "Who can access this application through any path?" requires reverse traversal from the application through all possible permission chains. These queries are expensive in a purely relational model (recursive CTEs with multiple joins) but natural in a graph model (traverse edges from node A to node B).

Google Zanzibar (the system behind Google's authorization: Docs, Drive, Cloud IAM) uses a relationship-tuple model that is essentially a graph. SpiceDB, OpenFGA, and Ory Keto are all open-source implementations of Zanzibar's graph-based authorization. AWS IAM evaluates access by traversing policy graphs. Neo4j has published reference architectures for IAM graph models. This approach is particularly powerful for large enterprises with complex organizational hierarchies, cross-tenant access patterns, and compliance requirements that involve "who can reach what through any path?" analysis.

**Best for:** Organisations with complex organizational hierarchies, many-layered RBAC/ABAC, separation-of-duties enforcement, "blast radius" analysis (if this account is compromised, what can be reached?), and AI-powered access analytics that require graph traversal.

**Trade-offs:**
- Pro: Permission resolution queries are fast — graph traversal vs. multi-join recursive CTEs
- Pro: "Who has access to X?" and "What can user Y reach?" are first-class queries
- Pro: Separation-of-duties detection is a graph cycle/path problem — naturally efficient
- Pro: Blast radius analysis (compromised account impact) is a breadth-first graph traversal
- Pro: Hierarchical groups and organizational units are native (ltree)
- Con: More complex than pure relational — developers need to understand graph patterns
- Con: The graph_edges table can grow very large in enterprises with many relationships
- Con: Graph consistency must be maintained alongside relational tables (dual-write concern)
- Con: Less mature tooling compared to pure relational — fewer ORMs understand graph patterns
- Con: Requires careful edge-type design to avoid an overly generic "everything is a graph" model

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Google Zanzibar / OpenFGA | Relationship tuples (object#relation@subject) modeled as graph edges. Permission checks are graph reachability queries. |
| SCIM 2.0 (RFC 7643/7644) | User and Group attributes are relational. Group membership and manager relationships are graph edges. |
| OAuth 2.1 / OIDC | Client registration and token management are relational. Client-scope and user-application relationships are graph edges. |
| NIST SP 800-63B | AAL levels tracked on credential nodes. Authentication policy evaluation traverses the graph. |
| ISO/IEC 27001:2022 A.5.15-5.18 | Access control relationships are graph edges with full audit trail. Access certification traverses the graph to enumerate all access paths. |
| NIST SP 800-207 (Zero Trust) | Every access decision evaluates the identity-to-resource graph path. Continuous evaluation re-checks graph reachability. |
| RBAC (NIST INCITS 359) | Role hierarchies modeled as graph edges (role -> parent_role). Permission inheritance follows graph edges. |

---

## Relational Layer (Operational CRUD)

```sql
-- ============================================================
-- TENANTS
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ORGANIZATIONAL UNITS (ltree hierarchy)
-- ============================================================

CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE org_units (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    path            LTREE NOT NULL,                      -- e.g. 'acme.engineering.platform'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_units_path ON org_units USING GIST (path);
CREATE INDEX idx_org_units_tenant ON org_units (tenant_id);

-- Example org_units:
-- path = 'acme'                          (root)
-- path = 'acme.engineering'              (Engineering division)
-- path = 'acme.engineering.platform'     (Platform team)
-- path = 'acme.engineering.security'     (Security team)
-- path = 'acme.finance'                  (Finance division)
-- path = 'acme.finance.audit'            (Audit team)

-- ============================================================
-- USERS
-- ============================================================

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    org_unit_id         UUID REFERENCES org_units(id) ON DELETE SET NULL,
    username            VARCHAR(255) NOT NULL,
    email               VARCHAR(320),
    email_verified      BOOLEAN NOT NULL DEFAULT false,
    display_name        VARCHAR(255),
    given_name          VARCHAR(128),
    family_name         VARCHAR(128),
    title               VARCHAR(255),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    identity_state      VARCHAR(32) NOT NULL DEFAULT 'active'
                        CHECK (identity_state IN ('staged', 'active', 'suspended', 'deprovisioned')),
    profile             JSONB NOT NULL DEFAULT '{}',     -- SCIM extensions, custom attrs
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, username)
);

CREATE INDEX idx_users_tenant ON users (tenant_id, identity_state);
CREATE INDEX idx_users_email ON users (tenant_id, email);
CREATE INDEX idx_users_org_unit ON users (org_unit_id) WHERE org_unit_id IS NOT NULL;

-- ============================================================
-- GROUPS
-- ============================================================

CREATE TABLE groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    display_name    VARCHAR(255) NOT NULL,
    description     TEXT,
    group_type      VARCHAR(32) NOT NULL DEFAULT 'standard',
    parent_group_id UUID REFERENCES groups(id) ON DELETE SET NULL,  -- nested groups
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, display_name)
);

CREATE INDEX idx_groups_tenant ON groups (tenant_id);
CREATE INDEX idx_groups_parent ON groups (parent_group_id) WHERE parent_group_id IS NOT NULL;

-- ============================================================
-- ROLES (with hierarchy)
-- ============================================================

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,
    display_name    VARCHAR(255),
    description     TEXT,
    parent_role_id  UUID REFERENCES roles(id) ON DELETE SET NULL,  -- role inheritance
    role_type       VARCHAR(32) NOT NULL DEFAULT 'custom',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_roles_tenant ON roles (tenant_id);
CREATE INDEX idx_roles_parent ON roles (parent_role_id) WHERE parent_role_id IS NOT NULL;

-- ============================================================
-- PERMISSIONS (resources and actions)
-- ============================================================

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    resource_type   VARCHAR(128) NOT NULL,
    action          VARCHAR(64) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, resource_type, action)
);

-- ============================================================
-- OAUTH CLIENTS
-- ============================================================

CREATE TABLE oauth_clients (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    client_id           VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash  TEXT,
    client_name         VARCHAR(255) NOT NULL,
    client_type         VARCHAR(16) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    config              JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oauth_clients_client_id ON oauth_clients (client_id);

-- ============================================================
-- CREDENTIALS
-- ============================================================

CREATE TABLE credentials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    credential_type     VARCHAR(32) NOT NULL,
    display_name        VARCHAR(255),
    aal_level           SMALLINT NOT NULL DEFAULT 1,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    credential_data     JSONB NOT NULL,
    last_used_at        TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credentials_user ON credentials (user_id, credential_type);

-- ============================================================
-- SESSIONS & TOKENS (standard relational)
-- ============================================================

CREATE TABLE sessions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_token_hash  VARCHAR(128) NOT NULL UNIQUE,
    aal_achieved        SMALLINT NOT NULL DEFAULT 1,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    context             JSONB NOT NULL DEFAULT '{}',
    authenticated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at          TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_user ON sessions (user_id, is_active);
CREATE INDEX idx_sessions_token ON sessions (session_token_hash);

CREATE TABLE access_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_hash      VARCHAR(128) NOT NULL UNIQUE,
    client_id       UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    scope           TEXT NOT NULL,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_access_tokens_hash ON access_tokens (token_hash);

CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_hash      VARCHAR(128) NOT NULL UNIQUE,
    access_token_id UUID NOT NULL REFERENCES access_tokens(id) ON DELETE CASCADE,
    client_id       UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    scope           TEXT NOT NULL,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens (token_hash);
```

## Graph Layer

```sql
-- ============================================================
-- GRAPH EDGES (Zanzibar-inspired relationship tuples)
--
-- Models ALL identity relationships as typed edges:
--   user --member_of--> group
--   user --has_role--> role
--   group --has_role--> role
--   role --inherits--> role (parent)
--   role --grants--> permission
--   user --assigned_to--> application
--   group --assigned_to--> application
--   user --reports_to--> user (manager)
--   user --owns--> oauth_client (agent owner)
--   org_unit --contains--> org_unit (hierarchy)
-- ============================================================

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    -- Source node
    source_type     VARCHAR(64) NOT NULL,                -- user, group, role, permission, application, org_unit
    source_id       UUID NOT NULL,
    -- Relationship
    relation        VARCHAR(64) NOT NULL,                -- member_of, has_role, grants, inherits, assigned_to, reports_to, owns, contains
    -- Target node
    target_type     VARCHAR(64) NOT NULL,
    target_id       UUID NOT NULL,
    -- Edge metadata
    granted_by      UUID,                                -- who created this relationship
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,                         -- time-bound relationships
    condition       JSONB,                               -- conditional edges (e.g. "only when ip in range")
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Prevent duplicate edges
    UNIQUE (tenant_id, source_type, source_id, relation, target_type, target_id)
);

-- Primary traversal indexes
CREATE INDEX idx_edges_forward ON graph_edges (tenant_id, source_type, source_id, relation);
CREATE INDEX idx_edges_reverse ON graph_edges (tenant_id, target_type, target_id, relation);
CREATE INDEX idx_edges_relation ON graph_edges (tenant_id, relation);
CREATE INDEX idx_edges_expires ON graph_edges (expires_at) WHERE expires_at IS NOT NULL;

-- ============================================================
-- GRAPH EDGE TYPE REGISTRY
-- Documents allowed edge types and their semantics.
-- ============================================================

CREATE TABLE edge_type_registry (
    relation        VARCHAR(64) PRIMARY KEY,
    source_type     VARCHAR(64) NOT NULL,
    target_type     VARCHAR(64) NOT NULL,
    description     TEXT NOT NULL,
    is_transitive   BOOLEAN NOT NULL DEFAULT false,      -- should traversal follow this edge type?
    max_depth       SMALLINT,                            -- max traversal depth for transitive edges
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed edge types
INSERT INTO edge_type_registry (relation, source_type, target_type, description, is_transitive, max_depth) VALUES
    ('member_of',    'user',        'group',       'User is a member of group', false, NULL),
    ('has_role',     'user',        'role',        'User has been assigned a role', false, NULL),
    ('has_role',     'group',       'role',        'Group has been assigned a role (inherited by members)', false, NULL),
    ('inherits',     'role',        'role',        'Role inherits permissions from parent role', true, 10),
    ('grants',       'role',        'permission',  'Role grants a specific permission', false, NULL),
    ('assigned_to',  'user',        'application', 'User is assigned to an application', false, NULL),
    ('assigned_to',  'group',       'application', 'Group is assigned to an application', false, NULL),
    ('reports_to',   'user',        'user',        'User reports to another user (manager)', false, NULL),
    ('owns',         'user',        'application', 'User owns an OAuth client / agent', false, NULL),
    ('contains',     'org_unit',    'org_unit',    'Org unit contains child org unit', true, 20),
    ('belongs_to',   'user',        'org_unit',    'User belongs to an organizational unit', false, NULL);
```

## Separation of Duties (Graph-Native)

```sql
-- ============================================================
-- SEPARATION OF DUTIES POLICIES
-- SoD violations are detected by finding users who have paths
-- to BOTH roles in a conflicting pair.
-- ============================================================

CREATE TABLE sod_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    role_a_id       UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    role_b_id       UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    severity        VARCHAR(16) NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('info', 'warning', 'block')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (role_a_id <> role_b_id)
);

CREATE INDEX idx_sod_tenant ON sod_policies (tenant_id, is_active);

-- ============================================================
-- CERTIFICATION CAMPAIGNS
-- ============================================================

CREATE TABLE certification_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    campaign_type   VARCHAR(32) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'draft',
    owner_id        UUID NOT NULL REFERENCES users(id),
    config          JSONB NOT NULL DEFAULT '{}',
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE certification_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES certification_campaigns(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    edge_id         UUID REFERENCES graph_edges(id) ON DELETE SET NULL,  -- the specific relationship being reviewed
    reviewer_id     UUID REFERENCES users(id),
    access_summary  JSONB NOT NULL,
    decision        VARCHAR(16),
    decision_reason TEXT,
    risk_score      SMALLINT,
    decided_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cert_items_campaign ON certification_items (campaign_id);

-- ============================================================
-- AUDIT EVENTS
-- ============================================================

CREATE TABLE audit_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(32) NOT NULL,
    event_type      VARCHAR(64) NOT NULL,
    event_category  VARCHAR(32) NOT NULL,
    event_outcome   VARCHAR(16) NOT NULL,
    target_type     VARCHAR(64),
    target_id       UUID,
    -- Graph context: which edge(s) were affected?
    affected_edges  UUID[],
    details         JSONB NOT NULL DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_time ON audit_events (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_actor ON audit_events (actor_id, occurred_at DESC) WHERE actor_id IS NOT NULL;
CREATE INDEX idx_audit_type ON audit_events (event_type, occurred_at DESC);
```

## Example: Permission Check (Graph Traversal)

```sql
-- Can user X perform action 'write' on resource type 'document'?
-- Traverses: user -> (member_of) -> group -> (has_role) -> role -> (inherits)* -> role -> (grants) -> permission
-- Also: user -> (has_role) -> role -> (inherits)* -> role -> (grants) -> permission

WITH RECURSIVE accessible_roles AS (
    -- Direct role assignments to user
    SELECT e.target_id AS role_id, 1 AS depth
    FROM graph_edges e
    WHERE e.source_type = 'user'
      AND e.source_id = :user_id
      AND e.relation = 'has_role'
      AND e.tenant_id = :tenant_id
      AND (e.expires_at IS NULL OR e.expires_at > now())

    UNION

    -- Roles assigned via group membership
    SELECT gr.target_id AS role_id, 1 AS depth
    FROM graph_edges gm
    JOIN graph_edges gr ON gr.source_type = 'group'
                       AND gr.source_id = gm.target_id
                       AND gr.relation = 'has_role'
    WHERE gm.source_type = 'user'
      AND gm.source_id = :user_id
      AND gm.relation = 'member_of'
      AND gm.tenant_id = :tenant_id
      AND (gm.expires_at IS NULL OR gm.expires_at > now())
      AND (gr.expires_at IS NULL OR gr.expires_at > now())

    UNION

    -- Role inheritance (transitive)
    SELECT e.target_id AS role_id, ar.depth + 1 AS depth
    FROM accessible_roles ar
    JOIN graph_edges e ON e.source_type = 'role'
                      AND e.source_id = ar.role_id
                      AND e.relation = 'inherits'
    WHERE ar.depth < 10  -- prevent infinite recursion
)
SELECT EXISTS (
    SELECT 1
    FROM accessible_roles ar
    JOIN graph_edges e ON e.source_type = 'role'
                      AND e.source_id = ar.role_id
                      AND e.relation = 'grants'
    JOIN permissions p ON p.id = e.target_id
    WHERE p.resource_type = 'document'
      AND p.action = 'write'
) AS has_access;
```

## Example: Blast Radius Analysis

```sql
-- If user X's account is compromised, what applications can be reached?
-- Traverses ALL outbound paths from the user through groups, roles, and permissions.

WITH RECURSIVE reachable AS (
    -- Start from the compromised user
    SELECT
        source_type, source_id,
        relation,
        target_type, target_id,
        1 AS depth,
        ARRAY[source_id] AS path
    FROM graph_edges
    WHERE source_type = 'user'
      AND source_id = :user_id
      AND tenant_id = :tenant_id
      AND (expires_at IS NULL OR expires_at > now())

    UNION ALL

    -- Follow edges transitively
    SELECT
        e.source_type, e.source_id,
        e.relation,
        e.target_type, e.target_id,
        r.depth + 1,
        r.path || e.target_id
    FROM reachable r
    JOIN graph_edges e ON e.source_type = r.target_type
                      AND e.source_id = r.target_id
                      AND e.tenant_id = :tenant_id
                      AND (e.expires_at IS NULL OR e.expires_at > now())
    WHERE r.depth < 10
      AND e.target_id <> ALL(r.path)  -- prevent cycles
)
SELECT DISTINCT
    target_type,
    target_id,
    depth,
    array_to_string(path, ' -> ') AS access_path
FROM reachable
WHERE target_type = 'application'
ORDER BY depth;
```

## Example: Separation of Duties Violation Detection

```sql
-- Find all users who have paths to BOTH roles in any active SoD policy

SELECT
    sp.name AS policy_name,
    sp.severity,
    u.id AS user_id,
    u.username,
    ra.display_name AS role_a_name,
    rb.display_name AS role_b_name
FROM sod_policies sp
JOIN roles ra ON ra.id = sp.role_a_id
JOIN roles rb ON rb.id = sp.role_b_id
CROSS JOIN users u
WHERE sp.is_active = true
  AND sp.tenant_id = :tenant_id
  AND u.tenant_id = :tenant_id
  AND u.is_active = true
  -- User has path to role A
  AND EXISTS (
      SELECT 1 FROM graph_edges e
      WHERE (
          (e.source_type = 'user' AND e.source_id = u.id AND e.relation = 'has_role' AND e.target_id = sp.role_a_id)
          OR
          (e.source_type = 'group' AND e.relation = 'has_role' AND e.target_id = sp.role_a_id
           AND EXISTS (
               SELECT 1 FROM graph_edges gm
               WHERE gm.source_type = 'user' AND gm.source_id = u.id
                 AND gm.relation = 'member_of' AND gm.target_id = e.source_id
           ))
      )
      AND e.tenant_id = :tenant_id
      AND (e.expires_at IS NULL OR e.expires_at > now())
  )
  -- User has path to role B
  AND EXISTS (
      SELECT 1 FROM graph_edges e
      WHERE (
          (e.source_type = 'user' AND e.source_id = u.id AND e.relation = 'has_role' AND e.target_id = sp.role_b_id)
          OR
          (e.source_type = 'group' AND e.relation = 'has_role' AND e.target_id = sp.role_b_id
           AND EXISTS (
               SELECT 1 FROM graph_edges gm
               WHERE gm.source_type = 'user' AND gm.source_id = u.id
                 AND gm.relation = 'member_of' AND gm.target_id = e.source_id
           ))
      )
      AND e.tenant_id = :tenant_id
      AND (e.expires_at IS NULL OR e.expires_at > now())
  );
```

## Example: Organizational Hierarchy Query (ltree)

```sql
-- Find all users in the Engineering division and all its sub-units
SELECT u.id, u.username, u.display_name, ou.name AS org_unit, ou.path
FROM users u
JOIN org_units ou ON ou.id = u.org_unit_id
WHERE ou.path <@ 'acme.engineering'  -- ltree ancestor query
  AND u.tenant_id = :tenant_id
  AND u.is_active = true
ORDER BY ou.path, u.username;

-- Find all ancestor org units for a specific team
SELECT id, name, path
FROM org_units
WHERE path @> 'acme.engineering.platform'
ORDER BY nlevel(path);
-- Returns: acme, acme.engineering, acme.engineering.platform
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant Management | 1 | tenants |
| Organizational Hierarchy | 1 | org_units (ltree) |
| User Management | 1 | users |
| Groups | 1 | groups |
| Roles | 1 | roles (with parent hierarchy) |
| Permissions | 1 | permissions |
| Credentials | 1 | credentials |
| OAuth Clients | 1 | oauth_clients |
| Sessions | 1 | sessions |
| Tokens | 2 | access_tokens, refresh_tokens |
| Graph Layer | 2 | graph_edges, edge_type_registry |
| Identity Governance | 3 | sod_policies, certification_campaigns, certification_items |
| Audit | 1 | audit_events |
| **Total** | **17** | Plus relational integrity via graph_edges |

---

## Key Design Decisions

1. **Unified graph_edges table** — All relationships (user-group, user-role, group-role, role-permission, role-inheritance, org hierarchy, manager, agent ownership) are stored in a single graph_edges table with typed source/target nodes and named relations. This enables generic graph traversal queries that work across relationship types.

2. **Edge type registry** — The edge_type_registry documents allowed relationships, their transitivity (should traversal follow them?), and maximum depth. This prevents unbounded traversal and documents the graph schema.

3. **Dual-layer architecture** — Entity attributes (user profile, client config) are relational for efficient single-entity CRUD. Relationships between entities are graph edges for efficient traversal. This avoids the "everything is a graph node" anti-pattern while gaining graph query power.

4. **ltree for organizational hierarchy** — PostgreSQL's ltree extension models the organizational unit tree efficiently. Path queries ("all users under Engineering") use GIST indexes and are faster than recursive CTEs on adjacency-list tables.

5. **Time-bounded edges** — Graph edges support expires_at for just-in-time access. All traversal queries filter on expiry, ensuring that expired relationships are automatically excluded from permission checks.

6. **Conditional edges** — The condition JSONB column on graph_edges supports edges that are only valid under certain conditions (e.g., "this role assignment only applies when the user is on the corporate network"). This enables ABAC-style policies within the graph model.

7. **Certification items reference edges** — Access certification items point to the specific graph_edge being reviewed. When a certification results in revocation, the edge is deleted from the graph. This creates a direct link between governance decisions and access state.

8. **Blast radius as first-class query** — The graph model makes "if this account is compromised, what can be reached?" a breadth-first traversal with cycle detection. This query is impractical in a normalized relational model without the graph layer.

9. **No separate junction tables** — Unlike Suggestion 1 (which has user_roles, group_roles, group_members, role_permissions, application_assignments), all relationships are in graph_edges. The trade-off is weaker type safety (the edge_type_registry constrains types by convention, not by foreign keys).

10. **Audit events track affected edges** — Audit events include an affected_edges UUID array pointing to the graph edges that were created, modified, or deleted. This enables full audit trail reconstruction: "what access changes happened to this user in the last 30 days?" is a join between audit_events and graph_edges.
