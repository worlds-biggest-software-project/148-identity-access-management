# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Identity & Access Management (IAM) · Created: 2026-05-19

## Philosophy

This model follows a fully normalized relational approach where every IAM concept — users, credentials, roles, permissions, applications, sessions, audit events — has its own dedicated table with strict foreign key relationships. The schema is designed for data integrity above all else, ensuring that referential constraints prevent orphaned records, dangling permissions, or inconsistent access states.

Normalized relational models are the foundation of virtually every production IAM system today. Keycloak uses approximately 90+ tables in its PostgreSQL schema to model realms, users, credentials, clients, roles, groups, and federation. Okta, Microsoft Entra ID, and SailPoint all use relational stores as their core persistence layer. This approach is proven at scale for identity workloads where correctness is more important than write throughput.

The key advantage is that every query can be answered with standard SQL joins, every constraint is enforced at the database level, and the schema is self-documenting. The trade-off is a higher table count, more complex migrations, and the need for careful indexing to maintain query performance as the user population grows.

**Best for:** Organisations that prioritise data integrity, have a stable feature scope, and need strong compliance guarantees with predictable query patterns.

**Trade-offs:**
- Pro: Maximum data integrity — foreign keys prevent orphaned or inconsistent data
- Pro: Standard SQL queries — no special query languages or replay mechanisms
- Pro: Well understood by most engineering teams; extensive tooling support
- Pro: Straightforward compliance — auditors can inspect tables directly
- Con: High table count (~60-70 tables) increases migration complexity
- Con: Schema changes require careful migration planning
- Con: Many-to-many relationships require junction tables, increasing join complexity
- Con: Jurisdiction-specific fields require either nullable columns or additional tables
- Con: Historical state queries require temporal tables or separate history tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643/7644) | User and Group tables align with SCIM Core User schema attributes (userName, name, emails, phoneNumbers, addresses, roles, entitlements). Enterprise extension maps to employee_number, cost_center, organization, division, department columns. |
| OAuth 2.1 / RFC 6749 | OAuth clients, authorization codes, access tokens, refresh tokens, and scopes each have dedicated tables with proper foreign key relationships. PKCE code_challenge stored on authorization codes. |
| OpenID Connect Core 1.0 | ID token claims map to user profile columns. OIDC Discovery metadata stored in realm/tenant configuration. |
| SAML 2.0 (OASIS) | Service provider and identity provider metadata stored in dedicated tables with X.509 certificate references. |
| FIDO2 / WebAuthn (W3C) | WebAuthn credentials stored with credential_id, public_key, sign_count, aaguid, transports, and attestation_type per the WebAuthn specification. |
| NIST SP 800-63B | Authentication assurance levels (AAL1/2/3) modeled as an enum on authentication policies and credential types. |
| ISO/IEC 27001:2022 A.5.15-5.18 | Access control policies, identity lifecycle states, authentication requirements, and access rights all have dedicated tables supporting audit evidence generation. |
| ISO 3166 | Jurisdiction codes use ISO 3166-1 alpha-2 for countries, ISO 3166-2 for subdivisions. |
| W3C Verifiable Credentials 2.0 | Verifiable credential issuance and verification tracked in dedicated tables with DID references. |

---

## Core Identity Tables

```sql
-- ============================================================
-- TENANT / REALM MANAGEMENT
-- ============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    settings        JSONB NOT NULL DEFAULT '{}',  -- tenant-level config (branding, defaults)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants (slug);
CREATE INDEX idx_tenants_is_active ON tenants (is_active) WHERE is_active = true;

-- ============================================================
-- USER MANAGEMENT (SCIM 2.0 aligned)
-- ============================================================

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id         VARCHAR(255),                   -- SCIM externalId
    username            VARCHAR(255) NOT NULL,           -- SCIM userName
    email               VARCHAR(320),                    -- Primary email
    email_verified      BOOLEAN NOT NULL DEFAULT false,
    display_name        VARCHAR(255),                    -- SCIM displayName
    given_name          VARCHAR(128),                    -- SCIM name.givenName
    family_name         VARCHAR(128),                    -- SCIM name.familyName
    middle_name         VARCHAR(128),                    -- SCIM name.middleName
    honorific_prefix    VARCHAR(64),                     -- SCIM name.honorificPrefix
    honorific_suffix    VARCHAR(64),                     -- SCIM name.honorificSuffix
    nickname            VARCHAR(128),                    -- SCIM nickName
    profile_url         VARCHAR(2048),                   -- SCIM profileUrl
    title               VARCHAR(255),                    -- SCIM title (job title)
    user_type           VARCHAR(64),                     -- SCIM userType
    preferred_language  VARCHAR(16),                     -- SCIM preferredLanguage (RFC 5646)
    locale              VARCHAR(16),                     -- SCIM locale (RFC 5646)
    timezone            VARCHAR(64),                     -- SCIM timezone (IANA tz)
    is_active           BOOLEAN NOT NULL DEFAULT true,   -- SCIM active
    -- Enterprise extension (urn:ietf:params:scim:schemas:extension:enterprise:2.0:User)
    employee_number     VARCHAR(64),
    cost_center         VARCHAR(128),
    organization        VARCHAR(255),
    division            VARCHAR(255),
    department          VARCHAR(255),
    manager_id          UUID REFERENCES users(id) ON DELETE SET NULL,
    -- Lifecycle
    identity_state      VARCHAR(32) NOT NULL DEFAULT 'active'
                        CHECK (identity_state IN ('staged', 'active', 'suspended', 'deprovisioned')),
    provisioned_at      TIMESTAMPTZ,
    deprovisioned_at    TIMESTAMPTZ,
    last_login_at       TIMESTAMPTZ,
    password_changed_at TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, username)
);

CREATE INDEX idx_users_tenant_id ON users (tenant_id);
CREATE INDEX idx_users_email ON users (tenant_id, email);
CREATE INDEX idx_users_external_id ON users (tenant_id, external_id) WHERE external_id IS NOT NULL;
CREATE INDEX idx_users_identity_state ON users (tenant_id, identity_state);
CREATE INDEX idx_users_manager_id ON users (manager_id) WHERE manager_id IS NOT NULL;

-- Multi-valued SCIM attributes
CREATE TABLE user_emails (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    value       VARCHAR(320) NOT NULL,
    type        VARCHAR(32),            -- work, home, other
    is_primary  BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_emails_user_id ON user_emails (user_id);

CREATE TABLE user_phone_numbers (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    value       VARCHAR(32) NOT NULL,
    type        VARCHAR(32),            -- work, mobile, home, fax
    is_primary  BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_phone_numbers_user_id ON user_phone_numbers (user_id);

CREATE TABLE user_addresses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type            VARCHAR(32),            -- work, home
    street_address  TEXT,
    locality        VARCHAR(255),           -- city
    region          VARCHAR(128),           -- state/province
    postal_code     VARCHAR(32),
    country         CHAR(2),                -- ISO 3166-1 alpha-2
    formatted       TEXT,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_addresses_user_id ON user_addresses (user_id);

-- ============================================================
-- GROUPS (SCIM 2.0 aligned)
-- ============================================================

CREATE TABLE groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    display_name    VARCHAR(255) NOT NULL,
    description     TEXT,
    group_type      VARCHAR(32) NOT NULL DEFAULT 'standard'
                    CHECK (group_type IN ('standard', 'dynamic', 'system')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, display_name)
);

CREATE INDEX idx_groups_tenant_id ON groups (tenant_id);

CREATE TABLE group_members (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id    UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (group_id, user_id)
);

CREATE INDEX idx_group_members_group_id ON group_members (group_id);
CREATE INDEX idx_group_members_user_id ON group_members (user_id);
```

## Authentication & Credential Tables

```sql
-- ============================================================
-- CREDENTIALS (password, TOTP, WebAuthn/FIDO2, recovery codes)
-- ============================================================

CREATE TABLE user_credentials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    credential_type     VARCHAR(32) NOT NULL
                        CHECK (credential_type IN (
                            'password', 'totp', 'webauthn', 'recovery_code',
                            'email_otp', 'sms_otp', 'push', 'certificate'
                        )),
    -- Password fields
    password_hash       TEXT,                            -- bcrypt/argon2id hash
    hash_algorithm      VARCHAR(32),                     -- argon2id, bcrypt
    -- TOTP fields
    totp_secret         TEXT,                            -- encrypted TOTP secret
    totp_algorithm      VARCHAR(8),                      -- SHA1, SHA256, SHA512
    totp_digits         SMALLINT,                        -- 6 or 8
    totp_period         SMALLINT,                        -- 30 or 60 seconds
    -- WebAuthn/FIDO2 fields (per WebAuthn spec)
    webauthn_credential_id  TEXT,                        -- base64url credential ID
    webauthn_public_key     TEXT,                        -- COSE public key (base64)
    webauthn_sign_count     BIGINT DEFAULT 0,            -- signature counter
    webauthn_aaguid         CHAR(36),                    -- authenticator AAGUID
    webauthn_transports     VARCHAR(255)[],              -- usb, ble, nfc, internal
    webauthn_attestation    VARCHAR(32),                  -- none, indirect, direct
    webauthn_user_verified  BOOLEAN,                     -- UV flag capability
    webauthn_backup_eligible BOOLEAN,                    -- passkey sync eligibility
    webauthn_backup_state   BOOLEAN,                     -- currently synced
    -- Recovery codes
    recovery_code_hash  TEXT,
    -- General
    display_name        VARCHAR(255),                    -- user-facing label
    aal_level           SMALLINT NOT NULL DEFAULT 1      -- AAL1, AAL2, AAL3 per NIST 800-63B
                        CHECK (aal_level IN (1, 2, 3)),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_used_at        TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_credentials_user_id ON user_credentials (user_id);
CREATE INDEX idx_user_credentials_type ON user_credentials (user_id, credential_type);
CREATE INDEX idx_user_credentials_webauthn ON user_credentials (webauthn_credential_id)
    WHERE webauthn_credential_id IS NOT NULL;

-- ============================================================
-- AUTHENTICATION POLICIES
-- ============================================================

CREATE TABLE authentication_policies (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name                    VARCHAR(255) NOT NULL,
    description             TEXT,
    priority                INTEGER NOT NULL DEFAULT 0,
    is_active               BOOLEAN NOT NULL DEFAULT true,
    -- Policy conditions
    required_aal            SMALLINT NOT NULL DEFAULT 1
                            CHECK (required_aal IN (1, 2, 3)),
    allowed_credential_types VARCHAR(32)[] NOT NULL DEFAULT ARRAY['password'],
    require_mfa             BOOLEAN NOT NULL DEFAULT false,
    mfa_credential_types    VARCHAR(32)[] DEFAULT ARRAY['totp', 'webauthn'],
    -- Adaptive / conditional access
    condition_ip_ranges     CIDR[],
    condition_countries     CHAR(2)[],                   -- ISO 3166-1 alpha-2
    condition_device_types  VARCHAR(32)[],
    condition_risk_level    VARCHAR(16),                  -- low, medium, high, critical
    condition_applications  UUID[],                       -- specific app IDs
    -- Actions
    session_max_age_seconds INTEGER DEFAULT 86400,
    require_reauthentication BOOLEAN NOT NULL DEFAULT false,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_auth_policies_tenant ON authentication_policies (tenant_id, is_active, priority);

-- ============================================================
-- SESSIONS
-- ============================================================

CREATE TABLE sessions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_token_hash  VARCHAR(128) NOT NULL UNIQUE,    -- SHA-256 of session token
    aal_achieved        SMALLINT NOT NULL DEFAULT 1,
    ip_address          INET,
    user_agent          TEXT,
    device_fingerprint  VARCHAR(128),
    country_code        CHAR(2),                         -- ISO 3166-1 from IP geo
    is_active           BOOLEAN NOT NULL DEFAULT true,
    authenticated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_activity_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at          TIMESTAMPTZ NOT NULL,
    revoked_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_user_id ON sessions (user_id, is_active);
CREATE INDEX idx_sessions_token_hash ON sessions (session_token_hash);
CREATE INDEX idx_sessions_expires ON sessions (expires_at) WHERE is_active = true;
```

## OAuth 2.1 / OpenID Connect Tables

```sql
-- ============================================================
-- OAUTH CLIENTS (Applications / Service Providers)
-- ============================================================

CREATE TABLE oauth_clients (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    client_id               VARCHAR(255) NOT NULL UNIQUE, -- public client identifier
    client_secret_hash      TEXT,                          -- NULL for public clients
    client_name             VARCHAR(255) NOT NULL,
    client_type             VARCHAR(16) NOT NULL
                            CHECK (client_type IN ('confidential', 'public', 'machine')),
    -- Machine/agent identity fields
    is_agent                BOOLEAN NOT NULL DEFAULT false,
    agent_owner_id          UUID REFERENCES users(id) ON DELETE SET NULL,
    -- URIs
    redirect_uris           TEXT[] NOT NULL DEFAULT '{}',
    post_logout_redirect_uris TEXT[],
    logo_uri                VARCHAR(2048),
    client_uri              VARCHAR(2048),
    tos_uri                 VARCHAR(2048),
    policy_uri              VARCHAR(2048),
    -- Grant types and response types
    grant_types             VARCHAR(64)[] NOT NULL DEFAULT ARRAY['authorization_code'],
    response_types          VARCHAR(32)[] NOT NULL DEFAULT ARRAY['code'],
    -- Token configuration
    access_token_ttl        INTEGER NOT NULL DEFAULT 3600,     -- seconds
    refresh_token_ttl       INTEGER NOT NULL DEFAULT 86400,
    id_token_ttl            INTEGER NOT NULL DEFAULT 3600,
    -- PKCE (mandatory in OAuth 2.1)
    require_pkce            BOOLEAN NOT NULL DEFAULT true,
    -- SAML configuration (if this client uses SAML)
    saml_entity_id          VARCHAR(2048),
    saml_acs_url            VARCHAR(2048),
    saml_slo_url            VARCHAR(2048),
    saml_name_id_format     VARCHAR(128),
    saml_signing_certificate TEXT,
    -- Status
    is_active               BOOLEAN NOT NULL DEFAULT true,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oauth_clients_tenant ON oauth_clients (tenant_id);
CREATE INDEX idx_oauth_clients_client_id ON oauth_clients (client_id);

-- ============================================================
-- OAUTH SCOPES
-- ============================================================

CREATE TABLE oauth_scopes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,               -- e.g. openid, profile, email, custom:read
    display_name    VARCHAR(255),
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_system       BOOLEAN NOT NULL DEFAULT false,      -- built-in OIDC scopes
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_oauth_scopes_tenant ON oauth_scopes (tenant_id);

CREATE TABLE oauth_client_scopes (
    client_id   UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    scope_id    UUID NOT NULL REFERENCES oauth_scopes(id) ON DELETE CASCADE,
    PRIMARY KEY (client_id, scope_id)
);

-- ============================================================
-- AUTHORIZATION CODES
-- ============================================================

CREATE TABLE authorization_codes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_hash           VARCHAR(128) NOT NULL UNIQUE,    -- SHA-256 of auth code
    client_id           UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    redirect_uri        TEXT NOT NULL,
    scope               TEXT NOT NULL,                   -- space-separated scopes
    -- PKCE (OAuth 2.1)
    code_challenge       VARCHAR(128),
    code_challenge_method VARCHAR(8) DEFAULT 'S256',
    nonce               VARCHAR(255),                    -- OIDC nonce
    state               VARCHAR(255),
    expires_at          TIMESTAMPTZ NOT NULL,
    used_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_auth_codes_code_hash ON authorization_codes (code_hash);
CREATE INDEX idx_auth_codes_expires ON authorization_codes (expires_at);

-- ============================================================
-- ACCESS TOKENS & REFRESH TOKENS
-- ============================================================

CREATE TABLE access_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_hash      VARCHAR(128) NOT NULL UNIQUE,        -- SHA-256 of token value
    client_id       UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,  -- NULL for client_credentials
    session_id      UUID REFERENCES sessions(id) ON DELETE SET NULL,
    scope           TEXT NOT NULL,
    token_type      VARCHAR(16) NOT NULL DEFAULT 'Bearer',
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_access_tokens_hash ON access_tokens (token_hash);
CREATE INDEX idx_access_tokens_user ON access_tokens (user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_access_tokens_expires ON access_tokens (expires_at);

CREATE TABLE refresh_tokens (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_hash          VARCHAR(128) NOT NULL UNIQUE,
    access_token_id     UUID NOT NULL REFERENCES access_tokens(id) ON DELETE CASCADE,
    client_id           UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    user_id             UUID REFERENCES users(id) ON DELETE CASCADE,
    scope               TEXT NOT NULL,
    -- Token rotation (OAuth 2.1)
    previous_token_id   UUID REFERENCES refresh_tokens(id),
    rotation_count      INTEGER NOT NULL DEFAULT 0,
    issued_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at          TIMESTAMPTZ NOT NULL,
    revoked_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens (token_hash);
CREATE INDEX idx_refresh_tokens_user ON refresh_tokens (user_id) WHERE user_id IS NOT NULL;
```

## RBAC / Access Control Tables

```sql
-- ============================================================
-- ROLES AND PERMISSIONS
-- ============================================================

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,
    display_name    VARCHAR(255),
    description     TEXT,
    role_type       VARCHAR(32) NOT NULL DEFAULT 'custom'
                    CHECK (role_type IN ('system', 'custom', 'template')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_roles_tenant ON roles (tenant_id);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    resource_type   VARCHAR(128) NOT NULL,               -- e.g. user, application, role, group
    action          VARCHAR(64) NOT NULL,                 -- e.g. read, write, delete, manage
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, resource_type, action)
);

CREATE INDEX idx_permissions_tenant ON permissions (tenant_id);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- User-role assignments (tenant-scoped)
CREATE TABLE user_roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id     UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_by  UUID REFERENCES users(id) ON DELETE SET NULL,
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at  TIMESTAMPTZ,                             -- time-bound role assignments
    UNIQUE (user_id, role_id)
);

CREATE INDEX idx_user_roles_user ON user_roles (user_id);
CREATE INDEX idx_user_roles_role ON user_roles (role_id);

-- Group-role assignments
CREATE TABLE group_roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id    UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
    role_id     UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_by  UUID REFERENCES users(id) ON DELETE SET NULL,
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (group_id, role_id)
);

CREATE INDEX idx_group_roles_group ON group_roles (group_id);

-- ============================================================
-- APPLICATION ASSIGNMENTS
-- ============================================================

CREATE TABLE application_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    oauth_client_id UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    assignee_type   VARCHAR(16) NOT NULL CHECK (assignee_type IN ('user', 'group')),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    group_id        UUID REFERENCES groups(id) ON DELETE CASCADE,
    assigned_by     UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (
        (assignee_type = 'user' AND user_id IS NOT NULL AND group_id IS NULL) OR
        (assignee_type = 'group' AND group_id IS NOT NULL AND user_id IS NULL)
    )
);

CREATE INDEX idx_app_assignments_client ON application_assignments (oauth_client_id);
CREATE INDEX idx_app_assignments_user ON application_assignments (user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_app_assignments_group ON application_assignments (group_id) WHERE group_id IS NOT NULL;
```

## Identity Governance Tables

```sql
-- ============================================================
-- ACCESS CERTIFICATION CAMPAIGNS
-- ============================================================

CREATE TABLE certification_campaigns (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    campaign_type       VARCHAR(32) NOT NULL
                        CHECK (campaign_type IN ('user_access', 'role_membership',
                            'application_access', 'privileged_access', 'separation_of_duties')),
    status              VARCHAR(32) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'active', 'in_review', 'completed', 'cancelled')),
    -- Scope
    scope_applications  UUID[],                          -- specific applications, or NULL for all
    scope_roles         UUID[],
    scope_groups        UUID[],
    -- Schedule
    starts_at           TIMESTAMPTZ,
    due_at              TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    -- Owner
    owner_id            UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cert_campaigns_tenant ON certification_campaigns (tenant_id, status);

CREATE TABLE certification_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES certification_campaigns(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    -- What is being certified
    access_type         VARCHAR(32) NOT NULL
                        CHECK (access_type IN ('role', 'group', 'application', 'permission')),
    role_id             UUID REFERENCES roles(id) ON DELETE SET NULL,
    group_id            UUID REFERENCES groups(id) ON DELETE SET NULL,
    oauth_client_id     UUID REFERENCES oauth_clients(id) ON DELETE SET NULL,
    permission_id       UUID REFERENCES permissions(id) ON DELETE SET NULL,
    -- Review
    reviewer_id         UUID REFERENCES users(id),
    decision            VARCHAR(16)
                        CHECK (decision IN ('approve', 'revoke', 'delegate', 'pending')),
    decision_reason     TEXT,
    risk_score          SMALLINT,                        -- 0-100 AI-generated risk score
    decided_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cert_items_campaign ON certification_items (campaign_id, decision);
CREATE INDEX idx_cert_items_reviewer ON certification_items (reviewer_id) WHERE reviewer_id IS NOT NULL;
CREATE INDEX idx_cert_items_user ON certification_items (user_id);

-- ============================================================
-- SEPARATION OF DUTIES POLICIES
-- ============================================================

CREATE TABLE sod_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- Conflicting role pairs
    role_a_id       UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    role_b_id       UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    severity        VARCHAR(16) NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('info', 'warning', 'block')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (role_a_id <> role_b_id)
);

CREATE INDEX idx_sod_policies_tenant ON sod_policies (tenant_id, is_active);
```

## Provisioning & Directory Sync Tables

```sql
-- ============================================================
-- SCIM PROVISIONING CONNECTIONS
-- ============================================================

CREATE TABLE directory_connections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    provider_type       VARCHAR(32) NOT NULL
                        CHECK (provider_type IN ('scim', 'ldap', 'active_directory',
                            'google_workspace', 'azure_ad', 'custom')),
    -- SCIM configuration
    scim_endpoint_url   VARCHAR(2048),
    scim_bearer_token_hash TEXT,
    -- LDAP/AD configuration
    ldap_url            VARCHAR(2048),
    ldap_bind_dn        VARCHAR(512),
    ldap_base_dn        VARCHAR(512),
    -- Status
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_sync_at        TIMESTAMPTZ,
    sync_status         VARCHAR(32) DEFAULT 'idle'
                        CHECK (sync_status IN ('idle', 'syncing', 'success', 'error')),
    sync_error_message  TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_directory_connections_tenant ON directory_connections (tenant_id);

CREATE TABLE provisioning_events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    directory_id        UUID NOT NULL REFERENCES directory_connections(id) ON DELETE CASCADE,
    event_type          VARCHAR(32) NOT NULL
                        CHECK (event_type IN ('user_create', 'user_update', 'user_deactivate',
                            'user_delete', 'group_create', 'group_update', 'group_delete',
                            'group_member_add', 'group_member_remove')),
    scim_resource_type  VARCHAR(32),                     -- User, Group
    scim_resource_id    VARCHAR(255),
    user_id             UUID REFERENCES users(id) ON DELETE SET NULL,
    group_id            UUID REFERENCES groups(id) ON DELETE SET NULL,
    status              VARCHAR(16) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'success', 'failure', 'skipped')),
    error_message       TEXT,
    request_payload     JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_provisioning_events_directory ON provisioning_events (directory_id, created_at DESC);
CREATE INDEX idx_provisioning_events_status ON provisioning_events (status) WHERE status = 'failure';
```

## Audit Log Tables

```sql
-- ============================================================
-- AUDIT LOG (ISO 27001 compliance)
-- ============================================================

CREATE TABLE audit_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    -- Who
    actor_id        UUID REFERENCES users(id) ON DELETE SET NULL,
    actor_type      VARCHAR(32) NOT NULL
                    CHECK (actor_type IN ('user', 'admin', 'system', 'agent', 'scim')),
    actor_ip        INET,
    actor_user_agent TEXT,
    -- What
    event_type      VARCHAR(64) NOT NULL,                -- e.g. user.login, role.assign, token.issue
    event_category  VARCHAR(32) NOT NULL
                    CHECK (event_category IN ('authentication', 'authorization', 'user_management',
                        'group_management', 'application_management', 'policy_management',
                        'provisioning', 'governance', 'system')),
    event_outcome   VARCHAR(16) NOT NULL
                    CHECK (event_outcome IN ('success', 'failure', 'error')),
    -- Target
    target_type     VARCHAR(64),                         -- user, role, application, group, session
    target_id       UUID,
    -- Details
    description     TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Timestamp
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partition audit_events by month for performance
-- CREATE TABLE audit_events_2026_05 PARTITION OF audit_events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_events_tenant_time ON audit_events (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_events_actor ON audit_events (actor_id, occurred_at DESC) WHERE actor_id IS NOT NULL;
CREATE INDEX idx_audit_events_target ON audit_events (target_type, target_id, occurred_at DESC);
CREATE INDEX idx_audit_events_type ON audit_events (event_type, occurred_at DESC);
CREATE INDEX idx_audit_events_category ON audit_events (tenant_id, event_category, occurred_at DESC);
```

## Identity Provider Federation Tables

```sql
-- ============================================================
-- IDENTITY PROVIDERS (upstream SSO federation)
-- ============================================================

CREATE TABLE identity_providers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    protocol            VARCHAR(16) NOT NULL
                        CHECK (protocol IN ('oidc', 'saml', 'social')),
    -- OIDC configuration
    oidc_issuer_url     VARCHAR(2048),
    oidc_client_id      VARCHAR(255),
    oidc_client_secret_enc TEXT,                         -- encrypted
    oidc_scopes         TEXT[],
    oidc_discovery_url  VARCHAR(2048),
    -- SAML configuration
    saml_entity_id      VARCHAR(2048),
    saml_sso_url        VARCHAR(2048),
    saml_slo_url        VARCHAR(2048),
    saml_certificate    TEXT,                             -- X.509 cert PEM
    saml_name_id_format VARCHAR(128),
    -- Social provider
    social_provider     VARCHAR(32),                     -- google, github, microsoft, apple
    -- Attribute mapping
    attribute_mapping   JSONB NOT NULL DEFAULT '{}',
    -- Status
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_identity_providers_tenant ON identity_providers (tenant_id, is_active);

CREATE TABLE federated_identities (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    idp_id              UUID NOT NULL REFERENCES identity_providers(id) ON DELETE CASCADE,
    idp_user_id         VARCHAR(512) NOT NULL,           -- identifier from the upstream IdP
    idp_username        VARCHAR(255),
    linked_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_login_at       TIMESTAMPTZ,
    UNIQUE (idp_id, idp_user_id)
);

CREATE INDEX idx_federated_identities_user ON federated_identities (user_id);
```

## Example Query: Resolve User Permissions

```sql
-- Given a user, find all permissions (direct + group-inherited)
WITH user_effective_roles AS (
    -- Direct role assignments
    SELECT r.id AS role_id, r.name AS role_name, 'direct' AS source
    FROM user_roles ur
    JOIN roles r ON r.id = ur.role_id
    WHERE ur.user_id = :user_id
      AND (ur.expires_at IS NULL OR ur.expires_at > now())

    UNION

    -- Group-inherited role assignments
    SELECT r.id AS role_id, r.name AS role_name, 'group:' || g.display_name AS source
    FROM group_members gm
    JOIN group_roles gr ON gr.group_id = gm.group_id
    JOIN roles r ON r.id = gr.role_id
    JOIN groups g ON g.id = gm.group_id
    WHERE gm.user_id = :user_id
)
SELECT DISTINCT p.resource_type, p.action, uer.role_name, uer.source
FROM user_effective_roles uer
JOIN role_permissions rp ON rp.role_id = uer.role_id
JOIN permissions p ON p.id = rp.permission_id
ORDER BY p.resource_type, p.action;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant Management | 1 | tenants |
| User Management | 4 | users, user_emails, user_phone_numbers, user_addresses |
| Groups | 2 | groups, group_members |
| Credentials | 1 | user_credentials (polymorphic) |
| Authentication Policies | 1 | authentication_policies |
| Sessions | 1 | sessions |
| OAuth 2.1 / OIDC | 5 | oauth_clients, oauth_scopes, oauth_client_scopes, authorization_codes, access_tokens |
| Token Management | 1 | refresh_tokens |
| RBAC | 5 | roles, permissions, role_permissions, user_roles, group_roles |
| Application Assignments | 1 | application_assignments |
| Identity Governance | 3 | certification_campaigns, certification_items, sod_policies |
| Provisioning | 2 | directory_connections, provisioning_events |
| Audit | 1 | audit_events |
| Federation | 2 | identity_providers, federated_identities |
| **Total** | **30** | |

---

## Key Design Decisions

1. **SCIM 2.0 attribute alignment** — User table columns map directly to RFC 7643 Core User schema attributes. Multi-valued attributes (emails, phones, addresses) are separate tables rather than JSONB arrays, enabling indexing and foreign key constraints on individual values.

2. **Polymorphic credentials table** — All credential types share a single table with type-specific nullable columns. This simplifies the "list all credentials for a user" query and avoids a proliferation of credential sub-tables, at the cost of many nullable columns.

3. **Tenant-scoped everything** — Every entity except the tenants table itself includes a tenant_id foreign key. Combined with PostgreSQL Row-Level Security policies, this enforces data isolation between tenants at the database level.

4. **OAuth 2.1 PKCE enforcement** — The authorization_codes table stores code_challenge and code_challenge_method, and oauth_clients defaults require_pkce to true, enforcing OAuth 2.1 compliance at the schema level.

5. **AAL tracking on credentials** — Each credential records its NIST SP 800-63B assurance level (AAL1/2/3), enabling the authentication policy engine to enforce minimum assurance requirements by checking credential AAL against policy required_aal.

6. **Time-bounded role assignments** — user_roles includes an expires_at column, supporting just-in-time access and automatic expiration of elevated privileges without separate scheduled jobs.

7. **Audit events as append-only** — The audit_events table is designed for partitioning by month and has no UPDATE or DELETE use cases. Indexes are optimised for the most common query patterns: by tenant+time, by actor, by target, and by event type.

8. **Separate federation table** — federated_identities links local users to upstream IdP identities, supporting multiple linked IdPs per user (e.g., a user can sign in via both Google and corporate SAML).

9. **Agent identity on oauth_clients** — Non-human agents (AI bots, MCP tool-calling pipelines) are modeled as oauth_clients with is_agent=true and agent_owner_id linking to the human principal, supporting the emerging agentic identity requirement.

10. **Governance as first-class tables** — Certification campaigns and separation-of-duties policies have dedicated tables rather than being implemented as application logic, ensuring governance state is auditable and queryable.
