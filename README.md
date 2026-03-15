# Architecture Portfolio

Cloud architecture designs for banking and enterprise systems — focusing on event-driven patterns, network isolation, identity governance, and cost-aware design on Azure.

---

## Architecture Case Studies

Each case study documents a realistic system design with business context, architecture diagrams, technology justifications, security considerations, and trade-off analysis.

| Design | Domain | Key Azure Services | Pattern |
|--------|--------|--------------------|---------|
| [Loan Application Processing](architecture/loan-application-processing.md) | Lending | APIM, Functions, Service Bus, Cosmos DB | Event-driven pipeline |
| [Document Ingestion System](architecture/document-ingestion-system.md) | Compliance | Blob Storage, Cognitive Services, Functions | Blob-triggered processing |
| [Event-Driven Payment Processing](architecture/event-driven-payment-processing.md) | Payments | Event Hubs, Functions, Cosmos DB | CQRS + event sourcing |
| [Secure Data Pipeline](architecture/secure-data-pipeline.md) | Data | Data Factory, SQL, Private Endpoints | ETL with network isolation |
| [Week 01 Resource Organization](architecture/week-01-resource-organization.md) | Governance | Management Groups, Azure Policy, RBAC, Cost Management | Hierarchical governance baseline |

**Each case study covers:**
- Business problem and requirements
- Constraints (regulatory, performance, budget)
- Architecture overview with Mermaid diagrams
- Azure services used and why each was chosen
- Security considerations (identity, network, encryption)
- Trade-offs and alternatives considered
- Cost estimates

---

## Architecture Patterns

Reusable pattern guides with decision frameworks, service comparisons, and banking-specific examples.

| Pattern | When to Use |
|---------|-------------|
| [Event-Driven Architecture](patterns/event-driven-architecture.md) | Decoupled, async workflows — loan pipelines, payment processing |
| [API Gateway Pattern](patterns/api-gateway-pattern.md) | Unified entry point for multiple backend services |
| [Messaging Decision Guide](patterns/messaging-decision-guide.md) | Choosing between Event Grid, Event Hubs, Service Bus, and Storage Queues |
| [Hub-Spoke Networking](patterns/hub-spoke-networking.md) | Multi-environment network topology with shared services |
| [Identity & Auth Decisions](patterns/identity-auth-decisions.md) | Managed Identity, service principals, RBAC, Conditional Access |

---

## Diagrams

- [Reference Diagrams](diagrams/reference-diagrams.md) — Mermaid diagrams for common Azure architecture patterns (hub-spoke, identity flow, decision trees)

---

## Templates

Templates for creating new architecture documents with a consistent structure:

- [Case Study Template](templates/case-study-template.md) — Full structure for documenting an architecture design: business problem, requirements, constraints, architecture overview, services, security, trade-offs
- [ADR Template](templates/adr-template.md) — Architecture Decision Record for documenting individual technology choices with options considered and rationale
