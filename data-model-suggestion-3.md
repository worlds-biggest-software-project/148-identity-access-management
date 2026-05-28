# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Identity & Access Management (IAM) · Created: 2026-05-19

## Philosophy

This model uses a relational backbone for core identity entities (users, roles, groups, applications, tokens) with JSONB columns for variable, extensible, and jurisdiction-specific data. The relational layer enforces referential integrity and supports efficient joins for the most common queries. The JSONB layer absorbs variability — custom user attributes, jurisdiction-specific fields, configurable policy conditions, and extensible metadata — without requiring schema migrations.

This is the pragmatic middle ground between the strict normalized model (Suggestion 1) and the fully event-sourced model (Suggestion 2). It acknowledges that IAM platforms must support wildly different attribute sets across tenants (one tenant needs employee_number, another needs national_id, a third needs department_hierarchy), and that forcing all possible attributes into fixed columns leads to sparsely populated tables with hundreds of nullable columns.

Auth0, Okta, and Keycloak all use this pattern in practice: core fields are relational columns, while user_metadata and app_metadata are stored as JSON blobs. Microsoft Entra ID uses "extension attributes" — effectively a JSONB-equivalent pattern. This approach is particularly well-suited for a platform that must support multi-region deployments where each jurisdiction has different identity requirements (e.g., EU requires GDPR-specific fields, US government requires PIV certificate references, healthcare adds FHIR practitioner IDs).

**Best for:** Platforms serving diverse tenants with varying attribute requirements, multi-jurisdiction deployments, rapid feature iteration where schema changes must be non-breaking, and teams that want relational integrity without sacrificing extensibility.

**Trade-offs:**
- Pro: Core queries use standard relational joins — no performance penalty for common operations
- Pro: Extensible without migrations — new tenant-specific attributes are JSONB fields
- Pro: Multi-jurisdiction support — jurisdiction-specific fields live in JSONB without polluting the core schema
- Pro: Faster MVP development — start with essential columns, extend via JSONB as requirements emerge
- Pro: Lower table count than fully normalized model
- Con: JSONB fields lack foreign key constraints — referential integrity is application-enforced
- Con: Complex JSONB queries can be slower than indexed relational columns
- Con: Schema validation for JSONB must be done at the application layer (JSON Schema)
- Con: JSONB contents are less discoverable — developers must consult documentation
- Con: Migration from JSONB to relational columns requires data backfill when patterns stabilize

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643/7644) | Core SCIM User attributes (userName, name, emails) are relational columns. Enterprise extension and custom schema extensions stored in JSONB `scim_extensions` column. Multi-valued attributes (emails, phones) in JSONB arrays. |
| OAuth 2.1 / OIDC | Core OAuth fields are relational. Client metadata (custom claims, token customization) in JSONB. OIDC claims mapping stored as JSONB. |
| FIDO2 / WebAuthn | WebAuthn credential fields stored relationally for indexed lookups. Attestation details and extensions in JSONB. |
| NIST SP 800-63B | AAL levels are relational enums. Policy conditions (adaptive auth rules) use JSONB for flexible rule definitions. |
| ISO/IEC 27001:2022 | Audit events use relational fields for indexing, JSONB for variable event metadata. |
| W3C Verifiable Credentials 2.0 | VC documents stored as JSONB (they are inherently JSON-LD). |
| GDPR | Data subject request tracking uses relational fields. Jurisdiction-specific consent records in JSONB. |

---

## Core Identity Tables

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
    -- Flexible tenant configuration
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"},
    --   "default_locale": "en-US",
    --   "password_policy": {"min_length": 12, "require_mfa": true},
    --   "jurisdiction": "EU",
    --   "data_residency": "eu-west-1",
    --   "features": {"sso": true, "governance": true, "agent_identity": false}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants (slug);

-- ============================================================
-- USERS (relational core + JSONB extensions)
-- ============================================================

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    -- Core SCIM attributes (relational, indexed)
    username            VARCHAR(255) NOT NULL,
    email               VARCHAR(320),
    email_verified      BOOLEAN NOT NULL DEFAULT false,
    display_name        VARCHAR(255),
    given_name          VARCHAR(128),
    family_name         VARCHAR(128),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    identity_state      VARCHAR(32) NOT NULL DEFAULT 'active'
                        CHECK (identity_state IN ('staged', 'active', 'suspended', 'deprovisioned')),
    -- Multi-valued SCIM attributes as JSONB arrays
    emails              JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"value": "jane@work.com", "type": "work", "primary": true},
    --           {"value": "jane@home.com", "type": "home", "primary": false}]
    phone_numbers       JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"value": "+1-555-0100", "type": "mobile", "primary": true}]
    addresses           JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"streetAddress": "123 Main St", "locality": "Springfield",
    --            "region": "IL", "postalCode": "62701", "country": "US",
    --            "type": "work", "primary": true}]

    -- SCIM Enterprise Extension (JSONB for variable fields)
    enterprise_profile  JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "employeeNumber": "E12345",
    --   "costCenter": "CC-4400",
    --   "organization": "Acme Corp",
    --   "division": "Engineering",
    --   "department": "Platform",
    --   "managerId": "uuid-of-manager"
    -- }

    -- Custom / tenant-specific attributes
    custom_attributes   JSONB NOT NULL DEFAULT '{}',
    -- Example for EU tenant:
    -- {"national_id_type": "passport", "gdpr_consent_date": "2026-01-15",
    --  "data_processing_basis": "legitimate_interest"}
    -- Example for healthcare tenant:
    -- {"fhir_practitioner_id": "Practitioner/12345", "npi_number": "1234567890",
    --  "medical_licence_state": "CA"}
    -- Example for government tenant:
    -- {"piv_card_serial": "ABC123", "clearance_level": "secret",
    --  "agency_code": "DOD"}

    -- Lifecycle timestamps
    last_login_at       TIMESTAMPTZ,
    provisioned_at      TIMESTAMPTZ,
    deprovisioned_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, username)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);
CREATE INDEX idx_users_email ON users (tenant_id, email);
CREATE INDEX idx_users_state ON users (tenant_id, identity_state);
-- GIN index for JSONB queries on multi-valued attributes and custom fields
CREATE INDEX idx_users_emails_gin ON users USING GIN (emails jsonb_path_ops);
CREATE INDEX idx_users_custom_gin ON users USING GIN (custom_attributes jsonb_path_ops);
CREATE INDEX idx_users_enterprise_gin ON users USING GIN (enterprise_profile jsonb_path_ops);

-- ============================================================
-- GROUPS
-- ============================================================

CREATE TABLE groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    display_name    VARCHAR(255) NOT NULL,
    description     TEXT,
    group_type      VARCHAR(32) NOT NULL DEFAULT 'standard'
                    CHECK (group_type IN ('standard', 'dynamic', 'system')),
    -- Dynamic group rules (JSONB for flexible rule definitions)
    dynamic_rules   JSONB,
    -- Example: {"match": "all", "conditions": [
    --   {"field": "enterprise_profile.department", "op": "eq", "value": "Engineering"},
    --   {"field": "identity_state", "op": "eq", "value": "active"}
    -- ]}
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, display_name)
);

CREATE INDEX idx_groups_tenant ON groups (tenant_id);

CREATE TABLE group_members (
    group_id    UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    added_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (group_id, user_id)
);

CREATE INDEX idx_group_members_user ON group_members (user_id);
```

## Authentication & Credentials

```sql
-- ============================================================
-- CREDENTIALS
-- ============================================================

CREATE TABLE credentials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    credential_type     VARCHAR(32) NOT NULL
                        CHECK (credential_type IN (
                            'password', 'totp', 'webauthn', 'recovery_code',
                            'email_otp', 'sms_otp', 'push', 'certificate'
                        )),
    -- Core fields (relational for indexed lookups)
    display_name        VARCHAR(255),
    aal_level           SMALLINT NOT NULL DEFAULT 1 CHECK (aal_level IN (1, 2, 3)),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_used_at        TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    -- Type-specific data in JSONB (avoids nullable columns)
    credential_data     JSONB NOT NULL,
    -- Password example:
    -- {"hash": "$argon2id$...", "algorithm": "argon2id", "changed_at": "2026-01-15T..."}
    --
    -- TOTP example:
    -- {"secret_encrypted": "base64...", "algorithm": "SHA256", "digits": 6, "period": 30}
    --
    -- WebAuthn/FIDO2 example:
    -- {"credential_id": "base64url...", "public_key": "base64...", "sign_count": 42,
    --  "aaguid": "00000000-0000-0000-0000-000000000000", "transports": ["internal", "usb"],
    --  "attestation_type": "none", "user_verified": true,
    --  "backup_eligible": true, "backup_state": true}
    --
    -- Recovery code example:
    -- {"codes": [{"hash": "sha256...", "used": false}, ...]}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_credentials_user ON credentials (user_id, credential_type);
CREATE INDEX idx_credentials_webauthn ON credentials
    USING GIN (credential_data jsonb_path_ops)
    WHERE credential_type = 'webauthn';

-- ============================================================
-- AUTHENTICATION POLICIES
-- ============================================================

CREATE TABLE authentication_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Policy rules as JSONB (flexible conditional access)
    conditions      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "required_aal": 2,
    --   "require_mfa": true,
    --   "allowed_credential_types": ["password", "webauthn"],
    --   "mfa_types": ["totp", "webauthn"],
    --   "ip_ranges": ["10.0.0.0/8", "192.168.0.0/16"],
    --   "countries": ["US", "CA", "GB"],
    --   "device_types": ["managed"],
    --   "risk_level_max": "medium",
    --   "time_window": {"days": ["mon","tue","wed","thu","fri"], "hours": {"start": 6, "end": 22}}
    -- }
    actions         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "session_max_age_seconds": 28800,
    --   "require_reauthentication": false,
    --   "step_up_aal": 3,
    --   "notify_admin": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_auth_policies_tenant ON authentication_policies (tenant_id, is_active, priority);

-- ============================================================
-- SESSIONS
-- ============================================================

CREATE TABLE sessions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_token_hash  VARCHAR(128) NOT NULL UNIQUE,
    aal_achieved        SMALLINT NOT NULL DEFAULT 1,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    -- Session context as JSONB
    context             JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {"ip": "203.0.113.50", "user_agent": "Mozilla/5.0...",
    --  "device_fingerprint": "abc123", "country": "US",
    --  "city": "San Francisco", "os": "macOS", "browser": "Chrome"}
    authenticated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_activity_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at          TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_user ON sessions (user_id, is_active);
CREATE INDEX idx_sessions_token ON sessions (session_token_hash);
CREATE INDEX idx_sessions_expires ON sessions (expires_at) WHERE is_active = true;
```

## OAuth 2.1 / OpenID Connect

```sql
-- ============================================================
-- OAUTH CLIENTS (Applications)
-- ============================================================

CREATE TABLE oauth_clients (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    client_id           VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash  TEXT,
    client_name         VARCHAR(255) NOT NULL,
    client_type         VARCHAR(16) NOT NULL
                        CHECK (client_type IN ('confidential', 'public', 'machine')),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    -- OAuth configuration as JSONB (many optional fields)
    oauth_config        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "redirect_uris": ["https://app.example.com/callback"],
    --   "post_logout_redirect_uris": ["https://app.example.com/logged-out"],
    --   "grant_types": ["authorization_code", "refresh_token"],
    --   "response_types": ["code"],
    --   "require_pkce": true,
    --   "access_token_ttl": 3600,
    --   "refresh_token_ttl": 86400,
    --   "id_token_ttl": 3600,
    --   "token_endpoint_auth_method": "client_secret_post"
    -- }
    -- SAML configuration (if applicable)
    saml_config         JSONB,
    -- Example:
    -- {
    --   "entity_id": "https://sp.example.com/saml",
    --   "acs_url": "https://sp.example.com/saml/acs",
    --   "slo_url": "https://sp.example.com/saml/slo",
    --   "name_id_format": "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress",
    --   "signing_certificate": "-----BEGIN CERTIFICATE-----\n..."
    -- }
    -- Agent identity fields
    agent_config        JSONB,
    -- Example:
    -- {
    --   "is_agent": true,
    --   "owner_id": "uuid-of-human-principal",
    --   "agent_type": "mcp_tool",
    --   "max_scopes": ["read:users", "write:documents"],
    --   "auto_revoke_on_anomaly": true
    -- }
    metadata            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oauth_clients_tenant ON oauth_clients (tenant_id);
CREATE INDEX idx_oauth_clients_client_id ON oauth_clients (client_id);

-- ============================================================
-- SCOPES
-- ============================================================

CREATE TABLE oauth_scopes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,
    display_name    VARCHAR(255),
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE oauth_client_scopes (
    client_id   UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    scope_id    UUID NOT NULL REFERENCES oauth_scopes(id) ON DELETE CASCADE,
    PRIMARY KEY (client_id, scope_id)
);

-- ============================================================
-- TOKENS (authorization codes, access tokens, refresh tokens)
-- ============================================================

CREATE TABLE authorization_codes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_hash       VARCHAR(128) NOT NULL UNIQUE,
    client_id       UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    scope           TEXT NOT NULL,
    -- PKCE and OIDC fields in JSONB
    code_params     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"redirect_uri": "https://...", "code_challenge": "...",
    --           "code_challenge_method": "S256", "nonce": "...", "state": "..."}
    expires_at      TIMESTAMPTZ NOT NULL,
    used_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_auth_codes_hash ON authorization_codes (code_hash);

CREATE TABLE access_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_hash      VARCHAR(128) NOT NULL UNIQUE,
    client_id       UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES sessions(id) ON DELETE SET NULL,
    scope           TEXT NOT NULL,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_access_tokens_hash ON access_tokens (token_hash);
CREATE INDEX idx_access_tokens_expires ON access_tokens (expires_at) WHERE revoked_at IS NULL;

CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token_hash      VARCHAR(128) NOT NULL UNIQUE,
    access_token_id UUID NOT NULL REFERENCES access_tokens(id) ON DELETE CASCADE,
    client_id       UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    scope           TEXT NOT NULL,
    rotation_count  INTEGER NOT NULL DEFAULT 0,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens (token_hash);
```

## RBAC / Access Control

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
    -- Permissions embedded as JSONB (avoids junction table for simple cases)
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"resource": "user", "actions": ["read", "write", "delete"]},
    --   {"resource": "application", "actions": ["read"]},
    --   {"resource": "role", "actions": ["read", "assign"]}
    -- ]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_roles_tenant ON roles (tenant_id);
CREATE INDEX idx_roles_permissions_gin ON roles USING GIN (permissions jsonb_path_ops);

-- User-role assignments (relational for integrity)
CREATE TABLE user_roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id     UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_by  UUID REFERENCES users(id) ON DELETE SET NULL,
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at  TIMESTAMPTZ,
    metadata    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"justification": "Project Alpha requires admin access",
    --           "approved_by": "uuid", "ticket": "JIRA-1234"}
    UNIQUE (user_id, role_id)
);

CREATE INDEX idx_user_roles_user ON user_roles (user_id);
CREATE INDEX idx_user_roles_role ON user_roles (role_id);

CREATE TABLE group_roles (
    group_id    UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
    role_id     UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (group_id, role_id)
);

-- ============================================================
-- APPLICATION ASSIGNMENTS
-- ============================================================

CREATE TABLE application_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    oauth_client_id UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    assignee_type   VARCHAR(16) NOT NULL CHECK (assignee_type IN ('user', 'group')),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    group_id        UUID REFERENCES groups(id) ON DELETE CASCADE,
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    metadata        JSONB NOT NULL DEFAULT '{}',
    CHECK (
        (assignee_type = 'user' AND user_id IS NOT NULL AND group_id IS NULL) OR
        (assignee_type = 'group' AND group_id IS NOT NULL AND user_id IS NULL)
    )
);

CREATE INDEX idx_app_assignments_client ON application_assignments (oauth_client_id);
CREATE INDEX idx_app_assignments_user ON application_assignments (user_id) WHERE user_id IS NOT NULL;
```

## Identity Governance

```sql
-- ============================================================
-- ACCESS CERTIFICATION
-- ============================================================

CREATE TABLE certification_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    campaign_type   VARCHAR(32) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'active', 'in_review', 'completed', 'cancelled')),
    owner_id        UUID NOT NULL REFERENCES users(id),
    -- Flexible scope and schedule
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "scope": {"applications": ["uuid1"], "roles": ["uuid2"], "groups": []},
    --   "schedule": {"starts_at": "2026-06-01", "due_at": "2026-06-15"},
    --   "reminders": {"frequency": "3d", "escalate_after": "7d"},
    --   "auto_revoke_on_expiry": true
    -- }
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cert_campaigns_tenant ON certification_campaigns (tenant_id, status);

CREATE TABLE certification_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES certification_campaigns(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    reviewer_id     UUID REFERENCES users(id),
    -- What is being reviewed
    access_details  JSONB NOT NULL,
    -- Example:
    -- {"type": "role", "role_id": "uuid", "role_name": "Admin",
    --  "granted_at": "2025-06-01", "granted_by": "uuid",
    --  "last_used": "2026-04-15", "usage_count_30d": 12}
    decision        VARCHAR(16) CHECK (decision IN ('approve', 'revoke', 'delegate', 'pending')),
    decision_reason TEXT,
    risk_score      SMALLINT,
    ai_recommendation JSONB,
    -- Example: {"action": "revoke", "confidence": 0.87,
    --           "reason": "User has not used this role in 90 days",
    --           "similar_users_revoked_pct": 72}
    decided_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cert_items_campaign ON certification_items (campaign_id);
CREATE INDEX idx_cert_items_user ON certification_items (user_id);
```

## Provisioning, Federation & Audit

```sql
-- ============================================================
-- DIRECTORY CONNECTIONS
-- ============================================================

CREATE TABLE directory_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    provider_type   VARCHAR(32) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- All provider-specific config in JSONB
    connection_config JSONB NOT NULL,
    -- SCIM example:
    -- {"endpoint_url": "https://...", "bearer_token_hash": "sha256..."}
    -- LDAP example:
    -- {"url": "ldaps://...", "bind_dn": "cn=admin,...", "base_dn": "dc=corp,...",
    --  "search_filter": "(objectClass=person)", "attribute_mapping": {...}}
    sync_status     JSONB NOT NULL DEFAULT '{"status": "idle"}',
    -- Example: {"status": "syncing", "last_sync_at": "2026-05-19T...",
    --           "users_synced": 1234, "errors": 2}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dir_connections_tenant ON directory_connections (tenant_id);

-- ============================================================
-- IDENTITY PROVIDERS (Federation)
-- ============================================================

CREATE TABLE identity_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    protocol        VARCHAR(16) NOT NULL
                    CHECK (protocol IN ('oidc', 'saml', 'social')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Protocol-specific config in JSONB
    provider_config JSONB NOT NULL,
    -- OIDC example:
    -- {"issuer_url": "https://accounts.google.com",
    --  "client_id": "...", "client_secret_encrypted": "...",
    --  "scopes": ["openid", "email", "profile"],
    --  "discovery_url": "https://.../.well-known/openid-configuration"}
    -- SAML example:
    -- {"entity_id": "https://idp.corp.com/saml",
    --  "sso_url": "https://idp.corp.com/saml/sso",
    --  "certificate": "-----BEGIN CERTIFICATE-----\n...",
    --  "name_id_format": "emailAddress"}
    attribute_mapping JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_idps_tenant ON identity_providers (tenant_id, is_active);

CREATE TABLE federated_identities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    idp_id          UUID NOT NULL REFERENCES identity_providers(id) ON DELETE CASCADE,
    idp_user_id     VARCHAR(512) NOT NULL,
    idp_profile     JSONB NOT NULL DEFAULT '{}',         -- cached IdP profile data
    linked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_login_at   TIMESTAMPTZ,
    UNIQUE (idp_id, idp_user_id)
);

CREATE INDEX idx_fed_identities_user ON federated_identities (user_id);

-- ============================================================
-- AUDIT EVENTS
-- ============================================================

CREATE TABLE audit_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    -- Core indexed fields (relational)
    actor_id        UUID,
    actor_type      VARCHAR(32) NOT NULL,
    event_type      VARCHAR(64) NOT NULL,
    event_category  VARCHAR(32) NOT NULL,
    event_outcome   VARCHAR(16) NOT NULL,
    target_type     VARCHAR(64),
    target_id       UUID,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Variable event details (JSONB)
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example for login event:
    -- {"ip": "203.0.113.50", "user_agent": "Mozilla/5.0...",
    --  "credential_types": ["password", "totp"], "aal_achieved": 2,
    --  "session_id": "uuid", "country": "US", "risk_score": 15}
    -- Example for role assignment:
    -- {"role_id": "uuid", "role_name": "Admin", "granted_by": "uuid",
    --  "justification": "Approved by manager", "expires_at": "2026-12-01"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_time ON audit_events (tenant_id, occurred_at DESC);
CREATE INDEX idx_audit_actor ON audit_events (actor_id, occurred_at DESC) WHERE actor_id IS NOT NULL;
CREATE INDEX idx_audit_target ON audit_events (target_type, target_id, occurred_at DESC);
CREATE INDEX idx_audit_type ON audit_events (event_type, occurred_at DESC);
CREATE INDEX idx_audit_details_gin ON audit_events USING GIN (details jsonb_path_ops);
```

## Example: JSONB Containment Query

```sql
-- Find all users in the Engineering department (enterprise extension)
SELECT id, username, email, display_name
FROM users
WHERE tenant_id = :tenant_id
  AND enterprise_profile @> '{"department": "Engineering"}'
  AND identity_state = 'active';

-- Find all users with a specific custom attribute (healthcare tenant)
SELECT id, username, email, custom_attributes->>'npi_number' AS npi
FROM users
WHERE tenant_id = :tenant_id
  AND custom_attributes ? 'npi_number';

-- Find all WebAuthn credentials with backup eligibility
SELECT c.id, c.user_id, u.username,
       c.credential_data->>'credential_id' AS webauthn_id,
       c.credential_data->>'backup_eligible' AS syncable
FROM credentials c
JOIN users u ON u.id = c.user_id
WHERE c.credential_type = 'webauthn'
  AND c.credential_data @> '{"backup_eligible": true}';

-- Find roles that grant write access to users
SELECT id, name, display_name
FROM roles
WHERE tenant_id = :tenant_id
  AND permissions @> '[{"resource": "user", "actions": ["write"]}]';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant Management | 1 | tenants |
| User Management | 1 | users (multi-valued attrs in JSONB) |
| Groups | 2 | groups, group_members |
| Credentials | 1 | credentials (type-specific data in JSONB) |
| Authentication Policies | 1 | authentication_policies (rules in JSONB) |
| Sessions | 1 | sessions (context in JSONB) |
| OAuth 2.1 / OIDC | 4 | oauth_clients, oauth_scopes, oauth_client_scopes, authorization_codes |
| Tokens | 2 | access_tokens, refresh_tokens |
| RBAC | 3 | roles (permissions in JSONB), user_roles, group_roles |
| Application Assignments | 1 | application_assignments |
| Identity Governance | 2 | certification_campaigns, certification_items |
| Directory / Provisioning | 1 | directory_connections |
| Federation | 2 | identity_providers, federated_identities |
| Audit | 1 | audit_events (details in JSONB) |
| **Total** | **23** | |

---

## Key Design Decisions

1. **SCIM multi-valued attributes in JSONB arrays** — Rather than separate user_emails, user_phones, user_addresses tables (as in Suggestion 1), these are stored as JSONB arrays on the users table. This reduces table count and simplifies SCIM API serialization, at the cost of not being able to foreign-key individual email addresses.

2. **Credential data as JSONB** — All credential-type-specific fields (password hash, TOTP secret, WebAuthn public key) live in a single JSONB column. This eliminates the many nullable columns in Suggestion 1's polymorphic approach and makes it trivial to add new credential types without schema changes.

3. **Permissions embedded in roles** — Rather than a separate permissions table and role_permissions junction table, permissions are a JSONB array on the roles table. GIN indexing enables containment queries ("which roles grant write access to users?"). This is simpler for the common case but means permission names are not enforced by foreign keys.

4. **Authentication policy conditions as JSONB** — Adaptive authentication rules (IP ranges, countries, device types, risk levels, time windows) are JSONB rather than separate tables. This enables rapid iteration on policy conditions without schema migrations and supports arbitrarily complex condition trees.

5. **Provider-specific config in JSONB** — Directory connections and identity providers store their type-specific configuration (SCIM endpoints, LDAP bind DN, OIDC discovery URL, SAML certificates) in JSONB rather than type-specific columns. This supports adding new provider types without schema changes.

6. **Custom attributes for multi-jurisdiction** — The custom_attributes JSONB column on users absorbs jurisdiction-specific, tenant-specific, and industry-specific fields. EU tenants store GDPR consent, healthcare tenants store NPI numbers, government tenants store clearance levels — all without altering the schema.

7. **AI recommendation in governance** — certification_items includes an ai_recommendation JSONB column for ML-generated review suggestions, including confidence scores and explanations. This integrates AI-driven governance without adding a separate recommendation table.

8. **GIN indexes on JSONB** — All frequently queried JSONB columns have GIN indexes using jsonb_path_ops, enabling efficient containment queries. This is the key performance mitigation for JSONB-based querying.

9. **Audit details as JSONB** — Core audit fields (actor, target, event type, timestamp) are relational for indexed filtering. Variable event-specific details (IP, device, risk score, affected resources) are JSONB, avoiding a one-size-fits-all column set.

10. **Dynamic group rules** — Groups support a dynamic_rules JSONB column that defines membership criteria (e.g., "all active users in Engineering department"). This enables automatic group membership without manual assignment, similar to Azure AD dynamic groups.
