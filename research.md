# Identity & Access Management (IAM)

> Candidate #148 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| Okta | Widely adopted cloud IAM with SSO, adaptive MFA, lifecycle management, and 7,000+ pre-built app integrations | Commercial SaaS | ~$6–15/user/month (workforce tier) | Strength: largest integration catalogue, reliable SSO; Weakness: premium pricing, some PAM capabilities require add-ons |
| Microsoft Entra ID | Cloud-native directory and IAM tightly integrated with Microsoft 365 and Azure; conditional access, SSO, MFA | Commercial SaaS | Included in M365 E3/E5 or standalone ~$6/user/month | Strength: free for existing Microsoft shops, deep ecosystem; Weakness: complex licensing, limited non-Microsoft app depth |
| CyberArk Identity | PAM-first platform extended into full IAM suite: SSO, adaptive MFA, lifecycle management, identity governance | Commercial SaaS | Enterprise contract (undisclosed) | Strength: strongest privileged access management; Weakness: high cost and complexity |
| SailPoint | AI-driven identity governance: role mining, access certification, separation-of-duties enforcement, and lifecycle | Commercial SaaS | Enterprise contract (undisclosed) | Strength: IGA depth and AI-powered analytics; Weakness: long deployment cycles |
| Ping Identity | API-first SSO, adaptive authentication, MFA, and identity verification with strong developer experience | Commercial SaaS | Subscription (undisclosed) | Strength: API/developer-friendly, good for custom apps; Weakness: less intuitive admin UX |
| Auth0 (Okta) | Developer-focused CIAM platform with B2C/B2B flows, social login, and extensible Actions pipeline | Commercial SaaS | Free to 7,500 MAU; $23+/month for production | Strength: fastest developer onboarding; Weakness: governance features require Okta core |
| Keycloak | Open-source IAM with SSO, OIDC, SAML, RBAC, and federation; hosted by Red Hat as RHBK | Open source | Free; Red Hat support subscription | Strength: full-featured, no licence cost; Weakness: operational complexity, slower UI |
| WorkOS | SSO/SCIM API for SaaS products adding enterprise identity quickly (SAML, SCIM, Audit Log, Directory Sync) | Commercial SaaS | From $49/month | Strength: fastest enterprise SSO integration for SaaS builders; Weakness: not a full IAM suite |

## Relevant Industry Standards or Protocols

- **SAML 2.0 (Security Assertion Markup Language)** — XML-based SSO federation standard dominant in enterprise; OASIS standard; required by virtually all enterprise buyer procurement checklists
- **OAuth 2.1 / OpenID Connect (OIDC)** — Token-based authorisation and authentication for APIs, mobile apps, and browser applications; OAuth 2.1 is the current consolidation of best practices; emerging use for MCP agentic auth flows
- **SCIM 2.0 (System for Cross-domain Identity Management)** — IETF standard for automated user provisioning and deprovisioning across directories and applications
- **FIDO2 / WebAuthn / Passkeys** — W3C and FIDO Alliance standards for phishing-resistant passwordless authentication; baseline expectation for modern MFA implementations
- **NIST SP 800-63B** — Digital identity guidelines covering authentication assurance levels (AAL1–AAL3) used to specify MFA requirements in US government and regulated sectors
- **ISO/IEC 27001:2022 Annex A 5.15–5.18** — Controls for access management, identity management, authentication, and privileged access
- **NIST SP 800-207** — Zero Trust Architecture specification; defines identity as the primary security perimeter and informs IAM architecture design

## Available Research Materials

1. Kosenkov et al. (2023). *A Systematic Review of IAM Requirements in Enterprises and Potential Contributions of Self-Sovereign Identity*. Business & Information Systems Engineering / Springer. https://link.springer.com/article/10.1007/s12599-023-00830-x — peer-reviewed
2. Liu et al. (2024). *Analysis of Multi-Factor Authentication (MFA) Schemes in Zero Trust Architecture: Current State, Challenges, and Future Trends*. International Journal of Computer Applications. https://www.ijcaonline.org/archives/volume186/number57/liu-2024-ijca-924310.pdf — peer-reviewed
3. Alshaikh et al. (2024). *Identity and Access Management in Zero Trust Frameworks*. ResearchGate. https://www.researchgate.net/publication/388106052_Identity_and_Access_Management_in_Zero_Trust_Frameworks — not peer-reviewed (preprint)
4. Various authors (2025). *A Systematic Literature Review on the Implementation and Challenges of Zero Trust Architecture Across Domains*. PMC / NIH. https://pmc.ncbi.nlm.nih.gov/articles/PMC12526847/ — peer-reviewed
5. Soni et al. (2026). *A Comprehensive Review and Comparative Analysis of Zero Trust Architecture: Evolution, Implementation Strategies, and Key Challenges*. SAGE Journals. https://journals.sagepub.com/doi/10.1177/0926227X251409922 — peer-reviewed
6. Microsoft (2026). *Microsoft's Perspective on Agentic Identity Standards*. Microsoft Entra Blog. https://techcommunity.microsoft.com/blog/microsoft-entra-blog/microsoft%E2%80%99s-perspective-on-agentic-identity-standards/2111910 — vendor position paper
7. Cloud Security Alliance (2025). *Navigating IAM Standards and Protocols*. CSA Artifact. https://cloudsecurityalliance.org/artifacts/iam-standards-and-protocols — industry body guidance

## Market Research

**Market Size:** The global IAM market is estimated at USD 25–26 billion in 2026, projected to reach USD 65–78 billion by 2034, at a CAGR of approximately 12–15%. North America accounts for roughly 40% of global spend.

**Funding:** Okta is publicly traded (NASDAQ: OKTA) with a market cap around USD 16–18 billion in 2026. SailPoint was taken private by Thoma Bravo and relisted; CyberArk is publicly traded. Auth0 was acquired by Okta for USD 6.5 billion. Newer entrants such as WorkOS and Logto have raised seed and Series A rounds targeting developer-led IAM.

**Pricing Landscape:** Workforce IAM ranges from USD 4–15 per user per month for SSO+MFA; identity governance (SailPoint, Saviynt) runs significantly higher at enterprise contract rates. CIAM pricing is typically MAU-based, starting free and scaling to USD 0.05–0.20 per MAU at volume. Privileged access management (CyberArk) is priced per vault or per privileged account.

**Key Buyer Personas:** IT directors and CISOs at enterprises consolidating identity infrastructure; SaaS product teams adding enterprise SSO for B2B customers; compliance officers managing access certification for SOX, HIPAA, or FedRAMP; DevOps teams managing machine and service account identities.

**Notable Trends:** Passwordless adoption (passkeys/FIDO2) is crossing the mainstream threshold; 72% of enterprises now use multi-protocol SSO. AI agent identity is an emerging requirement — platforms supporting OAuth 2.1 for MCP and machine-to-machine flows are gaining differentiation. Zero Trust Architecture adoption is accelerating, making IAM the primary security control plane rather than the network perimeter. Decentralised identity (SSI/verifiable credentials) is moving from research to early enterprise pilots.

## AI-Native Opportunity

- AI-driven role mining and access recommendation engines can continuously analyse access patterns across thousands of users to identify overprivileged accounts and suggest least-privilege configurations without manual RBAC modelling
- LLM-based access review automation can summarise each user's access entitlements in plain language, draft risk-justified approval or revocation recommendations, and process certifier responses at scale
- Anomaly detection models trained on authentication telemetry (time, location, device, behaviour) can detect credential compromise and account takeover in real time without static rule thresholds
- AI orchestration can manage identity for non-human agents (AI bots, RPA processes, MCP tool-calling pipelines) as organisations deploy agentic AI at scale — a largely unaddressed capability gap in current platforms
- Predictive deprovisioning: ML models can identify users whose role, project, or organisational context has changed and proactively flag access for review before a formal offboarding event, reducing the orphaned-account risk window
