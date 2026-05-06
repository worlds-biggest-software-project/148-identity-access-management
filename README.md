# Identity & Access Management (IAM)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native identity and access management platform delivering SSO, MFA, RBAC, access reviews, and privileged access management without enterprise-tier pricing or vendor lock-in.

Identity & Access Management (IAM) is a candidate project to build an open-source IAM platform for engineering teams, SaaS builders, and security organisations. It addresses the core problem that enterprise identity is fragmented across expensive, siloed commercial suites, while open alternatives lag in governance, adaptive authentication, and emerging requirements such as agentic AI identity.

---

## Why Identity & Access Management (IAM)?

- Workforce IAM from Okta, Microsoft Entra ID, and Ping Identity ranges from USD 4–15 per user per month for SSO+MFA, with identity governance (SailPoint, CyberArk) priced at undisclosed enterprise contract rates that exclude mid-market buyers.
- Privileged access management is a separate purchase from most IAM suites; CyberArk leads but adds significant cost and complexity, and Okta, Auth0, and WorkOS lack PAM entirely.
- Keycloak is the only mature open-source option and carries operational complexity at scale, a slower UI, limited governance features, and no built-in PAM or secret management.
- AI agent identity, machine-to-machine authentication for MCP and agentic AI pipelines, and predictive deprovisioning are largely unaddressed across the incumbent landscape.
- Decentralised identity (verifiable credentials, SSI) and continuous access re-evaluation are emerging buyer requirements not met by current commercial offerings.

---

## Key Features

### Authentication and SSO

- Single Sign-On supporting SAML 2.0, OpenID Connect, and OAuth 2.0
- Multi-factor authentication including TOTP, email, and push notifications
- Passwordless authentication via WebAuthn, FIDO2, and email magic links
- Risk-based adaptive authentication using location, device, and behavioural signals
- Customisable authentication flows for B2B and B2C applications

### Directory, Provisioning, and Lifecycle

- User directory with attribute management and group structures
- Automated user provisioning and deprovisioning via SCIM 2.0
- Self-service password reset and profile management
- Multi-tenant support for B2B SaaS platforms
- Hybrid synchronisation for cloud and on-premises identity sources

### Access Control and Governance

- Role-based access control (RBAC) with group management
- Access review workflows and certification campaigns
- Separation-of-duties policy enforcement
- Audit logging aligned with PCI DSS, HIPAA, and SOX
- Compliance reporting and audit trails

### Developer and Integration Surface

- RESTful API for application integration
- Pre-built connectors for common SaaS applications
- Webhooks and event streaming for custom workflows
- Developer SDKs and quick-start guides
- Admin dashboard for user and application management

### Advanced and AI-Augmented Capabilities

- AI-driven role mining and least-privilege access recommendations
- LLM-based access review summarisation and recommendation drafting
- Continuous anomaly detection for credential compromise and account takeover
- Identity orchestration for non-human agents (AI bots, RPA, MCP pipelines)
- Predictive deprovisioning based on contextual changes

---

## AI-Native Advantage

AI-driven role mining can continuously analyse access patterns across thousands of users to identify overprivileged accounts and recommend least-privilege configurations without manual RBAC modelling. LLM-based access review automation summarises each user's entitlements in plain language, drafts risk-justified approval or revocation recommendations, and processes certifier responses at scale. Anomaly detection models trained on authentication telemetry detect credential compromise in real time without static rule thresholds. AI orchestration for non-human agents addresses an unaddressed capability gap as organisations deploy agentic AI and MCP tool-calling pipelines at scale.

---

## Tech Stack & Deployment

The platform is intended for self-hosted, cloud, and hybrid deployment. It aligns with established open standards: SAML 2.0 (OASIS), OAuth 2.1 and OpenID Connect, SCIM 2.0 (IETF) for provisioning, and FIDO2/WebAuthn (W3C, FIDO Alliance) for passwordless authentication. The architecture follows NIST SP 800-207 Zero Trust principles and supports authentication assurance levels defined in NIST SP 800-63B. Integration is API-first with REST endpoints, webhooks, and SDKs for application developers.

---

## Market Context

The global IAM market is estimated at USD 25–26 billion in 2026 and is projected to reach USD 65–78 billion by 2034 at a CAGR of approximately 12–15%, with North America accounting for roughly 40% of global spend. Workforce IAM pricing runs USD 4–15 per user per month; CIAM is typically MAU-based starting free and scaling to USD 0.05–0.20 per MAU at volume; PAM is priced per vault or privileged account. Primary buyers include IT directors and CISOs consolidating identity infrastructure, SaaS product teams adding enterprise SSO for B2B customers, compliance officers managing access certification, and DevOps teams managing machine and service account identities.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
