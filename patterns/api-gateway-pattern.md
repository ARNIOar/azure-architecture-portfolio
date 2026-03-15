# API Gateway Pattern

> **When to use:** Multiple backend services need a unified, secure, and managed entry point for external or internal consumers.

---

## Pattern Overview

An API Gateway sits between clients and backend services, providing cross-cutting concerns: authentication, rate limiting, request transformation, and routing.

```mermaid
graph LR
    C1[Web App] --> GW[API Gateway]
    C2[Mobile App] --> GW
    C3[Partner API] --> GW
    GW --> S1[Accounts Service]
    GW --> S2[Loans Service]
    GW --> S3[Payments Service]
```

## Azure Implementation — API Management (APIM)

| APIM Feature | Purpose |
|-------------|---------|
| **Products** | Group APIs with access policies (e.g., "Internal", "Partner") |
| **Policies** | Inbound/outbound/on-error XML pipeline for transforms, caching, rate-limit |
| **Named values** | Centralize configuration; reference Key Vault secrets |
| **Subscriptions** | API keys per consumer; track usage |
| **Backends** | Define backend URLs with circuit breaker and load balancing |
| **Versions & revisions** | Non-breaking changes (revisions) vs breaking changes (versions) |

## Banking Example — Core Banking API Facade

```mermaid
graph TB
    subgraph Clients
        BRANCH[Branch App]
        MOBILE[Mobile Banking]
        PARTNER[Partner Fintech]
    end

    subgraph "Azure API Management"
        PROD_INT[Product: Internal]
        PROD_EXT[Product: Partner]
        POL[Policies: JWT validation,<br/>rate-limit, IP filter]
    end

    subgraph Backend Services
        ACC[Accounts API<br/>App Service]
        LOAN[Loans API<br/>Functions]
        PAY[Payments API<br/>Container App]
    end

    BRANCH --> PROD_INT
    MOBILE --> PROD_INT
    PARTNER --> PROD_EXT
    PROD_INT --> POL --> ACC
    POL --> LOAN
    POL --> PAY
    PROD_EXT --> POL
```

**Why APIM here?**
- Branch and mobile apps share the same backend APIs with the same policies
- Partner access gets stricter rate limits, IP filtering, and different product grouping
- JWT validation in APIM offloads auth from every backend service
- Revision system allows updating APIs without downtime

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| APIM tier | Premium (or Developer for study) | VNet integration needed for private backends |
| Auth | OAuth2 + JWT validation policy | Entra ID tokens validated at the gateway |
| Backend protocol | HTTPS with managed identity | No credentials in APIM config; Key Vault for secrets |
| Versioning | URL path (`/v1/`, `/v2/`) | Simplest for partners; header-based adds complexity |

## Anti-Patterns to Avoid

- **Gateway as business logic** — Keep policies thin; don't put domain logic in XML transforms.
- **Single monolith behind the gateway** — The gateway pattern assumes multiple backend services.
- **No caching strategy** — Use APIM response caching for read-heavy, slowly-changing endpoints.
- **Exposing internal errors** — Use `on-error` policies to return safe error responses.
