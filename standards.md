# Standards & API Reference

> Project: Identity & Access Management (IAM) · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Annex A 5.15–5.18: Access Management Controls**
- URL: https://www.iso.org/standard/27001
- Relevance: ISO 27001:2022 Annex A Controls 5.15 (Access Control), 5.16 (Identity Management), 5.17 (Authentication Information), and 5.18 (Access Rights) collectively mandate that organisations implement systematic identity management, role-based access control, multi-factor authentication, and periodic access reviews. An IAM platform implementing these controls is the primary evidence-generating mechanism for ISO 27001 certification audits. Access certification campaigns, provisioning logs, and MFA enforcement policies are all direct audit artefacts.

**ISO/IEC 29115 — Entity Authentication Assurance Framework**
- URL: https://www.iso.org/standard/45138.html
- Relevance: Defines a framework for entity authentication assurance (Levels 1–4) broadly aligned with NIST SP 800-63B's Authenticator Assurance Levels. Relevant for IAM platforms targeting international regulated markets where NIST SP 800-63B is not directly applicable but an equivalent assurance level framework is required (healthcare, financial services, government in non-US jurisdictions).

### W3C & IETF Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Relevance: The foundational authorization standard for securing API access. All modern IAM platforms implement OAuth 2.0 as the primary mechanism for issuing scoped access tokens to applications and services. OAuth 2.0 scopes define what resources an authenticated entity may access — critical for RBAC and ABAC enforcement at the API layer. IAM platforms must implement OAuth 2.0 authorization server capabilities.

**OAuth 2.1 — Draft Consolidation (IETF draft-ietf-oauth-v2-1)**
- URL: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- Relevance: OAuth 2.1 is the current IETF draft consolidation of OAuth 2.0 plus widely adopted security extensions (PKCE mandatory for all clients, deprecation of implicit flow and password grant). As of 2026, 81% of remote MCP server implementations use OAuth 2.1 for agentic authentication. IAM platforms supporting AI agent identity should implement OAuth 2.1 flows including PAR (Pushed Authorisation Requests) and RAR (Rich Authorisation Requests) for fine-grained agent permissions.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Relevance: Defines how OAuth 2.0 access tokens are transmitted in API requests. Required for all IAM platform integrations with downstream APIs and services that validate bearer tokens for access control decisions.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Relevance: OIDC adds an identity layer (ID token as JWT) on top of OAuth 2.0, enabling client applications to verify end-user identity and obtain profile information in a standardised way. All modern IAM platforms must implement OIDC for SSO to cloud-native applications. Certified OIDC providers are listed by the OpenID Foundation — certification is a procurement requirement in regulated sectors.

**OpenID Connect Discovery 1.0**
- URL: https://openid.net/specs/openid-connect-discovery-1_0.html
- Relevance: Defines the well-known configuration endpoint (/.well-known/openid-configuration) that exposes IdP metadata — issuer URL, authorization endpoint, JWKS URI, supported scopes, and claims. Required for automated integration with OIDC-compliant applications and for AI agent MCP authentication flows that use OAuth 2.1 discovery.

**RFC 7644 / RFC 7643 / RFC 7642 — SCIM 2.0 (System for Cross-domain Identity Management)**
- URL: https://datatracker.ietf.org/doc/html/rfc7644 ; https://datatracker.ietf.org/doc/html/rfc7643
- Relevance: IETF standard for automating user provisioning and deprovisioning across applications and identity providers. RFC 7643 defines the User and Group schema; RFC 7644 defines the REST protocol (HTTP endpoints, operations, filtering, pagination); RFC 7642 defines use cases. SCIM 2.0 is mandatory for enterprise IAM procurement — it enables zero-touch onboarding and offboarding. All platforms (Okta, Microsoft Entra, Keycloak, Auth0, WorkOS) implement SCIM 2.0.

**SAML 2.0 — Security Assertion Markup Language (OASIS Standard)**
- URL: https://www.oasis-open.org/standard/saml/
- Relevance: XML-based SSO federation standard dominant in enterprise deployments, ratified by OASIS in March 2005. SAML 2.0 is still required by the majority of enterprise procurement checklists — many legacy enterprise applications (Salesforce, Workday, legacy SaaS) support only SAML 2.0. IAM platforms must implement both SP-initiated and IdP-initiated SAML 2.0 SSO flows, attribute mapping, and metadata exchange. RFC 7522 provides the SAML 2.0 profile for OAuth 2.0 grant types.

**WebAuthn — Web Authentication API (W3C Recommendation Level 2)**
- URL: https://www.w3.org/TR/webauthn-2/
- Relevance: W3C standard enabling browser-based passwordless authentication via public key cryptography. WebAuthn is the client-side specification within the FIDO2 framework. Browsers generate a public-private key pair; the private key never leaves the device. WebAuthn Level 2 was published as a W3C Recommendation on 8 April 2021; Level 3 is in Working Draft. IAM platforms must implement WebAuthn for passkey and FIDO2 security key authentication — this is now a baseline expectation for phishing-resistant MFA.

**FIDO2 / CTAP2 — Client to Authenticator Protocol (FIDO Alliance)**
- URL: https://fidoalliance.org/specifications/
- Relevance: FIDO2 comprises WebAuthn (W3C) and CTAP2 (FIDO Alliance's Client to Authenticator Protocol), which defines how authenticators (security keys, biometric scanners) communicate with client platforms. FIDO2 is the foundation for passkeys — the synchronisable credential format that enables cross-device passwordless authentication. IAM platforms must support FIDO2/WebAuthn to meet modern phishing-resistant MFA requirements mandated by CISA and NIST SP 800-63B-4.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Relevance: Defines the JWT format used for ID tokens (OIDC), access tokens (OAuth 2.0), and SCIM operation signing. IAM platforms issue and validate JWTs extensively. Correct JWT validation (signature algorithm, issuer, audience, expiry) is a critical security control that must be implemented correctly to prevent token forgery attacks.

**RFC 7517 — JSON Web Key (JWK) / RFC 7518 — JSON Web Algorithms (JWA)**
- URL: https://datatracker.ietf.org/doc/html/rfc7517 ; https://datatracker.ietf.org/doc/html/rfc7518
- Relevance: Define the JSON format for cryptographic keys (JWKS) and the algorithm identifiers used in JWTs. IAM platforms expose a JWKS endpoint (/.well-known/jwks.json) for relying parties to retrieve public keys for JWT signature verification. Correct JWKS rotation is operationally critical for IAM reliability.

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- Relevance: The industry standard for REST API documentation. Okta, Auth0, WorkOS, and Keycloak all publish OpenAPI 3.x specifications for their management APIs. An IAM platform must publish a comprehensive OAS 3.1 specification for its User Management, Application Management, Policy, Audit Log, and SCIM APIs — enabling integration partner SDK generation and automated testing.

**W3C Verifiable Credentials Data Model 2.0**
- URL: https://www.w3.org/TR/vc-data-model-2.0/
- Relevance: W3C Recommendation (15 May 2025) defining the data model for cryptographically verifiable digital credentials. Relevant for IAM platforms supporting decentralised identity (SSI) and digital trust use cases — enabling IAM to issue, verify, and revoke verifiable credentials for attributes such as employment, qualifications, or age. An emerging but increasingly important capability as enterprise pilots of SSI accelerate in 2025–2026.

**W3C Decentralised Identifiers (DIDs) v1.1**
- URL: https://www.w3.org/TR/did-1.1/
- Relevance: W3C Candidate Recommendation defining globally unique identifiers that are cryptographically verifiable and not dependent on any centralised authority. DIDs underpin self-sovereign identity (SSI) and verifiable credential ecosystems. IAM platforms extending into decentralised identity must support DID resolution. DID v1.1 is not expected to advance to full Recommendation before April 2026.

**JSON Schema 2020-12**
- URL: https://json-schema.org/specification
- Relevance: Standard for validating JSON payloads. IAM API request bodies (user creation, group assignment, policy definition, SCIM provisioning events) must be validated against published JSON Schema definitions to prevent malformed data from corrupting user directories or granting incorrect access.

### Security & Compliance Standards

**NIST SP 800-63B-4 — Digital Identity Guidelines: Authentication and Authenticator Management**
- URL: https://csrc.nist.gov/pubs/sp/800/63/b/4/final
- Relevance: The authoritative US government specification for authentication assurance levels (AAL1, AAL2, AAL3) and authenticator requirements. AAL2 requires phishing-resistant MFA (FIDO2/WebAuthn or physical OTP device); AAL3 requires hardware-bound cryptographic authenticators. IAM platforms must implement AAL2+ to meet FedRAMP requirements and increasingly to satisfy enterprise security requirements. SP 800-63B-4 (final release July 2025) integrates syncable authenticator (passkey) guidance as normative text.

**NIST SP 800-207 — Zero Trust Architecture**
- URL: https://csrc.nist.gov/pubs/sp/800/207/final
- Relevance: Defines the Zero Trust Architecture specification with identity as the primary security perimeter. IAM is the Policy Engine and Policy Administrator in NIST 800-207's ZTA model — every access request is evaluated using identity attributes, device posture, and context before granting access. IAM platforms must support continuous authentication, per-session least-privilege access, and contextual access decisions to implement NIST 800-207-aligned ZTA. SP 800-207A extends this to cloud-native and multi-cloud environments.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr.eu/
- Relevance: IAM platforms process some of the most sensitive personal data in any organisation — identity, authentication history, access rights, and behavioural telemetry. GDPR Article 5 (data minimisation), Article 17 (right to erasure — deprovisioning must delete or anonymise all identity data), Article 25 (privacy by design), and Article 32 (technical security measures) all directly constrain IAM architecture. Data Processing Agreements are required with all IAM SaaS providers. SCIM deprovisioning must satisfy GDPR erasure requirements.

**OWASP Application Security Verification Standard (ASVS) — Level 2/3**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- Relevance: ASVS defines security requirements for web applications; Chapter 2 (Authentication) and Chapter 3 (Session Management) are directly applicable to IAM platform implementation. IAM platforms must meet ASVS Level 2 (commercial) or Level 3 (high-value targets) for authentication flows, token handling, session lifecycle, and credential storage. ASVS compliance is increasingly required by enterprise procurement security assessments.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Relevance: IAM APIs are the most sensitive APIs in an organisation's stack — exposing them enables account takeover, privilege escalation, and full directory compromise. Critical risks: Broken Object Level Authorization (a user must not be able to access another user's profile or tokens); Broken Function Level Authorization (admin user management endpoints must be restricted to privileged roles); Unrestricted Access to Sensitive Business Flows (token issuance endpoints are high-value brute-force targets requiring rate limiting); and Broken Authentication (all IAM API authentication must implement PKCE, short-lived tokens, and token rotation).

### MCP Server Specifications

**Model Context Protocol (MCP) — Agentic Identity and OAuth 2.1**
- URL: https://modelcontextprotocol.io/ ; https://aembit.io/blog/mcp-oauth-2-1-pkce-and-the-future-of-ai-authorization/
- Relevance: MCP is the de facto standard for AI agent tool connectivity (97M+ monthly SDK downloads, 10,000+ public MCP servers as of 2026). MCP's authorization framework is built on OAuth 2.1 — 81% of remote MCP servers authenticate with OAuth 2.1 as of early 2026. IAM platforms must support machine-to-machine (M2M) OAuth 2.1 flows specifically designed for AI agents: client credentials with minimal scopes, dynamic client registration for ephemeral agents, and token introspection for per-request authorisation decisions. Microsoft has published an "Entra Agent ID" API (Microsoft Graph beta) for managing AI agent identities. IAM platforms that support agentic identity — issuing, managing, and revoking credentials for non-human agents — will gain significant competitive differentiation as AI adoption at scale creates a new class of identity problem.

---

## Similar Products — Developer Documentation & APIs

### Okta (Workforce Identity and CIAM)

- **Description:** Market-leading cloud IAM platform with 7,000+ pre-built app integrations, adaptive MFA, lifecycle management, and identity governance. Serves both workforce and CIAM use cases. OAuth 2.0 authorization server and certified OIDC provider.
- **API Documentation:** https://developer.okta.com/docs/api
- **OpenID Connect & OAuth 2.0:** https://developer.okta.com/docs/api/openapi/okta-oauth/guides/overview
- **SCIM Protocol Reference:** https://developer.okta.com/docs/reference/scim/scim-20/
- **SDKs/Libraries:** JavaScript, Python, Java, .NET, Go, PHP — https://developer.okta.com/code/
- **Developer Guide:** https://developer.okta.com/docs/concepts/oauth-openid/
- **Standards:** REST/JSON; OpenAPI 3.0 specification published; SAML 2.0, OIDC 1.0, OAuth 2.0, SCIM 2.0, FIDO2/WebAuthn, JWT/JWK
- **Authentication:** API Token; OAuth 2.0 scoped access tokens for admin APIs

### Microsoft Entra ID (Microsoft Graph API)

- **Description:** Cloud-native identity platform integrated with Microsoft 365 and Azure. Provides conditional access, SSO, MFA, lifecycle management, and identity governance. Programmatic access via Microsoft Graph REST API.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/api/resources/identity-network-access-overview
- **Microsoft Graph Overview:** https://learn.microsoft.com/en-us/graph/overview
- **Native Authentication API:** https://learn.microsoft.com/en-us/entra/identity-platform/reference-native-authentication-api
- **Entra Agent ID (AI agent identity, preview):** https://learn.microsoft.com/en-us/graph/api/resources/agentid-platform-overview
- **SDKs/Libraries:** MSAL for .NET, JavaScript, Python, Java, Android, iOS — https://learn.microsoft.com/en-us/entra/identity-platform/msal-overview
- **Standards:** REST/JSON; OpenAPI specification published; SAML 2.0, OIDC 1.0, OAuth 2.0, SCIM 2.0, FIDO2/WebAuthn; OData query parameters
- **Authentication:** OAuth 2.0 (Microsoft Entra ID / Azure AD app registration)

### Auth0 Management API

- **Description:** Developer-focused CIAM platform (Okta subsidiary) with B2C/B2B flows, social login, extensible Actions pipeline, and a generous free tier (7,500 MAU). Management API v2 enables programmatic control of users, applications, connections, and rules.
- **API Documentation:** https://auth0.com/docs/api/management/v2
- **Developer Docs:** https://auth0.com/docs
- **SDKs/Libraries:** JavaScript, Python, Java, .NET, Ruby, PHP, Go — https://auth0.com/docs/libraries
- **Developer Guide:** https://auth0.com/docs/get-started
- **Standards:** REST/JSON; OpenAPI 3.0 specification published; SAML 2.0, OIDC 1.0, OAuth 2.0, SCIM 2.0, FIDO2/WebAuthn, Social login providers
- **Authentication:** Management API access tokens (OAuth 2.0 client credentials); API keys for certain operations

### Keycloak Admin REST API

- **Description:** Open-source IAM (Apache License 2.0) from the JBoss/Red Hat ecosystem. Full-featured SSO, OIDC, SAML 2.0, RBAC, and federation with a REST Admin API for programmatic management. Hosted by Red Hat as Red Hat Build of Keycloak (RHBK).
- **API Documentation:** https://www.keycloak.org/docs-api/latest/rest-api/index.html
- **Documentation:** https://www.keycloak.org/documentation
- **Server Administration Guide:** https://www.keycloak.org/docs/latest/server_admin/index.html
- **GitHub:** https://github.com/keycloak/keycloak
- **SDKs/Libraries:** keycloak-js (JavaScript); community SDKs for Python, Java, Go, PHP available on GitHub
- **Standards:** REST/JSON; OpenAPI specification published; SAML 2.0, OIDC 1.0, OAuth 2.0, SCIM 2.0 (via extension), FIDO2/WebAuthn, JWT/JWK
- **Authentication:** Bearer token (admin REST API token obtained via token endpoint); OAuth 2.0 client credentials
- **Licence:** Apache License 2.0

### WorkOS API

- **Description:** Developer-first API for adding enterprise SSO (SAML 2.0, OIDC), Directory Sync (SCIM 2.0), Audit Log, and Admin Portal to SaaS applications. Targeted at SaaS founders and product teams adding enterprise identity features quickly. From $49/month.
- **API Documentation:** https://workos.com/docs/reference
- **SSO Documentation:** https://workos.com/docs/sso
- **Developer Docs:** https://workos.com/docs
- **SDKs/Libraries:** Node.js, Python, Ruby, PHP, Go, Java — https://workos.com/docs/sdks
- **Standards:** REST/JSON; OpenAPI specification published; SAML 2.0, OIDC 1.0, SCIM 2.0, Audit Log API
- **Authentication:** API key (WorkOS secret key)

### Ping Identity Developer Platform

- **Description:** API-first enterprise IAM platform covering adaptive authentication, identity verification, biometrics, OIDC, and advanced OAuth 2.0 flows. Developer portal provides APIs, SDKs, and comprehensive documentation. Strong in financial services and advanced authentication scenarios.
- **API Documentation:** https://apidocs.pingidentity.com/
- **Developer Portal:** https://developer.pingidentity.com/
- **PingOne OIDC Sample:** https://github.com/pingidentity/pingone-sample-js
- **OIDC Client SDK (npm):** https://github.com/pingidentity-developers-experience/ping-oidc-client-sdk
- **Standards:** REST/JSON; OpenAPI specification published; SAML 2.0, OIDC 1.0, OAuth 2.0 (including FAPI 2.0 for financial-grade), SCIM 2.0, FIDO2/WebAuthn
- **Authentication:** OAuth 2.0 bearer tokens; API key for management operations

### Microsoft Entra Verified ID (Verifiable Credentials)

- **Description:** Microsoft's W3C Verifiable Credentials and DID-based decentralised identity service within Microsoft Entra. Enables issuance and verification of verifiable credentials (employment, qualifications, compliance attestations) using the W3C VC Data Model 2.0. An early enterprise implementation of SSI standards.
- **API Documentation:** https://learn.microsoft.com/en-us/entra/verified-id/verifiable-credentials-configure-issuer
- **Developer Guide:** https://learn.microsoft.com/en-us/entra/verified-id/
- **Standards:** W3C Verifiable Credentials Data Model 2.0, W3C DIDs v1.0/1.1, Microsoft DID:ION (on Bitcoin/IPFS), OpenID Connect for Verifiable Presentations (OID4VP)
- **Authentication:** Azure AD OAuth 2.0

---

## Notes

**Agentic Identity as the Next Frontier:** The emergence of AI agents operating via MCP is creating a new identity management challenge that no current IAM platform addresses comprehensively. AI agents need: ephemeral credentials with minimal scopes, just-in-time provisioning, full audit trails of all actions taken on behalf of a human principal, and revocation capabilities triggered by anomaly detection. Microsoft has published the Entra Agent ID API (beta) as an early signal of platform direction. An IAM platform that natively supports agentic identity — with dedicated lifecycle management for non-human agents — would address a significant gap in the current market.

**OAuth 2.1 PKCE Requirement:** OAuth 2.1 mandates PKCE (Proof Key for Code Exchange, RFC 7636) for all OAuth 2.0 clients including server-side applications — not just SPAs and mobile apps. IAM platforms that haven't yet enforced PKCE for all flows need to update their authorization server implementations to be OAuth 2.1 compliant. This is particularly relevant for MCP server authentication where AI agents use OAuth 2.1.

**Passkeys / Syncable Authenticators:** NIST SP 800-63B-4 (finalised July 2025) formally recognises syncable authenticators (passkeys) as normative AAL2-eligible authenticators, resolving prior ambiguity. IAM platforms can now implement passkeys as a compliant AAL2 method — enabling phishing-resistant passwordless authentication that syncs across devices via platform keychain (Apple iCloud Keychain, Google Password Manager, Windows Hello).

**Keycloak SCIM Availability:** Keycloak's native SCIM 2.0 support was added in Keycloak 26.x (released 2025). Earlier versions required third-party extensions. IAM platforms building on or integrating with Keycloak should confirm the target version's native SCIM support.
