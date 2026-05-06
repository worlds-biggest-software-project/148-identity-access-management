# Identity & Access Management (IAM) — Feature & Functionality Survey

> Candidate #148 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Okta | Commercial SaaS | Proprietary (~$6–15/user/month) | https://www.okta.com/ |
| Microsoft Entra ID | Commercial SaaS | Proprietary (included in M365 or ~$6/user/month) | https://www.microsoft.com/en-us/security/business/microsoft-entra |
| CyberArk Identity | Commercial SaaS | Proprietary (enterprise contract) | https://www.cyberark.com/products/identity-management/ |
| SailPoint | Commercial SaaS | Proprietary (enterprise contract) | https://www.sailpoint.com/ |
| Ping Identity | Commercial SaaS | Proprietary (subscription) | https://www.pingidentity.com/ |
| Auth0 | Commercial SaaS | Proprietary (free to 7,500 MAU; $23+/month production) | https://auth0.com/ |
| Keycloak | Open source | Apache License 2.0 | https://www.keycloak.org/ |
| WorkOS | Commercial SaaS | Proprietary (from $49/month) | https://workos.com/ |

## Feature Analysis by Solution

### Okta

**Core features**
- Single Sign-On (SSO) with 7,000+ pre-built app integrations
- Adaptive multi-factor authentication (MFA) with risk-based rules
- User lifecycle management (provisioning, deprovisioning)
- Identity governance and access reviews
- API access control and OAuth 2.0/OIDC support
- Universal directory for cloud and on-premises identity
- Customizable authentication flows

**Differentiating features**
- Largest app integration catalogue (7,000+) reducing custom integration burden
- Adaptive MFA with behaviour-based rules vs. static MFA
- Comprehensive app ecosystem reducing engineering effort
- Strong customer success and onboarding

**UX patterns**
- Admin dashboard for user and app management
- End-user portal with app access
- Customizable login flows without coding
- Progressive disclosure of advanced features

**Integration points**
- REST API for custom integrations
- SAML 2.0, OpenID Connect (OIDC), OAuth 2.0
- System for Cross-domain Identity Management (SCIM) for provisioning
- Webhooks for custom event handling
- App integration marketplace

**Known gaps**
- Privileged Access Management (PAM) requires separate purchase
- Premium pricing for mid-to-large deployments
- Some governance features require add-on SKUs
- Limited AI-driven analytics compared to SailPoint

**Licence / IP notes**
- Proprietary SaaS; no licensing conflicts; adaptive MFA algorithms are proprietary

---

### Microsoft Entra ID

**Core features**
- Cloud-native directory integrated with Microsoft 365 and Azure
- Single Sign-On (SSO) for cloud and on-premises applications
- Conditional Access policies for risk-based authentication
- Multi-factor authentication (MFA) with passwordless support
- User provisioning and deprovisioning (SCIM)
- Identity Protection for detecting account compromise
- Enterprise applications gallery

**Differentiating features**
- Deep integration with Microsoft 365 and Azure ecosystem (free for existing M365 customers)
- Conditional Access policies with signal-based rules (location, device, risk)
- Passwordless sign-in (Windows Hello for Business, FIDO2)
- Tightly integrated with Microsoft Defender

**UX patterns**
- Azure portal-native interface
- Conditional Access policy builder (visual policy designer)
- Risk-based authentication with progressive challenges
- Integration with Microsoft ecosystem tools

**Integration points**
- SAML 2.0, OIDC, OAuth 2.0 support
- SCIM 2.0 for provisioning
- Microsoft Graph API for custom integrations
- Hybrid identity (Azure AD Connect) for on-premises synchronization
- Power Platform and Microsoft app integrations

**Known gaps**
- Complex licensing model (M365 tier-dependent)
- Limited non-Microsoft app depth compared to Okta
- Identity governance less sophisticated than SailPoint
- PAM capabilities limited

**Licence / IP notes**
- Proprietary SaaS; included in M365 licensing; conditional access algorithms are proprietary

---

### CyberArk Identity

**Core features**
- Privileged Access Management (PAM) as foundation
- Cloud-native SSO and adaptive authentication
- Multi-factor authentication with passwordless support
- Identity governance and access certification
- Endpoint privilege management
- Secret management and rotation
- Compliance automation and audit logging

**Differentiating features**
- Strongest PAM capabilities in market
- Integrated secret and credential management
- Privileged account lifecycle management
- Strong endpoint privilege enforcement

**UX patterns**
- PAM-first interface with IAM modules layered on top
- Policy-based access rules with override workflows
- Audit and compliance dashboard
- Risk-based alert triage

**Integration points**
- REST API for custom integrations
- SAML 2.0, OIDC, OAuth 2.0
- SCIM 2.0 for provisioning
- Integration with PAM control plane (vault-based)
- Compliance reporting APIs

**Known gaps**
- High cost and complexity
- Less intuitive for non-PAM IAM workflows
- Steeper learning curve for pure SSO/identity governance use cases
- Long implementation cycles

**Licence / IP notes**
- Proprietary SaaS; PAM and credential management algorithms are proprietary IP

---

### SailPoint

**Core features**
- AI-driven identity governance with role mining
- Automated access certification and approval workflows
- Separation-of-duties policy enforcement
- Lifecycle management from hire to retire
- Predictive analytics for access risk
- Identity analytics dashboard
- Compliance automation and audit trails

**Differentiating features**
- Deepest identity governance capabilities in market
- AI-powered role recommendations and access analysis
- Advanced separation-of-duties enforcement
- Predictive analytics for access anomalies
- Strong compliance reporting

**UX patterns**
- Analytics-first interface with policy-driven workflows
- Governance dashboard showing role and access trends
- Certification campaigns with AI-guided recommendations
- Risk-scored accounts with override workflows

**Integration points**
- REST API for custom integrations
- SAML 2.0, OIDC, OAuth 2.0
- SCIM 2.0 for provisioning
- Integration with enterprise systems (SAP, Workday, Active Directory)
- Compliance reporting APIs

**Known gaps**
- Long deployment cycles (6–18 months typical)
- Not suitable for rapid SSO/MFA deployment
- Steep pricing for mid-market
- Pure identity governance focus may require separate SSO tool

**Licence / IP notes**
- Proprietary SaaS; AI-driven role mining and access recommendations are proprietary algorithms; no known patent encumbrances

---

### Ping Identity

**Core features**
- API-first SSO architecture with strong developer focus
- Adaptive authentication with behavioural analysis
- Multi-factor authentication with passwordless support
- Identity verification and proofing capabilities
- OAuth 2.0/OIDC native implementation
- Advanced access control and delegation
- Customizable authentication flows (no-code and code-based)

**Differentiating features**
- Developer-friendly API-first design
- Strong advanced authentication capabilities (identity proofing, biometrics)
- Flexible flow customization for custom applications
- Integration with modern application architectures

**UX patterns**
- Admin API-first interface (less visual admin dashboard)
- Flow builder for authentication customization
- Developer-oriented documentation and SDKs
- Flexible policy configuration

**Integration points**
- REST APIs with strong developer experience
- SAML 2.0, OIDC, OAuth 2.0
- SCIM 2.0 for provisioning
- WebAuthn/FIDO2 for passwordless
- Custom extension points for authentication logic

**Known gaps**
- Less intuitive admin UI compared to Okta
- Smaller app integration library vs. Okta
- Less mature governance features than SailPoint
- Smaller customer base may mean fewer reference architectures

**Licence / IP notes**
- Proprietary SaaS; authentication and flow algorithms are proprietary

---

### Auth0

**Core features**
- Developer-focused Customer Identity and Access Management (CIAM)
- Social login and enterprise SSO for B2C/B2B applications
- Customizable authentication flows (Actions pipeline)
- Multi-factor authentication with passwordless support
- User and application management APIs
- Rules engine for custom authentication logic
- Webhooks and event streaming

**Differentiating features**
- Fastest developer onboarding in the space
- Extensible Actions pipeline for custom logic
- Strong B2C/B2B application focus
- Pre-built social and enterprise identity provider integrations
- Generous free tier (7,500 MAU)

**UX patterns**
- Developer portal with quick-start templates
- Visual flow builder for authentication
- Rules/Actions pipeline for custom logic
- Dashboard for user and app management

**Integration points**
- REST API with excellent documentation
- SAML 2.0, OIDC, OAuth 2.0
- SCIM 2.0 for provisioning
- Webhooks for custom events
- Marketplace of extensions and integrations

**Known gaps**
- Governance features require integration with Okta core
- Less suitable for complex enterprise governance workflows
- Limited privilege access management
- Free tier limits for production deployments

**Licence / IP notes**
- Proprietary SaaS (Okta subsidiary); Actions pipeline logic is proprietary

---

### Keycloak

**Core features**
- Open-source IAM with SSO and federation
- Support for SAML 2.0, OpenID Connect, OAuth 2.0
- User directory and attribute mapping
- Role-based access control (RBAC)
- Multi-factor authentication
- User self-service portal
- Extensive customization and extensibility

**Differentiating features**
- Zero licensing cost enabling broad deployments
- Open-source allowing customization and auditing
- Full-featured platform with no commercial-only features
- Red Hat support option available

**UX patterns**
- Web admin console for configuration
- Customizable login and account themes
- Extensible architecture with custom providers
- Manual configuration vs. guided workflows

**Integration points**
- REST Admin API for custom integrations
- SAML 2.0, OIDC, OAuth 2.0
- SCIM 2.0 via extensions
- Webhook system extensions
- Custom provider development

**Known gaps**
- Operational complexity at scale (requires DevOps expertise)
- Slower UI compared to commercial alternatives
- Limited governance features (requires custom development)
- Smaller ecosystem of pre-built integrations
- No built-in PAM or secret management

**Licence / IP notes**
- Open source: Apache License 2.0; suitable for self-hosted deployments; community-driven development; review licensing if bundling with proprietary components

---

### WorkOS

**Core features**
- SSO (SAML 2.0, OIDC) API for SaaS products
- Directory Sync (SCIM 2.0) for user provisioning
- Audit Log for compliance (SOX, HIPAA)
- Directory APIs for synced user data
- Multi-tenant support for B2B SaaS
- Simple integration path for developers

**Differentiating features**
- Fastest enterprise SSO integration for SaaS builders
- Minimal implementation overhead (hours vs. weeks)
- Low price point for SMB SaaS founders
- Developer-friendly integration process

**UX patterns**
- Developer-first onboarding
- Simple configuration portal
- Ready-to-use UI components for login
- API-centric architecture

**Integration points**
- REST API for SSO and directory operations
- SAML 2.0 for enterprise SSO
- SCIM 2.0 for directory sync
- Audit Log API for compliance
- Webhooks for sync events

**Known gaps**
- Not a full IAM suite (no MFA, governance, PAM)
- Limited to SSO and directory sync
- Smaller feature set than competitors
- Newer vendor with smaller customer base

**Licence / IP notes**
- Proprietary SaaS; no licensing concerns; simple pricing model

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

- **Single Sign-On (SSO)** — All solutions offer SSO; protocol support (SAML 2.0, OIDC, OAuth 2.0) varies
- **Multi-factor authentication (MFA)** — All support MFA; passwordless (WebAuthn, passkeys) now baseline expectation
- **User provisioning and deprovisioning** — All support SCIM 2.0 or equivalent lifecycle automation
- **API access** — All expose REST APIs for custom integrations
- **Directory and user management** — All maintain user directories; cloud vs. on-premises approaches vary
- **Compliance reporting** — All support audit logging and compliance frameworks (PCI DSS, HIPAA, SOX)
- **Application integration** — All support integration with external applications
- **Role-based access control (RBAC)** — All support RBAC; attribute-based access control (ABAC) varies

### Differentiating Features

- **App integration breadth** — Okta (7,000+) dominates; others offer 100–1,000+ pre-built integrations
- **Risk-based adaptive authentication** — Okta and Microsoft Entra stand out with behaviour-based rules; others use static policies
- **Identity governance depth** — SailPoint leads with AI-driven role mining and certification; others have basic governance
- **Privileged access management** — CyberArk strongest; others lack or require add-ons
- **Developer experience** — Auth0 and Ping Identity excel with API-first design; others prioritize admin UX
- **CIAM vs. Workforce IAM** — Auth0 specializes in B2C; others focus on enterprise (B2B)
- **Passwordless and FIDO2** — All now support; depth and ease of deployment varies
- **Vendor ecosystem lock-in** — Microsoft Entra strongest for Microsoft shops; Okta most platform-agnostic

### Underserved Areas / Opportunities

- **Non-human identity management** — Emerging need for AI agent and RPA identity; no solution highlighted this
- **Decentralized identity (SSI)** — Verifiable credentials and self-sovereign identity largely unaddressed in commercial offerings
- **Continuous access reviews** — While governance exists (SailPoint), no solution emphasized continuous re-evaluation vs. periodic campaigns
- **Zero-trust machine identity** — With rise of microservices and agentic AI, machine-to-machine authentication is underserved
- **Contextual authorization at runtime** — All support static policies; runtime contextual decisions (e.g., "deny this request because it violates this user's access pattern") are limited
- **Predictive deprovisioning** — Research mentions ML-based early warning for access removal, but no vendor highlighted this
- **Natural-language access reviews** — No solution mentioned LLM-based summarization of access entitlements for easier certification
- **AI agent identity orchestration** — Emerging capability gap as agentic AI scales; none highlighted support for MCP or similar agent protocols

### AI-Augmentation Candidates

- **AI-driven role mining and access recommendations** — Analyse access patterns across thousands of users to identify least-privilege opportunities
- **LLM-based access review automation** — Summarize access entitlements in plain language, draft risk recommendations, process reviewer responses
- **Anomaly detection for account compromise** — Real-time ML-based detection of credential compromise using authentication telemetry
- **Predictive deprovisioning** — ML models flag users for access review before formal offboarding based on contextual changes
- **AI orchestration for agent identity** — Manage identity for non-human agents (AI bots, RPA, MCP pipelines) as scale increases
- **Continuous baseline learning** — Dynamically adjust risk thresholds based on user behaviour patterns vs. static rules

---

## Legal & IP Summary

All solutions are proprietary SaaS except Keycloak (Apache License 2.0). No copyright, licensing, or trademark conflicts identified. Adaptive MFA algorithms, role mining, and access recommendation engines are proprietary to vendors; no known active patents disclosed but techniques may overlap with existing ML-based identity patents. Keycloak licensing is clear (Apache 2.0); suitable for self-hosted deployments; no conflicts with proprietary extensions. Standards (SAML 2.0, OIDC, SCIM, FIDO2, WebAuthn) are publicly maintained (OASIS, IETF, W3C) with no licensing concerns. No materials omitted due to uncertainty. Recommend legal review if building AI-driven identity features using ML models, as techniques may overlap with existing patents in identity analytics and access prediction.

---

## Recommended Feature Scope

Based on the feature survey above, a competitive IAM platform should prioritise:

**Must-have (MVP)**
- Single Sign-On (SSO) supporting SAML 2.0, OpenID Connect, and OAuth 2.0
- User directory with attribute management
- Multi-factor authentication (MFA) with at least email and TOTP support
- User provisioning and deprovisioning (SCIM 2.0)
- RESTful API for application integration
- Admin dashboard for user and application management
- Audit logging for compliance (PCI DSS, HIPAA)
- Role-based access control (RBAC) with group management

**Should-have (v1.1)**
- Passwordless authentication (WebAuthn/FIDO2, email magic links)
- Risk-based adaptive authentication (conditional access based on location, device, behaviour)
- Pre-built application integrations (50+ common SaaS apps)
- Identity governance with access review workflows
- Multi-tenant support for B2B SaaS platforms
- Self-service password reset and profile management
- Advanced MFA options (biometrics, push notifications)
- Developer-friendly SDKs and quick-start guides

**Nice-to-have (backlog)**
- AI-driven role mining and access recommendations
- Privileged access management (PAM) integration
- Identity proofing and biometric verification
- Decentralized identity support (verifiable credentials)
- Continuous anomaly detection for account takeover
- AI agent and machine identity orchestration for MCP and agentic AI
- LLM-based access review automation
- Predictive deprovisioning based on contextual changes
- Social login integrations for B2C applications
